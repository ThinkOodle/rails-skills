# Active Model Reference

Detailed patterns, edge cases, and advanced usage for Active Model in Rails 8.1.

## Module Deep Dives

### ActiveModel::API vs ActiveModel::Model

Both provide form/controller integration. The difference:

- **`ActiveModel::API`** — The core interface. Includes: AttributeAssignment, Conversion, Naming, Translation, Validations.
- **`ActiveModel::Model`** — Includes `API` plus is designed for future Rails extensions. **Always prefer `Model` over `API`** unless you have a specific reason not to.

```ruby
# These are functionally equivalent today, but Model is future-proof:
class Foo
  include ActiveModel::API  # Just the basics
end

class Bar
  include ActiveModel::Model  # Basics + future extensions
end
```

Both support:
- Hash-based initialization: `Foo.new(name: "Jane")`
- `form_with model: @foo`
- `render @foo` (partial rendering)
- `valid?`, `errors`, all validation helpers
- `model_name`, `to_model`, `to_key`, `to_param`, `to_partial_path`

### ActiveModel::Attributes

Provides typed attributes with automatic casting and defaults. This is one of the most useful modules for form objects.

**Supported built-in types:**
- `:string` — casts to String
- `:integer` — casts to Integer
- `:float` — casts to Float
- `:decimal` — casts to BigDecimal
- `:boolean` — casts to true/false (handles `"1"`, `"true"`, `"yes"`, `0`, `"false"`, `"no"`, etc.)
- `:date` — casts to Date
- `:datetime` — casts to DateTime/Time
- `:time` — casts to Time

```ruby
class ImportJob
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :file_path, :string
  attribute :skip_header, :boolean, default: true
  attribute :delimiter, :string, default: ","
  attribute :batch_size, :integer, default: 1000
  attribute :started_at, :datetime
end

job = ImportJob.new(skip_header: "0", batch_size: "500")
job.skip_header   # => false (cast from "0")
job.batch_size     # => 500 (cast from "500")
job.delimiter      # => "," (default)
```

**`attribute_names` and `attributes` methods:**

```ruby
ImportJob.attribute_names
# => ["file_path", "skip_header", "delimiter", "batch_size", "started_at"]

job.attributes
# => {"file_path"=>nil, "skip_header"=>true, "delimiter"=>",", "batch_size"=>1000, "started_at"=>nil}
```

**Custom attribute types:**

```ruby
class MoneyType < ActiveModel::Type::Value
  def cast(value)
    case value
    when String then BigDecimal(value.gsub(/[$,]/, ""))
    when Numeric then BigDecimal(value.to_s)
    else value
    end
  end
end

ActiveModel::Type.register(:money, MoneyType)

class Invoice
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :total, :money
end

Invoice.new(total: "$1,234.56").total  # => 0.123456e4 (BigDecimal)
```

### ActiveModel::Validations

Everything from Active Record validations works here. Full list of built-in validators:

```ruby
class Payment
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :amount, :decimal
  attribute :currency, :string
  attribute :card_number, :string
  attribute :email, :string
  attribute :terms_accepted, :boolean

  # Presence
  validates :amount, :currency, presence: true

  # Numericality
  validates :amount, numericality: { greater_than: 0, less_than_or_equal_to: 999_999 }

  # Inclusion
  validates :currency, inclusion: { in: %w[usd eur gbp], message: "%{value} is not supported" }

  # Format
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }

  # Length
  validates :card_number, length: { is: 16 }

  # Acceptance (for checkboxes)
  validates :terms_accepted, acceptance: true

  # Custom validation method
  validate :amount_within_daily_limit

  private

  def amount_within_daily_limit
    return unless amount.present? && amount > 10_000
    errors.add(:amount, "exceeds daily limit of $10,000")
  end
end
```

**Conditional validations:**

