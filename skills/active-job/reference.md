# Active Job Reference — Patterns, Examples, Edge Cases

Detailed reference for Active Job with Solid Queue in Rails 8+. Companion to SKILL.md.

---

## Table of Contents

- [Job Lifecycle](#job-lifecycle)
- [Argument Serialization Deep Dive](#argument-serialization-deep-dive)
- [Error Handling Patterns](#error-handling-patterns)
- [Queue Configuration Patterns](#queue-configuration-patterns)
- [Testing Patterns](#testing-patterns)
- [Solid Queue Specifics](#solid-queue-specifics)
- [ActionMailer Integration](#actionmailer-integration)
- [Concurrency Controls](#concurrency-controls)
- [Job Continuations](#job-continuations)
- [Recurring Tasks](#recurring-tasks)
- [Performance Patterns](#performance-patterns)
- [Anti-Patterns](#anti-patterns)
- [Migration from Other Backends](#migration-from-other-backends)

---

## Job Lifecycle

### Full Lifecycle Flow

```
1. Job.perform_later(args)
   ├── Serialize arguments (GlobalID for AR objects)
   ├── before_enqueue callbacks
   ├── around_enqueue callbacks (yield)
   ├── Enqueue to backend (Solid Queue inserts DB row)
   ├── after_enqueue callbacks
   └── Returns job instance (or false on failure)

2. Worker picks up job
   ├── Deserialize arguments (GlobalID lookup)
   ├── before_perform callbacks
   ├── around_perform callbacks (yield)
   ├── perform(deserialized_args)
   ├── after_perform callbacks
   └── Job marked complete

3. On failure
   ├── retry_on match? → Re-enqueue with wait
   ├── discard_on match? → after_discard callback → Drop
   ├── rescue_from match? → Execute handler
   └── Unhandled → Job marked failed
```

### Job States in Solid Queue

| State | DB Table | Description |
|-------|----------|-------------|
| Scheduled | `solid_queue_scheduled_executions` | Waiting for scheduled time |
| Ready | `solid_queue_ready_executions` | Available for worker pickup |
| Claimed | `solid_queue_claimed_executions` | Being processed by a worker |
| Completed | (deleted) | Successfully finished |
| Failed | `solid_queue_failed_executions` | Failed, not retried |
| Blocked | `solid_queue_blocked_executions` | Waiting for concurrency slot |

---

## Argument Serialization Deep Dive

### How GlobalID Works

```ruby
# When you enqueue:
UserCleanupJob.perform_later(user)

# Active Job serializes user to:
# "gid://myapp/User/42"

# When the worker picks it up, it deserializes:
# User.find(42)
```

**Any model with `include GlobalID::Identification`** works. All ActiveRecord models include this by default.

### Supported Argument Types (Complete List)

```ruby
# All of these are safe to pass to perform_later:
SomeJob.perform_later(
  "string",                          # String
  42,                                # Integer
  3.14,                              # Float
  BigDecimal("19.99"),               # BigDecimal
  true,                              # TrueClass / FalseClass
  nil,                               # NilClass
  :my_symbol,                        # Symbol
  Date.today,                        # Date
  Time.current,                      # Time
  DateTime.now,                      # DateTime
  Time.zone.now,                     # ActiveSupport::TimeWithZone
  1.day,                             # ActiveSupport::Duration
  { key: "value" },                  # Hash (string/symbol keys)
  [1, 2, 3],                         # Array
  1..10,                             # Range
  String,                            # Class
  Enumerable,                        # Module
  ActiveSupport::HashWithIndifferentAccess.new(a: 1)
)
```

### Nested Structures

```ruby
# ✅ Safe — nested hashes and arrays with supported types
SomeJob.perform_later(
  users: [user1, user2],           # Array of AR objects
  config: { retries: 3, wait: 5 }, # Hash of primitives
  tags: ["urgent", "billing"]       # Array of strings
)

# ❌ UNSAFE — Hash with non-string/symbol keys
SomeJob.perform_later({ 1 => "one" })  # Integer key — will fail

# ❌ UNSAFE — Nested unsupported types
SomeJob.perform_later(data: OpenStruct.new(a: 1))
```

### Custom Serializer Pattern

```ruby
# app/serializers/date_range_serializer.rb
class DateRangeSerializer < ActiveJob::Serializers::ObjectSerializer
  def serialize(range)
    super(
      "start" => range.first.iso8601,
      "end" => range.last.iso8601
    )
  end

  def deserialize(hash)
    Date.parse(hash["start"])..Date.parse(hash["end"])
  end

  def klass
    Range
  end
end

# config/initializers/active_job_serializers.rb
Rails.application.config.active_job.custom_serializers << DateRangeSerializer

# Autoload once (don't reload in dev — serializers must be stable)
# config/application.rb
config.autoload_once_paths << "#{root}/app/serializers"
```

### DeserializationError Handling

```ruby
class NotificationJob < ApplicationJob
  # Record deleted between enqueue and perform
  discard_on ActiveJob::DeserializationError

  # Or handle gracefully:
  rescue_from ActiveJob::DeserializationError do |exception|
    Rails.logger.warn("Record gone: #{exception.message}")
    # Don't re-raise — job is done
  end

  def perform(user)
    user.send_notification
  end
end
```

---

## Error Handling Patterns

### retry_on — Full Options

```ruby
class ApiSyncJob < ApplicationJob
  # Basic — 3s wait, 5 attempts
  retry_on CustomError

  # With polynomial backoff: 3s, 18s, 83s, 258s, ...
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 10

  # Fixed wait
  retry_on RateLimitError, wait: 60.seconds, attempts: 3

  # Custom wait proc
  retry_on ApiError, wait: ->(executions) { executions * 30 }, attempts: 5

  # Move to a different queue on retry
  retry_on TransientError, queue: :retries, priority: 100

  # With jitter (default 15%, set 0 to disable)
  retry_on TimeoutError, wait: 30, jitter: 0.25  # ±25% randomness

  # Block executed after all attempts exhausted
  retry_on ExternalServiceError, attempts: 3 do |job, exception|
    AdminMailer.job_failed(job, exception).deliver_later
  end
end
```

### discard_on — When to Use

```ruby
class ProcessWebhookJob < ApplicationJob
  # Record no longer exists — nothing to process
  discard_on ActiveJob::DeserializationError

  # Bad data — retrying won't fix it
  discard_on JSON::ParserError
  discard_on ArgumentError

  # Specific business logic error
  discard_on AccountSuspendedError

  # With callback
  discard_on InvalidPayloadError do |job, error|
    Rails.logger.error("Bad webhook payload: #{error.message}")
    WebhookLog.create!(job_id: job.job_id, error: error.message, status: :discarded)
  end
end
```

### Combined Pattern — The Safety Net

```ruby
class RobustJob < ApplicationJob
  # 1. Retry transient errors
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 5
  retry_on ActiveRecord::Deadlocked, wait: 5.seconds, attempts: 3
  retry_on Redis::ConnectionError, wait: 10.seconds, attempts: 3

  # 2. Discard permanent errors
  discard_on ActiveJob::DeserializationError
  discard_on ArgumentError

  # 3. Catch-all for unexpected errors (report + re-raise for default handling)
  rescue_from Exception do |exception|
    Rails.error.report(exception)
    raise exception
  end

  def perform(record)
    # The actual work
  end
end
```

### Idempotency Guards

```ruby
class ChargeOrderJob < ApplicationJob
  def perform(order)
    # Guard: don't charge twice
    return if order.charged?

    PaymentGateway.charge(order)
    order.update!(charged_at: Time.current)
  end
end

class SendWelcomeEmailJob < ApplicationJob
  def perform(user)
    # Guard: don't send twice
    return if user.welcome_email_sent?

    UserMailer.welcome(user).deliver_now
    user.update!(welcome_email_sent_at: Time.current)
  end
end
```

---

## Queue Configuration Patterns

### Multi-Worker Setup

```yaml
# config/queue.yml
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 500

  workers:
    # High-priority worker: fast jobs only
    - queues: [critical, mailers]
      threads: 5
      processes: 2
      polling_interval: 0.1

    # Default worker: general purpose
    - queues: [default]
      threads: 3
      processes: 2
      polling_interval: 0.5

    # Background worker: slow/heavy jobs
    - queues: [low_priority, imports, exports]
      threads: 2
      processes: 1
      polling_interval: 2
```

### Queue Naming Conventions

```ruby
# Good — descriptive, hierarchical
queue_as :critical        # Payment processing, auth
queue_as :default         # Standard business logic
queue_as :mailers         # Email delivery
queue_as :notifications   # Push notifications, in-app
queue_as :imports         # Data imports (potentially slow)
queue_as :exports         # Report generation
queue_as :low_priority    # Cleanup, analytics, non-urgent

# Bad — too generic or duplicative
queue_as :queue1
queue_as :jobs
queue_as :background      # Everything is background
```

### Queue Name Prefixing

```ruby
# config/application.rb
config.active_job.queue_name_prefix = Rails.env
# Result: production_default, staging_default, etc.

# Override per-job if needed
class SharedJob < ApplicationJob
  self.queue_name_prefix = nil  # No prefix
  queue_as :shared
end
```

### Priority Within Queues

```ruby
# Priority is a positive integer. Lower = more important.
class CriticalAlertJob < ApplicationJob
  queue_as :notifications
  queue_with_priority 0    # Process first
end

class WeeklyDigestJob < ApplicationJob
  queue_as :notifications
  queue_with_priority 50   # Process after critical
end

# Dynamic priority
class ProcessOrderJob < ApplicationJob
  queue_with_priority do
    order = arguments.first
    order.express? ? 0 : 10
  end
end
```

---

## Testing Patterns

### Test Helper Configuration

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  # Jobs are NOT performed by default in tests.
  # Use perform_enqueued_jobs when you need them to run.
end
```

### Pattern 1: Test That a Job Is Enqueued

```ruby
class OrderServiceTest < ActiveSupport::TestCase
  test "placing order enqueues payment job" do
    order = orders(:pending)

    assert_enqueued_with(job: ProcessPaymentJob, args: [order]) do
      OrderService.place(order)
    end
  end

  test "placing order enqueues to critical queue" do
    order = orders(:pending)

    assert_enqueued_with(job: ProcessPaymentJob, queue: "critical") do
      OrderService.place(order)
    end
  end

  test "bulk import enqueues jobs for each row" do
    assert_enqueued_jobs 3, only: ImportRowJob do
      ImportService.process(rows: 3)
    end
  end

  test "no job enqueued for invalid order" do
    assert_no_enqueued_jobs only: ProcessPaymentJob do
      OrderService.place(orders(:invalid))
    end
  end
end
```

### Pattern 2: Test Job Behavior

```ruby
class ProcessPaymentJobTest < ActiveSupport::TestCase
  test "charges the order and marks paid" do
    order = orders(:unpaid)

    perform_enqueued_jobs do
      ProcessPaymentJob.perform_later(order)
    end

    order.reload
    assert order.paid?
    assert_not_nil order.paid_at
  end

  test "is idempotent — safe to run twice" do
    order = orders(:unpaid)

    2.times { ProcessPaymentJob.perform_now(order.reload) }

    assert_equal 1, PaymentLog.where(order: order).count
  end
end
```

### Pattern 3: Test Error Handling

```ruby
class ExternalApiJobTest < ActiveSupport::TestCase
  test "retries on timeout" do
    # Mock the API to raise timeout
    ExternalApi.stub(:call, -> { raise Net::OpenTimeout }) do
      assert_enqueued_jobs 1, only: ExternalApiJob do
        ExternalApiJob.perform_now(records(:example))
      end
    end
  end

  test "discards on deserialization error" do
    job = ExternalApiJob.new("gid://app/User/999999")

    assert_nothing_raised do
      job.perform_now
    end
  end
end
```

### Pattern 4: Test Scheduled Jobs

```ruby
class ReminderJobTest < ActiveSupport::TestCase
  test "enqueues for tomorrow" do
    freeze_time do
      assert_enqueued_with(
        job: ReminderJob,
        at: 1.day.from_now
      ) do
        ReminderService.schedule(users(:active))
      end
    end
  end
end
```

### Pattern 5: Test Mailer Jobs

```ruby
class OrderMailerTest < ActiveSupport::TestCase
  test "confirmation email is enqueued" do
    order = orders(:completed)

    assert_enqueued_emails 1 do
      OrderMailer.confirmation(order).deliver_later
    end
  end

  test "confirmation email content" do
    order = orders(:completed)

    email = OrderMailer.confirmation(order)
    assert_equal ["customer@example.com"], email.to
    assert_match "Order ##{order.number}", email.subject
  end
end
```

### Pattern 6: Test with perform_enqueued_jobs Filtering

```ruby
class ComplexWorkflowTest < ActiveSupport::TestCase
  test "only processes payment jobs, not notification jobs" do
    order = orders(:pending)

    # Only perform payment jobs; notification jobs stay enqueued
    perform_enqueued_jobs(only: ProcessPaymentJob) do
      OrderWorkflow.execute(order)
    end

    order.reload
    assert order.paid?

    # Notification job was enqueued but not yet performed
    assert_enqueued_with(job: NotifyCustomerJob)
  end
end
```

---

## Solid Queue Specifics

### Database Setup

```yaml
# config/database.yml — separate database for queue (default Rails 8 setup)
development:
  primary:
    <<: *default
    database: storage/development.sqlite3
  queue:
    <<: *default
    database: storage/development_queue.sqlite3
    migrations_paths: db/queue_migrate

production:
  primary:
    <<: *default
    database: storage/production.sqlite3
  queue:
    <<: *default
    database: storage/production_queue.sqlite3
    migrations_paths: db/queue_migrate
```

```ruby
# config/environments/production.rb
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }
```

### Development Mode

By default, Rails development uses the `:async` adapter (in-process, in-memory). Jobs run but are lost on restart. To use Solid Queue in dev:

```ruby
# config/environments/development.rb
config.active_job.queue_adapter = :solid_queue
config.solid_queue.connects_to = { database: { writing: :queue } }
```

Then run `bin/jobs start` in a separate terminal.

### Transactional Integrity

When Solid Queue shares the app's database, jobs are enqueued inside the same transaction as your application data. This can be powerful (job only exists if data is committed) or dangerous (you depend on this behavior, then switch backends).

```ruby
# Opt in to safe behavior: enqueue AFTER transaction commits
class ApplicationJob < ActiveJob::Base
  self.enqueue_after_transaction_commit = true
end
```

### Monitoring with mission_control-jobs

```ruby
# Gemfile
gem "mission_control-jobs"

# config/routes.rb
mount MissionControl::Jobs::Engine, at: "/jobs"
```

Provides a web UI for viewing job statuses, retrying failed jobs, and queue monitoring.

### Handling Enqueue Errors

```ruby
# Solid Queue raises SolidQueue::Job::EnqueueError on DB errors during enqueue.
# This is different from ActiveJob::EnqueueError.

# Check if enqueue succeeded:
job = MyJob.perform_later(args)
if job.successfully_enqueued?
  # Job is in the queue
else
  # Enqueue failed — handle gracefully
  Rails.logger.error("Failed to enqueue #{job.class}")
end
```

---

## ActionMailer Integration

### Basic Usage

```ruby
# Async (default in production) — uses Active Job
UserMailer.welcome(user).deliver_later

# With delay
UserMailer.welcome(user).deliver_later(wait: 1.hour)
UserMailer.welcome(user).deliver_later(wait_until: user.created_at + 1.day)

# Specific queue
UserMailer.welcome(user).deliver_later(queue: :mailers)

# Synchronous — bypasses Active Job entirely
UserMailer.welcome(user).deliver_now
```

### Locale Preservation

```ruby
# Active Job preserves I18n.locale from enqueue time
I18n.locale = :fr
UserMailer.welcome(user).deliver_later
# Email will be generated in French, even if worker locale is :en
```

### Error Handling for Mailer Jobs

```ruby
# Mailer jobs use ActionMailer::MailDeliveryJob, NOT ApplicationJob.
# So rescue_from in ApplicationJob won't catch mailer errors.

class ApplicationMailer < ActionMailer::Base
  ActionMailer::MailDeliveryJob.rescue_from(Exception) do |exception|
    Rails.error.report(exception)
    raise exception
  end
end
```

---

## Concurrency Controls

### Basic Concurrency Limit

```ruby
class ImportJob < ApplicationJob
  # Only 1 import per account at a time
  limits_concurrency to: 1, key: ->(account) { account }, duration: 30.minutes

  def perform(account)
    account.run_import
  end
end
```

### Shared Concurrency Group

```ruby
# Two different job classes sharing a concurrency limit
class MoveContactJob < ApplicationJob
  limits_concurrency key: ->(contact) { contact },
                     duration: 15.minutes,
                     group: "ContactActions"

  def perform(contact)
    # ...
  end
end

class MergeContactJob < ApplicationJob
  limits_concurrency key: ->(contact) { contact },
                     duration: 15.minutes,
                     group: "ContactActions"

  def perform(contact)
    # Only runs when no MoveContactJob for same contact is running
  end
end
```

### How It Works

When a job with `limits_concurrency` is enqueued:
1. Solid Queue checks if the concurrency limit is reached
2. If under limit → job goes to `ready_executions` (processable)
3. If at limit → job goes to `blocked_executions` (waiting)
4. When a running job completes → Solid Queue unblocks the next waiting job
5. `duration` is a safety valve: after it expires, blocked jobs are released even if the limit wasn't freed

---

## Job Continuations

### When to Use

- Jobs processing large datasets (thousands of records)
- Multi-phase jobs (validate → process → finalize)
- Jobs that may be interrupted by deploys or queue shutdowns
- Any job where restarting from scratch is expensive

### Step-by-Step Pattern

```ruby
class DataMigrationJob < ApplicationJob
  include ActiveJob::Continuable

  def perform(migration_id)
    @migration = DataMigration.find(migration_id)

    step :validate do
      @migration.validate_schema!
    end

    step :migrate_records do |step|
      @migration.records.find_each(start: step.cursor) do |record|
        record.migrate!
        step.advance! from: record.id
      end
    end

    step :update_indexes do
      @migration.rebuild_indexes!
    end

    step :complete do
      @migration.update!(status: :completed, completed_at: Time.current)
    end
  end
end
```

### How It Works

1. Each `step` is recorded as the job progresses
2. If the job is interrupted (deploy, shutdown, crash), it's re-enqueued
3. On retry, completed steps are skipped
4. Steps with cursors (`step.advance!`) resume from the last recorded position
5. Code BEFORE the first `step` runs every time (use for setup like `@migration = ...`)

---

## Recurring Tasks

### Configuration

```yaml
# config/recurring.yml
production:
  # Simple job
  daily_cleanup:
    class: CleanupJob
    schedule: every day at 3am

  # Job with arguments
  weekly_report:
    class: WeeklyReportJob
    args: ["summary", { include_inactive: false }]
    schedule: every Monday at 9am

  # Raw command (no job class needed)
  prune_sessions:
    command: "Session.where('updated_at < ?', 30.days.ago).delete_all"
    schedule: every day at 4am

  # Frequent task
  health_ping:
    class: HealthPingJob
    schedule: every 5 minutes
```

### Schedule Syntax

Uses [Fugit](https://github.com/floraison/fugit) for parsing:

```yaml
schedule: every second
schedule: every 5 minutes
schedule: every hour
schedule: every day at 3am
schedule: every Monday at 9am
schedule: "0 3 * * *"          # Cron syntax
schedule: every 1st of month at noon
```

---

## Performance Patterns

### Batch Processing in Jobs

```ruby
class ProcessRecordsJob < ApplicationJob
  BATCH_SIZE = 1000

  def perform(start_id = 0)
    records = Record.where("id > ?", start_id).order(:id).limit(BATCH_SIZE)
    return if records.empty?

    records.each { |r| r.process! }

    # Self-enqueue for next batch
    if records.size == BATCH_SIZE
      self.class.perform_later(records.last.id)
    end
  end
end
```

### Avoid N+1 in Jobs

```ruby
class NotifyFollowersJob < ApplicationJob
  def perform(post)
    # ❌ N+1 — each follower triggers a query
    post.author.followers.each { |f| f.notify(post) }

    # ✅ Eager load
    post.author.followers.includes(:notification_preferences).find_each do |follower|
      follower.notify(post) if follower.notification_preferences.posts?
    end
  end
end
```

### Timeouts for External Calls

```ruby
class WebhookDeliveryJob < ApplicationJob
  retry_on Net::OpenTimeout, wait: :polynomially_longer, attempts: 5
  retry_on Net::ReadTimeout, wait: :polynomially_longer, attempts: 5
  retry_on Faraday::TimeoutError, wait: :polynomially_longer, attempts: 5

  def perform(webhook)
    # ALWAYS set timeouts on external HTTP calls
    response = Faraday.post(webhook.url, webhook.payload.to_json) do |req|
      req.options.timeout = 10        # Read timeout
      req.options.open_timeout = 5    # Connection timeout
      req.headers["Content-Type"] = "application/json"
    end

    webhook.update!(
      last_delivered_at: Time.current,
      last_status: response.status
    )
  end
end
```

---

## Anti-Patterns

### 1. Fat Jobs

```ruby
# ❌ Bad — one massive job doing everything
class ProcessOrderJob < ApplicationJob
  def perform(order)
    charge_payment(order)
    send_confirmation_email(order)
    update_inventory(order)
    notify_warehouse(order)
    sync_to_crm(order)
  end
end

# ✅ Good — chain of focused jobs
class ProcessOrderJob < ApplicationJob
  def perform(order)
    PaymentService.charge(order)
    order.update!(status: :paid)

    # Enqueue downstream jobs
    SendConfirmationJob.perform_later(order)
    UpdateInventoryJob.perform_later(order)
    NotifyWarehouseJob.perform_later(order)
    CrmSyncJob.perform_later(order)
  end
end
```

### 2. Passing Complex State

```ruby
# ❌ Bad — passing computed state (not serializable, stale)
result = expensive_computation(data)
ProcessResultJob.perform_later(result)

# ✅ Good — pass identifiers, let the job compute
ProcessResultJob.perform_later(data_id)

# Or persist state and pass the record
result = ComputationResult.create!(data: data, output: expensive_computation(data))
ProcessResultJob.perform_later(result)
```

### 3. Silent Failures

```ruby
# ❌ Bad — swallows all errors
class DangerousJob < ApplicationJob
  def perform(record)
    record.process!
  rescue => e
    Rails.logger.error(e.message)
    # Error is lost. No retry. No alert.
  end
end

# ✅ Good — explicit retry/discard strategy
class SafeJob < ApplicationJob
  retry_on StandardError, wait: :polynomially_longer, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(record)
    record.process!
  end
end
```

### 4. Time-Dependent Logic Without freeze_time

```ruby
# ❌ Bad test — flaky on slow CI
test "schedules for tomorrow" do
  ReminderJob.perform_later(user)
  assert_enqueued_with(at: Date.tomorrow.noon)  # May fail if clock ticks
end

# ✅ Good test
test "schedules for tomorrow" do
  freeze_time do
    ReminderJob.perform_later(user)
    assert_enqueued_with(at: Date.tomorrow.noon)
  end
end
```

---

## Migration from Other Backends

### From Sidekiq

| Sidekiq | Active Job (Solid Queue) |
|---------|--------------------------|
| `include Sidekiq::Job` | `< ApplicationJob` |
| `perform_async(args)` | `perform_later(args)` |
| `perform_in(5.minutes, args)` | `set(wait: 5.minutes).perform_later(args)` |
| `sidekiq_options queue: :critical` | `queue_as :critical` |
| `sidekiq_retry_in { \|count\| count * 30 }` | `retry_on Error, wait: ->(n) { n * 30 }` |
| `Sidekiq::Cron` | `config/recurring.yml` |
| Redis | Database (SQLite/Postgres/MySQL) |
| `Sidekiq::Testing.inline!` | `perform_enqueued_jobs { }` |
| `Sidekiq::Testing.fake!` | Default test behavior (jobs enqueued, not run) |

### From Delayed Job

| Delayed Job | Active Job (Solid Queue) |
|-------------|--------------------------|
| `object.delay.method` | Create a job class explicitly |
| `handle_asynchronously :method` | `MethodJob.perform_later(object)` |
| `Delayed::Job.enqueue(job)` | `MyJob.perform_later(args)` |
| `run_at: 5.minutes.from_now` | `.set(wait: 5.minutes).perform_later` |
| `priority: 0` | `queue_with_priority 0` |
| `queue: "mailers"` | `queue_as :mailers` |

---

## Quick Reference Card

```ruby
# Generate
bin/rails generate job my_task

# Enqueue
MyJob.perform_later(args)                      # ASAP
MyJob.set(wait: 5.minutes).perform_later(args) # Delayed
MyJob.set(wait_until: time).perform_later(args) # Scheduled
MyJob.set(queue: :critical).perform_later(args) # Specific queue
MyJob.set(priority: 0).perform_later(args)      # High priority
MyJob.perform_now(args)                         # Synchronous

# Bulk enqueue
ActiveJob.perform_all_later([MyJob.new(a), MyJob.new(b)])

# Error handling
retry_on ErrorClass, wait: :polynomially_longer, attempts: 5
discard_on ErrorClass
rescue_from ErrorClass { |e| handle(e) }

# Queue config
queue_as :my_queue
queue_with_priority 10

# Callbacks
before_enqueue / around_enqueue / after_enqueue
before_perform / around_perform / after_perform
after_discard

# Testing
assert_enqueued_with(job: MyJob, args: [record], queue: "default")
assert_enqueued_jobs 3
assert_no_enqueued_jobs
perform_enqueued_jobs { MyJob.perform_later(args) }

# Start workers
bin/jobs start
```
