# Rails Testing Reference

Detailed patterns, examples, and edge cases for the Rails testing ecosystem. This supplements the main SKILL.md with deeper coverage.

> **See also:** The **minitest** skill for assertion reference, fixture deep-dive, and TDD workflow.

---

## Table of Contents

1. [Test Helper Configuration](#test-helper-configuration)
2. [Request Tests In Depth](#request-tests-in-depth)
3. [System Tests In Depth](#system-tests-in-depth)
4. [Mailer Tests In Depth](#mailer-tests-in-depth)
5. [Job Tests In Depth](#job-tests-in-depth)
6. [Action Cable Tests In Depth](#action-cable-tests-in-depth)
7. [Helper Tests In Depth](#helper-tests-in-depth)
8. [View Tests In Depth](#view-tests-in-depth)
9. [Fixture Strategies](#fixture-strategies)
10. [Parallel Testing Details](#parallel-testing-details)
11. [Test Database Strategies](#test-database-strategies)
12. [CI Configuration Patterns](#ci-configuration-patterns)
13. [Time Testing Patterns](#time-testing-patterns)
14. [File Upload Testing](#file-upload-testing)
15. [Testing Turbo & Stimulus](#testing-turbo--stimulus)
16. [Common Flaky Test Patterns](#common-flaky-test-patterns)
17. [Test Coverage](#test-coverage)
18. [Rails-Specific Assertions Reference](#rails-specific-assertions-reference)

---

## Test Helper Configuration

### Minimal `test_helper.rb`

```ruby
ENV["RAILS_ENV"] ||= "test"
require_relative "../config/environment"
require "rails/test_help"

# Auto-require all support files
Dir[Rails.root.join("test", "support", "**", "*.rb")].each { |f| require f }

module ActiveSupport
  class TestCase
    # Run tests in parallel with specified workers
    parallelize(workers: :number_of_processors)

    # Setup all fixtures in test/fixtures/*.yml
    fixtures :all

    # Add more helper methods to be used by all tests here...
  end
end
```

### Shared Test Helpers

Organize helpers in `test/support/`:

```ruby
# test/support/authentication_helper.rb
module AuthenticationHelper
  def sign_in_as(user, password: "password")
    post session_url, params: { email: user.email, password: password }
    assert_response :redirect
    follow_redirect!
  end

  def sign_out
    delete session_url
  end
end

# Include in integration/request tests
class ActionDispatch::IntegrationTest
  include AuthenticationHelper
end
```

```ruby
# test/support/json_helper.rb
module JsonHelper
  def json_response
    JSON.parse(response.body, symbolize_names: true)
  end

  def json_body
    response.parsed_body
  end
end

class ActionDispatch::IntegrationTest
  include JsonHelper
end
```

### Test Environment Config

```ruby
# config/environments/test.rb

Rails.application.configure do
  config.eager_load = ENV["CI"].present?

  # Speed up password hashing
  config.active_model.min_cost_bcrypt = true  # Rails 8+
  # Or for older Rails:
  # BCrypt::Engine.cost = 1

  # Raise on missing translations
  config.i18n.raise_on_missing_translations = true

  # Raise on missing callback actions
  config.action_controller.raise_on_missing_callback_actions = true

  # Disable logging for speed (or set to :warn)
  config.logger = Logger.new(nil)
  config.log_level = :fatal

  # Active Job uses test adapter by default
  config.active_job.queue_adapter = :test

  # Raise errors for missing template variables
  config.action_view.raise_on_missing_translations = true
end
```

---

## Request Tests In Depth

### Base Class and Inheritance

Request tests inherit from `ActionDispatch::IntegrationTest`, which gives you the full HTTP verb methods, session, cookies, and redirect following.

```ruby
class ArticlesControllerTest < ActionDispatch::IntegrationTest
  # This is the modern approach — despite living in test/controllers/
  # These ARE integration/request tests, not "functional" controller tests
end
```

### All HTTP Methods

```ruby
get    url, params: {}, headers: {}, env: {}, xhr: false, as: nil
post   url, params: {}, headers: {}, env: {}, xhr: false, as: nil
patch  url, params: {}, headers: {}, env: {}, xhr: false, as: nil
put    url, params: {}, headers: {}, env: {}, xhr: false, as: nil
delete url, params: {}, headers: {}, env: {}, xhr: false, as: nil
head   url, params: {}, headers: {}, env: {}, xhr: false, as: nil
```

### Testing JSON APIs

```ruby
class Api::V1::ArticlesTest < ActionDispatch::IntegrationTest
  setup do
    @article = articles(:published)
    @headers = {
      "Authorization" => "Bearer #{api_tokens(:admin).token}",
      "Accept" => "application/json"
    }
  end

  test "GET /api/v1/articles returns articles" do
    get api_v1_articles_url, headers: @headers, as: :json
    assert_response :success

    body = response.parsed_body
    assert_kind_of Array, body
    assert body.any? { |a| a["id"] == @article.id }
  end

  test "POST /api/v1/articles creates article" do
    assert_difference "Article.count", 1 do
      post api_v1_articles_url,
        params: { article: { title: "API Created", body: "Via JSON" } },
        headers: @headers,
        as: :json
    end
    assert_response :created

    body = response.parsed_body
    assert_equal "API Created", body["title"]
  end

  test "POST /api/v1/articles with invalid data returns errors" do
    assert_no_difference "Article.count" do
      post api_v1_articles_url,
        params: { article: { title: "" } },
        headers: @headers,
        as: :json
    end
    assert_response :unprocessable_entity

    body = response.parsed_body
    assert_includes body["errors"]["title"], "can't be blank"
  end

  test "GET /api/v1/articles without auth returns 401" do
    get api_v1_articles_url, as: :json
    assert_response :unauthorized
  end
end
```

### Testing Redirects

```ruby
test "non-admin is redirected to root" do
  sign_in_as users(:regular)
  get admin_dashboard_url
  assert_redirected_to root_url

  # Follow the redirect and check the final page
  follow_redirect!
  assert_response :success
  assert_select ".flash-alert", /not authorized/i
end
```

### Testing Flash Messages

```ruby
test "displays success flash after create" do
  sign_in_as users(:admin)
  post articles_url, params: { article: { title: "New", body: "Article" } }
  assert_equal "Article was successfully created.", flash[:notice]
end
```

### Testing with Cookies and Sessions

```ruby
test "remembers user preference" do
  get articles_url
  assert_nil cookies[:view_mode]

  patch preferences_url, params: { view_mode: "compact" }
  assert_equal "compact", cookies[:view_mode]
end
```

### Testing File Uploads

```ruby
test "uploads avatar" do
  sign_in_as users(:admin)

  avatar = fixture_file_upload("test/fixtures/files/avatar.png", "image/png")

  patch user_url(@user), params: { user: { avatar: avatar } }
  assert_redirected_to user_url(@user)

  @user.reload
  assert @user.avatar.attached?
end
```

### Testing Turbo Stream Responses

```ruby
test "create returns turbo stream" do
  sign_in_as users(:admin)

  post articles_url, params: { article: { title: "Turbo", body: "Stream" } },
    as: :turbo_stream

  assert_response :success
  assert_match "turbo-stream", response.body
  assert_match "append", response.body
end
```

### DOM Assertions in Request Tests

```ruby
test "index page shows articles" do
  get articles_url
  assert_response :success

  # Check specific HTML elements
  assert_select "h1", "Articles"
  assert_select ".article", count: Article.count
  assert_select ".article .title", @article.title

  # Nested assertions
  assert_select "ul.articles" do
    assert_select "li", minimum: 1
  end

  # Check absence
  assert_select ".admin-controls", count: 0
end
```

### Testing with Basic Auth

```ruby
test "requires basic auth" do
  get admin_url
  assert_response :unauthorized

  credentials = ActionController::HttpAuthentication::Basic.encode_credentials("admin", "secret")
  get admin_url, headers: { "Authorization" => credentials }
  assert_response :success
end
```

---

## System Tests In Depth

### Capybara DSL Quick Reference

**Navigation:**
```ruby
visit articles_path
visit "/articles"
go_back
go_forward
refresh
```

**Finders:**
```ruby
find("#element-id")
find(".class-name")
find("input[name='email']")
find(:css, "button.primary")
find(:xpath, "//button[@type='submit']")
find_button("Submit")
find_field("Email")
find_link("Sign in")
```

**Interactions:**
```ruby
click_on "Button or Link Text"
click_button "Submit"
click_link "Sign in"
fill_in "Email", with: "user@example.com"
choose "Option A"          # radio button
check "Remember me"        # checkbox
uncheck "Subscribe"        # checkbox
select "Admin", from: "Role"
attach_file "Avatar", Rails.root.join("test/fixtures/files/avatar.png")
```

**Assertions:**
```ruby
assert_text "Welcome"
assert_no_text "Error"
assert_selector "h1", text: "Dashboard"
assert_no_selector ".error-message"
assert_current_path articles_path
assert_link "Edit"
assert_button "Submit"
assert_field "Email", with: "user@example.com"
assert_checked_field "Remember me"
assert_unchecked_field "Subscribe"
```

**Scoping:**
```ruby
within "#sidebar" do
  click_on "Settings"
end

within_table "Users" do
  assert_text "admin@example.com"
end

within_fieldset "Address" do
  fill_in "City", with: "Portland"
end
```

### Configuring Cuprite (Faster Alternative to Selenium)

```ruby
# Gemfile
group :test do
  gem "cuprite"
end

# test/application_system_test_case.rb
require "test_helper"
require "capybara/cuprite"

class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  driven_by :cuprite, options: {
    window_size: [1400, 1400],
    browser_options: { "no-sandbox": nil },
    headless: true,
    process_timeout: 10,
    timeout: 10
  }
end
```

### Debugging System Tests

```ruby
# Take screenshot at any point
take_screenshot

# Save and open current page HTML
save_page

# Print current page text (useful in CI logs)
puts page.text

# Use visible browser for debugging
# Temporarily change to:
driven_by :selenium, using: :chrome  # non-headless

# Pause execution (add a breakpoint)
debugger  # Rails 7+
```

### Testing JavaScript Interactions

```ruby
test "live search filters results" do
  visit articles_path

  fill_in "Search", with: "rails"

  # Capybara auto-waits for this assertion
  assert_selector ".article", count: 2
  assert_text "Rails Guide"
  assert_no_text "Django Tutorial"
end

test "modal opens and closes" do
  visit articles_path

  click_on "Delete"
  assert_selector "#confirm-modal", visible: true
  assert_text "Are you sure?"

  click_on "Cancel"
  assert_no_selector "#confirm-modal", visible: true
end

test "drag and drop reorders items" do
  visit board_path(@board)

  source = find("#card-1")
  target = find("#card-3")
  source.drag_to(target)

  # Verify new order
  assert_selector "#card-list li:first-child", text: @card3.title
end
```

### System Tests with Turbo

```ruby
test "inline edit with turbo frame" do
  visit article_path(@article)

  # Click edit within a turbo frame
  within "#article-header" do
    click_on "Edit"
    # Turbo replaces the frame content
    fill_in "Title", with: "Updated Title"
    click_on "Save"
  end

  # Frame is replaced with show content
  within "#article-header" do
    assert_text "Updated Title"
    assert_no_selector "input[name='article[title]']"
  end
end
```

---

## Mailer Tests In Depth

### Testing Multipart Emails (HTML + Text)

```ruby
test "sends multipart welcome email" do
  email = UserMailer.welcome(users(:new_user))

  # Test HTML part
  assert_match "<h1>Welcome</h1>", email.html_part.body.to_s
  assert_match "click here", email.html_part.body.to_s

  # Test text part
  assert_match "Welcome", email.text_part.body.to_s
  refute_match "<h1>", email.text_part.body.to_s  # No HTML in text part
end
```

### Testing Attachments

```ruby
test "sends invoice with PDF attachment" do
  email = InvoiceMailer.send_invoice(invoices(:pending))

  assert_equal 1, email.attachments.size
  attachment = email.attachments.first
  assert_equal "invoice-001.pdf", attachment.filename
  assert_equal "application/pdf", attachment.content_type
end
```

### Testing Parameterized Mailers

```ruby
test "parameterized mailer uses correct tenant" do
  email = TenantMailer.with(tenant: tenants(:acme)).welcome(users(:admin))

  assert_match "Acme Corp", email.subject
  assert_equal ["noreply@acme.example.com"], email.from
end

test "enqueues parameterized mailer" do
  assert_enqueued_email_with TenantMailer.with(tenant: tenants(:acme)),
    :welcome, args: [users(:admin)] do
    TenantMailer.with(tenant: tenants(:acme)).welcome(users(:admin)).deliver_later
  end
end
```

### Mailer Previews (Not Tests, But Useful)

```ruby
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome
    UserMailer.welcome(User.first)
  end

  def password_reset
    UserMailer.password_reset(User.first)
  end
end

# View at: http://localhost:3000/rails/mailers/user_mailer/welcome
```

### Functional Email Testing (From Request Tests)

```ruby
test "password reset sends email" do
  user = users(:admin)

  assert_emails 1 do
    post password_resets_url, params: { email: user.email }
  end

  email = ActionMailer::Base.deliveries.last
  assert_equal [user.email], email.to
  assert_match "reset", email.subject.downcase
end
```

---

## Job Tests In Depth

### Testing Job Enqueuing Details

```ruby
class NotificationJobTest < ActiveJob::TestCase
  test "enqueues with correct queue" do
    assert_enqueued_with(job: NotificationJob, queue: "notifications") do
      NotificationJob.perform_later(users(:admin), "Hello")
    end
  end

  test "enqueues with scheduled time" do
    assert_enqueued_with(job: ReminderJob, at: 1.hour.from_now.to_f) do
      ReminderJob.set(wait: 1.hour).perform_later(users(:admin))
    end
  end

  test "enqueues multiple jobs" do
    assert_enqueued_jobs 3 do
      3.times { NotificationJob.perform_later(users(:admin), "Hello") }
    end
  end
end
```

### Testing Job Side Effects

```ruby
test "import job processes CSV and creates records" do
  csv_path = file_fixture("products.csv")

  assert_difference "Product.count", 5 do
    perform_enqueued_jobs do
      ImportJob.perform_later(csv_path.to_s)
    end
  end
end

test "cleanup job removes old records" do
  assert_difference "Session.count", -3 do
    perform_enqueued_jobs do
      CleanupJob.perform_later
    end
  end
end
```

### Testing Jobs That Send Emails

```ruby
test "welcome job sends email" do
  user = users(:new_user)

  assert_emails 1 do
    perform_enqueued_jobs do
      WelcomeJob.perform_later(user)
    end
  end
end
```

### Testing Jobs That Enqueue Other Jobs

```ruby
test "batch job enqueues individual jobs" do
  users = [users(:admin), users(:regular)]

  assert_enqueued_jobs 2, only: NotificationJob do
    BatchNotifyJob.perform_now(users)
  end
end
```

### Filtering Performed Jobs

```ruby
test "only performs specific job types" do
  perform_enqueued_jobs(only: ProcessOrderJob) do
    ProcessOrderJob.perform_later(orders(:new))
    NotificationJob.perform_later(users(:admin), "Processed")
  end

  # ProcessOrderJob ran, NotificationJob is still queued
  assert_enqueued_jobs 1, only: NotificationJob
end
```

---

## Action Cable Tests In Depth

### Channel with Custom Methods

```ruby
class ChatChannelTest < ActionCable::Channel::TestCase
  test "receives broadcasted messages" do
    subscribe room: "general"

    perform :speak, message: "Hello!"

    # Assert the broadcast happened
    assert_broadcast_on("chat_general", message: "Hello!", user: "anonymous")
  end
end
```

### Testing Streams

```ruby
class NotificationsChannelTest < ActionCable::Channel::TestCase
  test "streams for current user" do
    stub_connection current_user: users(:admin)
    subscribe

    assert_has_stream_for users(:admin)
  end

  test "streams for specific model" do
    stub_connection current_user: users(:admin)
    project = projects(:active)
    subscribe project_id: project.id

    assert_has_stream_for project
  end
end
```

### Broadcast Assertions from Models/Controllers

```ruby
class MessageTest < ActiveSupport::TestCase
  include ActionCable::TestHelper

  test "broadcasts new message" do
    room = rooms(:general)

    assert_broadcasts("chat_#{room.id}", 1) do
      Message.create!(room: room, body: "Hello!", user: users(:admin))
    end
  end

  test "broadcasts with specific payload" do
    room = rooms(:general)

    assert_broadcast_on("chat_#{room.id}", {
      body: "Hello!",
      user: "Admin"
    }) do
      Message.create!(room: room, body: "Hello!", user: users(:admin))
    end
  end
end
```

---

## Helper Tests In Depth

```ruby
class ApplicationHelperTest < ActionView::TestCase
  test "page_title with custom title" do
    assert_equal "My App | Dashboard", page_title("Dashboard")
  end

  test "page_title with default" do
    assert_equal "My App", page_title
  end

  test "renders markdown to HTML" do
    result = render_markdown("**bold** and *italic*")
    assert_dom_equal "<p><strong>bold</strong> and <em>italic</em></p>", result
  end

  test "time_ago_in_words helper" do
    # ActionView helpers are available
    assert_match /ago/, time_ago_in_words(3.hours.ago)
  end
end
```

### Testing Helpers That Use Request Context

```ruby
class NavigationHelperTest < ActionView::TestCase
  test "active_link_class returns active for current path" do
    # Stub the request
    controller.request = ActionDispatch::TestRequest.create
    controller.request.path = "/articles"

    assert_equal "active", active_link_class("/articles")
    assert_nil active_link_class("/users")
  end
end
```

---

## View Tests In Depth

### Testing Partials

```ruby
class ArticlePartialTest < ActionView::TestCase
  test "renders article title" do
    article = articles(:published)
    render partial: "articles/article", locals: { article: article }

    assert_includes rendered, article.title
    assert_dom "a[href=?]", article_path(article)
  end
end
```

### Parsed Content Testing (Rails 7.1+)

```ruby
class ApiArticleViewTest < ActionView::TestCase
  test "renders JSON correctly" do
    article = articles(:published)
    render formats: :json, partial: "articles/article", locals: { article: article }

    json = rendered.json
    assert_equal article.title, json[:title]
    assert_equal article.id, json[:id]
  end
end
```

---

## Fixture Strategies

### Fixtures vs Factories — The Definitive Answer

**Use fixtures (Rails default).** Here's why:

| Factor | Fixtures | Factory Bot |
|--------|----------|-------------|
| Speed | Loaded once via transaction | Created per test |
| Data consistency | Same data every run | Random/sequential |
| Association handling | Reference by name | Cascading creation |
| Test isolation | Shared read, transactional write | Fully isolated |
| Setup time | Near zero | Linear with test count |
| Debugging | Known data, inspectable | Generated, harder to trace |

### When Factories Might Be Acceptable

- Truly dynamic data requirements (rare)
- When the project already uses Factory Bot extensively (don't rewrite)
- Combinatorial testing with many attribute variations

### Fixture Tips

```yaml
# test/fixtures/users.yml

# Use ERB for dynamic values
admin:
  id: <%= Digest::UUID.uuid_v5(Digest::UUID::OID_NAMESPACE, "admin") %>
  email: "admin@example.com"
  name: "Admin User"
  password_digest: <%= BCrypt::Password.create("password", cost: 1) %>
  admin: true
  created_at: <%= 1.year.ago %>

regular:
  id: <%= Digest::UUID.uuid_v5(Digest::UUID::OID_NAMESPACE, "regular") %>
  email: "regular@example.com"
  name: "Regular User"
  password_digest: <%= BCrypt::Password.create("password", cost: 1) %>
  admin: false

# Reference associations by name, not ID
# test/fixtures/articles.yml
published:
  title: "Published Article"
  body: "This is published."
  author: admin          # References users(:admin)
  published_at: <%= 1.day.ago %>
  status: published

draft:
  title: "Draft Article"
  body: "Work in progress."
  author: regular
  status: draft
```

### File Fixtures

```ruby
# test/fixtures/files/sample.csv
# test/fixtures/files/avatar.png
# test/fixtures/files/document.pdf

# Access in tests:
file_fixture("sample.csv")         # Returns Pathname
file_fixture("sample.csv").read    # Read contents
```

### Active Storage Fixtures

```yaml
# test/fixtures/active_storage/blobs.yml
avatar_blob: <%= ActiveStorage::FixtureSet.blob(
  filename: "avatar.png",
  content_type: "image/png"
) %>

# test/fixtures/active_storage/attachments.yml
admin_avatar:
  name: avatar
  record: admin (User)
  blob: avatar_blob
```

---

## Parallel Testing Details

### Process vs Thread Comparison

| Feature | Processes (default) | Threads |
|---------|-------------------|---------|
| Isolation | Full process isolation | Shared memory |
| Database | Separate DB per worker | Shared DB (careful!) |
| Speed | Overhead from forking | Less overhead |
| Compatibility | Works with everything | Thread-safety required |
| Best for | CRuby (MRI) | JRuby, TruffleRuby |

### Common Parallel Testing Issues

**Issue: Shared file system resources**
```ruby
# BAD — workers overwrite each other
test "generates report" do
  ReportGenerator.generate(path: "/tmp/report.pdf")
  assert File.exist?("/tmp/report.pdf")
end

# GOOD — unique paths per worker
parallelize_setup do |worker|
  FileUtils.mkdir_p(Rails.root.join("tmp/test-#{worker}"))
end

test "generates report" do
  path = Rails.root.join("tmp/test-#{ENV['TEST_ENV_NUMBER']}/report.pdf")
  ReportGenerator.generate(path: path)
  assert File.exist?(path)
end
```

**Issue: External service rate limits**
```ruby
# Use VCR or WebMock to avoid hitting real services in parallel
# Or use per-worker API keys
parallelize_setup do |worker|
  ENV["API_KEY"] = "test-key-#{worker}"
end
```

---

## Test Database Strategies

### Schema Loading

```bash
# Prepare test DB from schema.rb/structure.sql
bin/rails db:test:prepare

# Rebuild completely (drop + create + load schema)
bin/rails db:test:purge
bin/rails db:test:prepare

# Or in one command
RAILS_ENV=test bin/rails db:reset
```

### Multiple Databases

```ruby
# config/database.yml
test:
  primary:
    <<: *default
    database: myapp_test
  cache:
    <<: *default
    database: myapp_test_cache
    migrations_paths: db/cache_migrate
```

Rails wraps each database in its own transaction during tests.

### Seed Data in Tests

```ruby
# Generally avoid seeds in tests — use fixtures instead
# But if you must:
class ActiveSupport::TestCase
  setup do
    # Only if absolutely necessary
    Rails.application.load_seed if SomeModel.count.zero?
  end
end
```

---

## CI Configuration Patterns

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports: ["6379:6379"]

    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/myapp_test
      CI: true

    steps:
      - uses: actions/checkout@v4

      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Setup DB
        run: bin/rails db:test:prepare

      - name: Run tests
        run: bin/rails test

      - name: Run system tests
        run: bin/rails test:system

      - name: Upload screenshots
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: screenshots
          path: tmp/screenshots/
```

### Separate System Test Job (Recommended)

```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - # ... setup ...
      - run: bin/rails test
        env:
          PARALLEL_WORKERS: 4

  system-tests:
    runs-on: ubuntu-latest
    steps:
      - # ... setup ...
      - uses: browser-actions/setup-chrome@v1
      - run: bin/rails test:system
```

---

## Time Testing Patterns

### Available Time Helpers

```ruby
# Freeze time at current moment
freeze_time do
  now = Time.current
  sleep 1
  assert_equal now, Time.current  # Time didn't advance
end

# Travel to specific point
travel_to DateTime.new(2024, 12, 25, 9, 0, 0) do
  assert_equal "Wednesday", Date.current.strftime("%A")
end

# Travel by duration
travel 2.days do
  assert user.trial_expired?  # If trial was 1 day
end

# Travel back (undo travel)
travel_to 1.week.ago
# ... do stuff ...
travel_back  # Return to real time
```

### Common Time Testing Patterns

```ruby
test "subscription expires after 30 days" do
  sub = Subscription.create!(started_at: Time.current)
  refute sub.expired?

  travel 29.days do
    refute sub.expired?
  end

  travel 31.days do
    assert sub.expired?
  end
end

test "scheduled job runs at correct time" do
  travel_to Time.zone.local(2024, 1, 15, 8, 59) do
    refute DailyDigestJob.should_run?
  end

  travel_to Time.zone.local(2024, 1, 15, 9, 0) do
    assert DailyDigestJob.should_run?
  end
end
```

---

## File Upload Testing

### In Request Tests

```ruby
test "uploads a document" do
  sign_in_as users(:admin)
  file = fixture_file_upload("test/fixtures/files/document.pdf", "application/pdf")

  assert_difference "Document.count", 1 do
    post documents_url, params: { document: { file: file, title: "Test Doc" } }
  end

  assert_redirected_to documents_url
  assert Document.last.file.attached?
end
```

### In System Tests

```ruby
test "uploads avatar via form" do
  sign_in users(:admin)
  visit edit_profile_path

  attach_file "Avatar", Rails.root.join("test/fixtures/files/avatar.png")
  click_on "Save"

  assert_text "Profile updated"
end
```

### Multiple File Uploads

```ruby
test "uploads multiple images" do
  sign_in_as users(:admin)

  files = [
    fixture_file_upload("test/fixtures/files/photo1.jpg", "image/jpeg"),
    fixture_file_upload("test/fixtures/files/photo2.jpg", "image/jpeg")
  ]

  post gallery_images_url, params: { images: files }
  assert_response :redirect
end
```

---

## Testing Turbo & Stimulus

### Turbo Frame Responses (Request Tests)

```ruby
test "edit renders turbo frame" do
  sign_in_as users(:admin)

  get edit_article_url(@article),
    headers: { "Turbo-Frame" => "article_#{@article.id}" }

  assert_response :success
  assert_select "turbo-frame#article_#{@article.id}"
end
```

### Turbo Stream Responses (Request Tests)

```ruby
test "create returns turbo stream when requested" do
  sign_in_as users(:admin)

  post articles_url,
    params: { article: { title: "Turbo", body: "Test" } },
    as: :turbo_stream

  assert_response :success
  assert_includes response.media_type, "turbo-stream"
end
```

### Stimulus in System Tests

```ruby
test "character counter updates live" do
  visit new_article_path

  fill_in "Body", with: "x" * 100

  # Stimulus controller updates counter
  assert_selector "[data-character-count]", text: "100/500"

  fill_in "Body", with: "x" * 501
  assert_selector "[data-character-count]", text: "501/500"
  assert_selector "[data-character-count].over-limit"
end
```

---

## Common Flaky Test Patterns

### Problem: Database Race Conditions

```ruby
# FLAKY — parallel test might see different count
test "there are 5 users" do
  assert_equal 5, User.count
end

# STABLE — test relative changes
test "creating adds one user" do
  assert_difference "User.count", 1 do
    User.create!(email: "new@example.com")
  end
end
```

### Problem: Time-Dependent Logic

```ruby
# FLAKY — fails near midnight or in different timezones
test "created today" do
  article = Article.create!
  assert_equal Date.today, article.created_at.to_date
end

# STABLE — freeze time
test "created today" do
  freeze_time do
    article = Article.create!
    assert_equal Date.current, article.created_at.to_date
  end
end
```

### Problem: Random Ordering Assumptions

```ruby
# FLAKY — database doesn't guarantee order
test "first user is admin" do
  assert User.first.admin?
end

# STABLE — be explicit
test "admin fixture exists" do
  assert users(:admin).admin?
end
```

### Problem: Capybara Timing

```ruby
# FLAKY — element might not be there yet
test "shows notification" do
  click_on "Save"
  assert page.has_css?(".notification")  # Doesn't wait!

  # STABLE — use assertion that waits
  click_on "Save"
  assert_selector ".notification"  # Auto-waits
end
```

### Problem: External Services

```ruby
# FLAKY — depends on network, rate limits
test "geocodes address" do
  user = User.create!(address: "123 Main St")
  assert_not_nil user.latitude
end

# STABLE — stub external calls
test "geocodes address" do
  Geocoder::Lookup::Test.add_stub("123 Main St", [
    { "latitude" => 40.7, "longitude" => -74.0 }
  ])

  user = User.create!(address: "123 Main St")
  assert_in_delta 40.7, user.latitude, 0.01
end
```

---

## Test Coverage

### SimpleCov Setup

```ruby
# Gemfile
group :test do
  gem "simplecov", require: false
end

# test/test_helper.rb (MUST be first lines)
if ENV["COVERAGE"]
  require "simplecov"
  SimpleCov.start "rails" do
    add_filter "/test/"
    add_filter "/config/"
    minimum_coverage 80
    minimum_coverage_by_file 50
  end
end

# Run with coverage
COVERAGE=true bin/rails test
```

### What Coverage Numbers Actually Mean

- **Line coverage** — which lines executed (most common)
- **Branch coverage** — which conditional branches taken
- **100% coverage ≠ bug-free** — you can cover every line with bad assertions
- **Aim for 80-90%** — diminishing returns above that
- **Focus on business logic coverage** — models, services, policies

---

## Rails-Specific Assertions Reference

### Record Change Assertions

```ruby
# Count changes
assert_difference "Article.count", 1 do
  Article.create!(title: "New")
end

assert_difference "Article.count", -1 do
  @article.destroy
end

assert_no_difference "Article.count" do
  Article.new.save  # fails validation
end

# Multiple counts at once
assert_difference ["Article.count", "Comment.count"], 1 do
  Article.create_with_comment!(...)
end

# Value changes
assert_changes -> { @user.reload.name }, from: "Old", to: "New" do
  @user.update!(name: "New")
end

assert_no_changes -> { @user.reload.email } do
  @user.update!(name: "New Name")
end
```

### Response Assertions

```ruby
assert_response :success        # 200-299
assert_response :redirect       # 300-399
assert_response :missing        # 404
assert_response :error          # 500-599
assert_response 201             # specific status
assert_response :created        # symbolic status

assert_redirected_to article_path(@article)
assert_redirected_to root_url
assert_redirected_to %r{/articles/\d+}  # regex match
```

### Query Assertions (Rails 7.1+)

```ruby
# Assert exact query count
assert_queries_count(2) do
  User.find(1)
  User.find(2)
end

# Assert no queries (e.g., testing caching)
assert_no_queries do
  Rails.cache.read("key")
end

# Assert query pattern
assert_queries_match(/SELECT.*FROM.*users/) do
  User.all.to_a
end

# Assert no queries matching pattern
assert_no_queries_match(/INSERT/) do
  User.all.to_a
end
```

### Email Assertions

```ruby
# Count emails sent
assert_emails 1 do
  UserMailer.welcome(@user).deliver_now
end

assert_no_emails do
  # action that shouldn't send email
end

# Count enqueued emails
assert_enqueued_emails 1 do
  UserMailer.welcome(@user).deliver_later
end

# Assert specific email enqueued
assert_enqueued_email_with UserMailer, :welcome, args: [@user] do
  UserMailer.welcome(@user).deliver_later
end
```

### Error Reporting Assertions (Rails 7.1+)

```ruby
assert_error_reported(IOError) do
  Rails.error.report(IOError.new("disk full"))
end

assert_no_error_reported do
  perform_safe_operation
end
```

### Routing Assertions

```ruby
# Path generates correct route
assert_generates "/articles/1", controller: "articles", action: "show", id: "1"

# Route recognizes path
assert_recognizes({ controller: "articles", action: "show", id: "1" }, "/articles/1")

# Both directions at once
assert_routing "/articles/1", controller: "articles", action: "show", id: "1"
```

### DOM Assertions

```ruby
# Element exists
assert_select "h1", "Welcome"
assert_select "h1", /welcome/i
assert_select ".article", count: 3
assert_select ".article", minimum: 1
assert_select ".article", maximum: 10
assert_select ".error", false  # must not exist
assert_select ".error", count: 0  # same as above

# Nested assertions
assert_select "table tr" do
  assert_select "td", minimum: 3
end

# In email body
assert_dom_email do
  assert_select "a[href=?]", confirmation_url
end
```