```ruby
class Order
  include ActiveModel::Model

  attr_accessor :payment_type, :card_number, :po_number

  validates :card_number, presence: true, if: -> { payment_type == "credit_card" }
  validates :po_number, presence: true, if: :purchase_order?

  private

  def purchase_order?
    payment_type == "purchase_order"
  end
end
```

**Validation contexts:**

```ruby
class User
  include ActiveModel::Model

  attr_accessor :name, :email, :admin_notes

  validates :name, :email, presence: true
  validates :admin_notes, presence: true, on: :admin_review

  def valid_for_admin_review?
    valid?(:admin_review)
  end
end

user = User.new(name: "Jane", email: "jane@example.com")
user.valid?                # => true (default context)
user.valid?(:admin_review) # => false (admin_notes missing)
```

**Strict validations (raise instead of adding errors):**

```ruby
class ApiKey
  include ActiveModel::Model

  attr_accessor :token, :scope

  validates! :token, presence: true  # Raises ActiveModel::StrictValidationFailed
  validates :scope, presence: true   # Adds to errors normally
end
```

**Custom validators:**

```ruby
class EmailDomainValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    domain = value.split("@").last
    unless options[:allowed_domains].include?(domain)
      record.errors.add(attribute, "must be from an allowed domain")
    end
  end
end

class Employee
  include ActiveModel::Model
  attr_accessor :email

  validates :email, email_domain: { allowed_domains: ["company.com", "subsidiary.com"] }
end
```

### ActiveModel::Callbacks

**Key differences from Active Record callbacks:**

1. Use `extend` (not `include`)
2. You must call `run_callbacks(:event) { ... }` yourself
3. Method names can't end with `!`, `?`, or `=`

```ruby
class Processor
  include ActiveModel::Model
  extend ActiveModel::Callbacks

  define_model_callbacks :process
  define_model_callbacks :validate, only: [:before, :after]  # Skip :around

  before_process :log_start
  after_process :log_end
  around_process :measure_time

  def process
    run_callbacks(:process) do
      # actual work here
      do_the_thing
    end
  end

  private

  def log_start
    Rails.logger.info "Starting process..."
  end

  def log_end
    Rails.logger.info "Process complete"
  end

  def measure_time
    start = Time.current
    yield  # MUST yield in around callbacks
    Rails.logger.info "Took #{Time.current - start}s"
  end
end
```

**Aborting callbacks:**

```ruby
before_process :check_preconditions

def check_preconditions
  throw :abort unless ready?
end
```

When `:abort` is thrown, the callback chain stops and the block in `run_callbacks` is not executed.

**Class-based callbacks:**

```ruby
class AuditLogger
  def self.before_process(obj)
    AuditLog.record(action: "process_start", subject: obj.class.name)
  end
end

class Processor
  extend ActiveModel::Callbacks
  define_model_callbacks :process
  before_process AuditLogger
end
```

### ActiveModel::Dirty

Track attribute changes. Two approaches:

**Manual (without Attributes):**

```ruby
class Config
  include ActiveModel::Dirty

  define_attribute_methods :theme, :locale

  def theme
    @theme
  end

  def theme=(val)
    theme_will_change! unless val == @theme
    @theme = val
  end

  def locale
    @locale
  end

  def locale=(val)
    locale_will_change! unless val == @locale
    @locale = val
  end

  def save
    # persist...
    changes_applied  # Clears current changes, moves to previous_changes
  end

  def reload!
    clear_changes_information  # Clears everything
  end

  def rollback!
    restore_attributes  # Reverts to previous values
  end
end
```

**Automatic (with Attributes) — much simpler:**

When you use `ActiveModel::Attributes`, dirty tracking comes for free:

```ruby
class Config
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Dirty

  attribute :theme, :string, default: "light"
  attribute :locale, :string, default: "en"
end

config = Config.new
config.changed?         # => false
config.theme = "dark"
config.changed?         # => true
config.changes          # => {"theme"=>["light", "dark"]}
config.theme_was        # => "light"
config.theme_changed?   # => true
```

**Available query methods:**

| Method | Returns |
|--------|---------|
| `changed?` | `true` if any attribute changed |
| `changed` | Array of changed attribute names |
| `changes` | Hash: `{ attr => [old, new] }` |
| `changed_attributes` | Hash: `{ attr => old_value }` |
| `previous_changes` | Changes from before last `changes_applied` |
| `[attr]_changed?` | Did this specific attribute change? |
| `[attr]_was` | Previous value |
| `[attr]_change` | `[old, new]` or `nil` |
| `[attr]_previously_changed?` | Changed before last save? |
| `[attr]_previous_change` | `[old, new]` from before last save |

### ActiveModel::Serialization

Two levels of serialization support:

**Basic (`ActiveModel::Serialization`):**

```ruby
class Report
  include ActiveModel::Serialization

  attr_accessor :title, :generated_at, :data

  def attributes
    { "title" => nil, "generated_at" => nil, "data" => nil }
  end
end

report = Report.new
report.title = "Q4 Sales"
report.generated_at = Time.current
report.data = { total: 50_000 }

report.serializable_hash
# => {"title"=>"Q4 Sales", "generated_at"=>..., "data"=>{:total=>50000}}

report.serializable_hash(only: [:title])
# => {"title"=>"Q4 Sales"}

report.serializable_hash(except: [:data])
# => {"title"=>"Q4 Sales", "generated_at"=>...}
```

**JSON (`ActiveModel::Serializers::JSON`)** — includes Serialization:

```ruby
class Report
  include ActiveModel::Serializers::JSON

  attr_accessor :title, :data

  def attributes
    { "title" => nil, "data" => nil }
  end

  def attributes=(hash)
    hash.each { |k, v| public_send("#{k}=", v) }
  end
end

report = Report.new(title: "Q4")
report.as_json   # => {"title"=>"Q4", "data"=>nil}
report.to_json   # => '{"title":"Q4","data":null}'

# Deserialize from JSON:
report2 = Report.new
report2.from_json('{"title":"Q4","data":{"total":50000}}')
report2.title  # => "Q4"
report2.data   # => {"total" => 50000}
```

**Critical:** You must define an `attributes` method that returns a hash with string keys. The values are defaults (usually `nil`). This tells the serializer which attributes to include.

### ActiveModel::Naming

Usually included automatically via `Model`/`API`. Key methods on `model_name`:

```ruby
class Admin::UserForm
  include ActiveModel::Model
end

Admin::UserForm.model_name.name              # => "Admin::UserForm"
Admin::UserForm.model_name.singular          # => "admin_user_form"
Admin::UserForm.model_name.plural            # => "admin_user_forms"
Admin::UserForm.model_name.element           # => "user_form"
Admin::UserForm.model_name.human             # => "User form"
Admin::UserForm.model_name.param_key         # => "admin_user_form"
Admin::UserForm.model_name.route_key         # => "admin_user_forms"
Admin::UserForm.model_name.singular_route_key # => "admin_user_form"
Admin::UserForm.model_name.i18n_key          # => :"admin/user_form"
Admin::UserForm.model_name.collection        # => "admin/user_forms"
```

**Override to strip namespace:**

```ruby
class Admin::UserForm
  include ActiveModel::Model

  def self.model_name
    ActiveModel::Name.new(self, nil, "UserForm")
  end
end

Admin::UserForm.model_name.param_key  # => "user_form" (not "admin_user_form")
```

### ActiveModel::Translation

Integrates with Rails I18n. Included automatically via `Model`/`API`.

```yaml
# config/locales/en.yml
en:
  activemodel:
    models:
      contact_form: "Contact Request"
    attributes:
      contact_form:
        name: "Your Name"
        email: "Email Address"
        phone: "Phone Number"
    errors:
      models:
        contact_form:
          attributes:
            name:
              blank: "is required — we need to know who you are"
            email:
              invalid: "doesn't look like a valid email"
```

```ruby
ContactForm.model_name.human                    # => "Contact Request"
ContactForm.human_attribute_name(:name)          # => "Your Name"
ContactForm.human_attribute_name(:phone)         # => "Phone Number"
```

### ActiveModel::SecurePassword

Adds bcrypt-based password handling. Requires the `bcrypt` gem.

```ruby
class Account
  include ActiveModel::Model
  include ActiveModel::SecurePassword

  has_secure_password
  has_secure_password :recovery_password, validations: false

  attr_accessor :password_digest, :recovery_password_digest
end

account = Account.new
account.password = "secret123"
account.password_confirmation = "secret123"
account.valid?  # => true

account.authenticate("secret123")   # => account
account.authenticate("wrong")       # => false

# Named passwords:
account.recovery_password = "backup42"
account.authenticate_recovery_password("backup42")  # => account
```

**Default validations (on the primary password):**
1. Password must be present on creation
2. Password confirmation must match (if `password_confirmation` is set)
3. Max 72 bytes (bcrypt limitation)

Pass `validations: false` to skip these for secondary passwords.

### ActiveModel::Conversion

Provides `to_model`, `to_key`, `to_param`, `to_partial_path`. Included via `Model`/`API`.

```ruby
class Article
  include ActiveModel::Model
  attr_accessor :id, :title

  def persisted?
    id.present?
  end
end

article = Article.new(id: 5, title: "Hello")
article.to_model         # => self
article.to_key           # => [5]
article.to_param         # => "5"
article.to_partial_path  # => "articles/article"

unpersisted = Article.new(title: "Draft")
unpersisted.to_key    # => nil
unpersisted.to_param  # => nil
```

**`to_partial_path`** lets `render @article` find `app/views/articles/_article.html.erb`.

### ActiveModel::AttributeMethods

Low-level module for defining dynamic attribute methods with prefixes/suffixes. Most agents won't need this directly — it's used internally by Active Record.

```ruby
class DataPoint
  include ActiveModel::AttributeMethods

  attribute_method_prefix "clear_"
  attribute_method_suffix "_present?"

  define_attribute_methods :value, :label

  attr_accessor :value, :label

  private

  def clear_attribute(attr)
    public_send("#{attr}=", nil)
  end

  def attribute_present?(attr)
    public_send(attr).present?
  end
end

dp = DataPoint.new
dp.value = 42
dp.value_present?   # => true
dp.clear_value
dp.value            # => nil
```

Also supports `alias_attribute`:

```ruby
alias_attribute :full_name, :name
# Now full_name and name are interchangeable
```

## Advanced Patterns

### Form Object with Nested Attributes

```ruby
class EventRegistrationForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :event_name, :string
  attribute :event_date, :date
  attribute :organizer_name, :string
  attribute :organizer_email, :string
  attribute :max_attendees, :integer, default: 100

  validates :event_name, :event_date, :organizer_name, :organizer_email, presence: true
  validates :organizer_email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :max_attendees, numericality: { greater_than: 0 }
  validate :event_date_in_future

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      organizer = User.find_or_create_by!(email: organizer_email) do |u|
        u.name = organizer_name
      end
      Event.create!(
        name: event_name,
        date: event_date,
        organizer:,
        max_attendees:
      )
    end
    true
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    false
  end

  private

  def event_date_in_future
    return if event_date.blank?
    errors.add(:event_date, "must be in the future") if event_date <= Date.current
  end
end
```

### Decorator/Presenter with Validation

```ruby
class PublishableArticle
  include ActiveModel::Model
  include ActiveModel::Attributes

  attr_reader :article

  attribute :publish_date, :datetime
  attribute :featured, :boolean, default: false
  attribute :seo_title, :string
  attribute :seo_description, :string

  validates :publish_date, presence: true
  validates :seo_title, presence: true, length: { maximum: 60 }
  validates :seo_description, presence: true, length: { maximum: 160 }
  validate :article_ready_to_publish

  def initialize(article, **attrs)
    @article = article
    super(**attrs)
  end

  def publish!
    return false unless valid?

    article.update!(
      published_at: publish_date,
      featured:,
      seo_title:,
      seo_description:,
      status: :published
    )
    true
  end

  private

  def article_ready_to_publish
    errors.add(:base, "Article body is empty") if article.body.blank?
    errors.add(:base, "Article needs a title") if article.title.blank?
  end
end
```

### Wizard/Multi-Step Form

```ruby
class OnboardingForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :step, :integer, default: 1

  # Step 1
  attribute :name, :string
  attribute :email, :string

  # Step 2
  attribute :company_name, :string
  attribute :role, :string

  # Step 3
  attribute :plan, :string
  attribute :billing_email, :string

  validates :name, :email, presence: true, if: -> { step >= 1 }
  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }, if: -> { step >= 1 && email.present? }
  validates :company_name, :role, presence: true, if: -> { step >= 2 }
  validates :plan, presence: true, inclusion: { in: %w[starter growth enterprise] }, if: -> { step >= 3 }
  validates :billing_email, format: { with: URI::MailTo::EMAIL_REGEXP }, if: -> { step >= 3 && billing_email.present? }

  def valid_step?
    valid?
  end

  def next_step
    self.step += 1 if valid_step?
  end

  def final_step?
    step == 3
  end

  def complete!
    self.step = 3
    return false unless valid?

    ActiveRecord::Base.transaction do
      company = Company.create!(name: company_name)
      user = company.users.create!(name:, email:, role:)
      company.subscriptions.create!(plan:, billing_email: billing_email || email)
    end
    true
  end
end
```

### Service Object with Active Model Validation

```ruby
class TransferFunds
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :from_account_id, :integer
  attribute :to_account_id, :integer
  attribute :amount, :decimal
  attribute :memo, :string

  validates :from_account_id, :to_account_id, :amount, presence: true
  validates :amount, numericality: { greater_than: 0 }
  validate :accounts_are_different
  validate :sufficient_balance

  def execute
    return failure unless valid?

    ActiveRecord::Base.transaction do
      from_account.lock!
      to_account.lock!

      from_account.decrement!(:balance, amount)
      to_account.increment!(:balance, amount)

      Transfer.create!(
        from_account:, to_account:,
        amount:, memo:
      )
    end
    success
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    failure
  end

  private

  def from_account = @from_account ||= Account.find_by(id: from_account_id)
  def to_account = @to_account ||= Account.find_by(id: to_account_id)

  def accounts_are_different
    return if from_account_id != to_account_id
    errors.add(:to_account_id, "must be different from source")
  end

  def sufficient_balance
    return unless from_account && amount
    errors.add(:amount, "exceeds available balance") if from_account.balance < amount
  end

  def success = OpenStruct.new(success?: true, errors: [])
  def failure = OpenStruct.new(success?: false, errors: errors.full_messages)
end
```

## Lint Tests

Use `ActiveModel::Lint::Tests` to verify your object is fully API-compliant:

```ruby
require "test_helper"

class ContactFormLintTest < ActiveSupport::TestCase
  include ActiveModel::Lint::Tests

  setup do
    @model = ContactForm.new
  end
end
```

This runs compliance checks for `to_model`, `to_key`, `to_param`, `to_partial_path`, `persisted?`, and `model_name`.

## Edge Cases & Gotchas

### 1. `form_with` Requires `url:`

Active Model objects don't have routing. You must always pass `url:`:

```erb
<%# WRONG — will error or guess wrong route %>
<%= form_with model: @contact_form do |f| %>

<%# CORRECT %>
<%= form_with model: @contact_form, url: contact_forms_path do |f| %>
```

### 2. `persisted?` Defaults to `false`

Active Model objects return `false` from `persisted?` by default. This affects:
- `to_key` returns `nil`
- `to_param` returns `nil`
- `form_with` will use POST (not PATCH)

Override if needed:

```ruby
def persisted?
  id.present?
end
```

### 3. Attributes Requires Explicit `attributes` for Serialization

If you use both `ActiveModel::Attributes` and `ActiveModel::Serialization`, the `attribute` macro defines attributes for casting, but you still need an `attributes` method for serialization:

```ruby
class Thing
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Serializers::JSON

  attribute :name, :string
  attribute :count, :integer

  # This is needed for serializable_hash / as_json / to_json
  def attributes
    { "name" => name, "count" => count }
  end
end
```

### 4. Callbacks Use `extend`, Not `include`

```ruby
# WRONG
class Foo
  include ActiveModel::Callbacks  # NoMethodError on define_model_callbacks
end

# CORRECT
class Foo
  extend ActiveModel::Callbacks
end
```

### 5. Dirty Tracking with Attributes

When using `ActiveModel::Attributes` + `ActiveModel::Dirty` together, you get automatic tracking without manual `_will_change!` calls. But you must include `Dirty` after `Attributes`:

```ruby
class Config
  include ActiveModel::Model
  include ActiveModel::Attributes
  include ActiveModel::Dirty  # After Attributes

  attribute :theme, :string
end
```

### 6. Strong Parameters Integration

`assign_attributes` checks for `permitted?` on the hash. If you pass raw `params`, you'll get `ActiveModel::ForbiddenAttributesError`:

```ruby
# WRONG
form.assign_attributes(params[:contact_form])

# CORRECT
form.assign_attributes(params.require(:contact_form).permit(:name, :email))
```

### 7. `validates_associated` Doesn't Work

`validates_associated` is an Active Record validator. For form objects wrapping other models, validate manually:

```ruby
validate :nested_records_valid

def nested_records_valid
  items.each_with_index do |item, i|
    next if item.valid?
    item.errors.each do |error|
      errors.add(:base, "Item #{i + 1}: #{error.full_message}")
    end
  end
end
```

### 8. Boolean Casting Gotcha

`ActiveModel::Attributes` casts booleans aggressively. `"0"`, `"false"`, `"f"`, `"no"`, `"off"`, and `""` all become `false`. `"1"`, `"true"`, `"t"`, `"yes"`, `"on"` become `true`. Any other string becomes `true`.

```ruby
attribute :active, :boolean
# "banana" => true  (careful!)
# "" => false
# nil => nil
# 0 => false
# 1 => true
```

### 9. Initialization Order Matters

With `ActiveModel::Model`, attributes are assigned in hash order during `new`. If you have dependencies between attributes, handle them in a method, not in the initializer:

```ruby
# FRAGILE — depends on hash key ordering
class Foo
  include ActiveModel::Model
  attr_accessor :start_date, :duration, :end_date

  def initialize(attrs = {})
    super
    self.end_date = start_date + duration.days if start_date && duration
  end
end

# BETTER — compute on demand
class Foo
  include ActiveModel::Model
  attr_accessor :start_date, :duration

  def end_date
    start_date + duration.days if start_date && duration
  end
end
```

### 10. Error Handling Patterns

Active Model errors work the same as Active Record:

```ruby
form = ContactForm.new
form.valid?

form.errors.any?              # => true
form.errors.count             # => 3
form.errors[:name]            # => ["can't be blank"]
form.errors.full_messages     # => ["Name can't be blank", ...]
form.errors.added?(:name, :blank) # => true

# Add errors manually
form.errors.add(:base, "Something went wrong")
form.errors.add(:email, :taken, value: "test@example.com")

# Clear errors
form.errors.clear

# Iterate
form.errors.each do |error|
  puts "#{error.attribute}: #{error.message}"
end

# Group by attribute
form.errors.group_by_attribute
# => { name: [...], email: [...] }
```
