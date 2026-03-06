# Active Record Validations — Reference

Deep-dive patterns, edge cases, and complete examples for Active Record validations in Rails 8.1.

## Table of Contents

- [Built-in Validators Complete Reference](#built-in-validators-complete-reference)
- [Normalizations Deep Dive](#normalizations-deep-dive)
- [Custom Validators Patterns](#custom-validators-patterns)
- [Errors API Complete Reference](#errors-api-complete-reference)
- [Conditional Validations Patterns](#conditional-validations-patterns)
- [Uniqueness Race Conditions](#uniqueness-race-conditions)
- [Numericality Edge Cases](#numericality-edge-cases)
- [I18n and Error Messages](#i18n-and-error-messages)
- [Testing Validations](#testing-validations)
- [DB Constraint Pairing Patterns](#db-constraint-pairing-patterns)
- [Multi-Step Form Validations](#multi-step-form-validations)
- [Performance Considerations](#performance-considerations)

---

## Built-in Validators Complete Reference

### absence

Validates the attribute is blank (`nil`, `""`, or whitespace-only string).

```ruby
validates :supplementary_info, absence: true
validates :end_date, absence: true, unless: :multi_day_event?
```

For associations:
```ruby
belongs_to :order, optional: true
validates :order, absence: true  # validate the association object, not foreign key
```

For booleans (since `false.present?` is false):
```ruby
validates :hidden, exclusion: { in: [true, false] }  # truly absent — no true or false
```

Default error: `"must be blank"`

### acceptance

Virtual attribute — no DB column needed unless you explicitly create one.

```ruby
validates :terms_of_service, acceptance: true
validates :eula, acceptance: { accept: ["TRUE", "accepted", "yes"] }
validates :terms, acceptance: { message: "must be agreed to before continuing" }
```

Only runs when the attribute is not `nil`. Default accepts `['1', true]`.

If a DB column exists, the `accept` option must include `true` or the validation won't trigger.

Default error: `"must be accepted"`

### comparison

Compare any two `Comparable` values. Options accept values, procs, or symbols referencing methods.

```ruby
validates :end_date, comparison: { greater_than: :start_date }
validates :age, comparison: { greater_than_or_equal_to: 18 }
validates :weight, comparison: { less_than: 500 }
validates :rating, comparison: { in: 1..5 }  # not documented everywhere but works

# With proc:
validates :renewal_date, comparison: { greater_than: -> { Date.current } }

# With method symbol:
validates :discount_price, comparison: { less_than: :regular_price }
```

All comparison options:
- `:greater_than`
- `:greater_than_or_equal_to`
- `:equal_to`
- `:less_than`
- `:less_than_or_equal_to`
- `:other_than`

Default error: `"failed comparison"` (plus specific messages per option)

### confirmation

Creates a virtual `_confirmation` attribute.

```ruby
class User < ApplicationRecord
  validates :password, confirmation: true
  validates :password_confirmation, presence: true, if: :password_changed?

  # Case-insensitive email confirmation:
  validates :email, confirmation: { case_sensitive: false }
  validates :email_confirmation, presence: true, if: :email_changed?
end
```

**Key detail:** Only validates when the `_confirmation` attribute is not `nil`. Always add `presence: true` on the confirmation field, scoped to when the original field changes.

Default error: `"doesn't match confirmation"`

### exclusion

```ruby
validates :subdomain, exclusion: { in: %w[www admin api mail ftp] }
validates :format, exclusion: { in: %w[exe bat cmd], message: "%{value} is not allowed" }

# Dynamic:
validates :username, exclusion: { in: ->(user) { user.reserved_names } }
```

Uses `include?` for arrays, `cover?` for ranges.

Default error: `"is reserved"`

### format

```ruby
# ALWAYS use \A and \z (string boundaries), NEVER ^ and $ (line boundaries)
validates :username, format: { with: /\A[a-z0-9_]{3,30}\z/i }
validates :legacy_code, format: { with: /\A[A-Z]{3}-\d{4}\z/, message: "must be format XXX-0000" }

# Reject pattern:
validates :content, format: { without: /\b(spam|viagra)\b/i, message: "contains prohibited words" }

# With proc:
validates :code, format: { with: ->(record) { record.code_pattern } }
```

**Security warning:** Using `^` and `$` allows multiline bypass:
```ruby
# VULNERABLE: "valid\n<script>alert('xss')</script>" passes
validates :name, format: { with: /^[a-z]+$/ }

# SAFE:
validates :name, format: { with: /\A[a-z]+\z/ }
```

If you must use `^` or `$`, pass `multiline: true` to acknowledge the risk.

Default error: `"is invalid"`

### inclusion

```ruby
validates :size, inclusion: { in: %w[small medium large] }
validates :priority, inclusion: { in: 1..5 }
validates :role, inclusion: { in: %w[admin editor viewer], message: "%{value} is not a valid role" }

# Dynamic (proc receives the record):
validates :category, inclusion: { in: ->(record) { record.workspace.allowed_categories } }
```

`:within` is an alias for `:in`.

Default error: `"is not included in the list"`

### length

```ruby
validates :name, length: { minimum: 2 }
validates :bio, length: { maximum: 1000 }
validates :password, length: { in: 8..128 }       # range
validates :ssn, length: { is: 9 }                  # exact
validates :title, length: { minimum: 5, maximum: 100 }  # min + max combo

# Custom messages per constraint:
validates :essay, length: {
  minimum: 100,
  too_short: "must have at least %{count} characters",
  maximum: 5000,
  too_long: "must have at most %{count} characters"
}
```

**Gotcha:** When `:minimum` is 1, use `presence: true` instead — better error message.

Default errors: `"is too short (minimum is %{count} characters)"`, `"is too long (maximum is %{count} characters)"`, `"is the wrong length (should be %{count} characters)"`

### numericality

```ruby
validates :price, numericality: true                                    # any number
validates :quantity, numericality: { only_integer: true }               # integers only
validates :score, numericality: { in: 0..100 }                         # range
validates :temperature, numericality: { greater_than: -273.15 }        # absolute zero
validates :discount, numericality: { greater_than_or_equal_to: 0, less_than_or_equal_to: 100 }
validates :age, numericality: { only_integer: true, odd: true }        # odd integers
```

All numericality options:
- `:only_integer` — must match `/\A[+-]?\d+\z/`
- `:only_numeric` — must be `Numeric` instance (or parseable string)
- `:greater_than`, `:greater_than_or_equal_to`
- `:equal_to`, `:other_than`
- `:less_than`, `:less_than_or_equal_to`
- `:in` — must be in range
- `:odd`, `:even`

**Edge cases:**
- Rejects `nil` by default — add `allow_nil: true` for optional fields
- Empty strings on `Integer`/`Float` columns get converted to `nil` by Active Record
- Float precision: values are converted through `Float` then `BigDecimal` (max 15 digits precision)
- For money: use `decimal` column with `precision: 10, scale: 2` and validate, or store as integer cents

Default errors: `"is not a number"`, `"must be an integer"`, plus constraint-specific messages

### presence

```ruby
validates :name, presence: true
validates :title, :body, presence: true  # multiple attributes

# For associations — validate the object, not the FK:
belongs_to :author  # already validates presence by default in Rails 5+
has_one :profile
validates :profile, presence: true  # checks object is present, not marked_for_destruction?
```

**Boolean trap:** `false.blank?` returns `true`, so `presence: true` rejects `false`.
```ruby
# WRONG:
validates :active, presence: true  # rejects false!

# RIGHT:
validates :active, inclusion: { in: [true, false] }
```

Default error: `"can't be blank"`

### uniqueness

```ruby
validates :email, uniqueness: true
validates :slug, uniqueness: { scope: :blog_id }           # unique per blog
validates :name, uniqueness: { scope: [:account_id, :year] } # composite scope
validates :code, uniqueness: { case_sensitive: false }
validates :email, uniqueness: { conditions: -> { where(deleted_at: nil) } }  # soft-delete aware
```

**Always back with a DB index:**
```ruby
# Single column:
add_index :users, :email, unique: true

# Scoped:
add_index :posts, [:blog_id, :slug], unique: true

# Partial (PostgreSQL, for soft-delete):
add_index :users, :email, unique: true, where: "deleted_at IS NULL"
```

Default error: `"has already been taken"`

### validates_associated

```ruby
class Library < ApplicationRecord
  has_many :books
  validates_associated :books  # calls valid? on each associated book
end
```

**Rules:**
- Only use on the parent side, never both sides (infinite loop)
- Only works with ActiveRecord objects
- Errors stay on the child objects, they don't bubble up
- Consider if you actually need this — `autosave: true` on the association already validates before saving

Default error: `"is invalid"`

### validates_each

Block-based validation for quick custom checks without a full method:

```ruby
validates_each :name, :surname do |record, attr, value|
  record.errors.add(attr, "must start with uppercase") if /\A[[:lower:]]/.match?(value)
end
```

### validates_with

```ruby
# Simple:
validates_with AddressValidator

# With options:
validates_with AddressValidator, fields: [:street, :city, :zip]

# Multiple validators:
validates_with FirstValidator, SecondValidator

# Conditional:
validates_with ShippingValidator, if: :requires_shipping?
```

**Important:** The validator instance is shared across all validations (initialized once per app lifecycle). Don't use instance variables for per-validation state.

---

## Normalizations Deep Dive

Rails 7.1+ `normalizes` declaration. Runs before validation AND on attribute assignment.

### Basic patterns

```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
  normalizes :phone, with: ->(phone) { phone.gsub(/[^\d+]/, "") }  # keep digits and +
  normalizes :name, with: ->(name) { name.squish }                  # collapse all whitespace
  normalizes :website, with: ->(url) { url.strip.downcase.delete_suffix("/") }
end
```

### When normalizations run

```ruby
user = User.new
user.email = "  FOO@BAR.COM  "
user.email  # => "foo@bar.com"  — normalized on assignment

User.find_by(email: "  FOO@BAR.COM  ")
# SQL: SELECT * FROM users WHERE email = 'foo@bar.com'
# Normalization applied to the query value too!

User.where(email: "  FOO@BAR.COM  ").first
# Also normalized in where clauses
```

### Handling nil

By default, normalization lambdas receive `nil` when the attribute is nil. Handle it:

```ruby
# Option 1: Guard in lambda
normalizes :email, with: ->(email) { email&.strip&.downcase }

# Option 2: apply_to_nil: false (skip normalization for nil entirely)
normalizes :nickname, with: ->(n) { n.strip }, apply_to_nil: false
```

### Normalization + uniqueness interaction

```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(e) { e.strip.downcase }
  validates :email, uniqueness: true
end

# Now "FOO@BAR.COM" and "foo@bar.com" are the same — no duplicates from case.
# The uniqueness query also normalizes, so the SELECT matches correctly.
```

Without normalization, you'd need `uniqueness: { case_sensitive: false }` which generates a slower `LOWER()` query.

---

## Custom Validators Patterns

### EachValidator — the workhorse

File: `app/validators/phone_number_validator.rb`

```ruby
class PhoneNumberValidator < ActiveModel::EachValidator
  PHONE_REGEX = /\A\+?[1-9]\d{6,14}\z/

  def validate_each(record, attribute, value)
    return if value.blank? && options[:allow_blank]
    unless PHONE_REGEX.match?(value)
      record.errors.add(attribute, options[:message] || :invalid_phone)
    end
  end
end

# Usage:
validates :phone, phone_number: true
validates :fax, phone_number: { allow_blank: true, message: "doesn't look like a phone number" }
```

### EachValidator with options

```ruby
class FileSizeValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return unless value.attached?
    max = options[:max] || 10.megabytes
    if value.blob.byte_size > max
      record.errors.add(attribute, options[:message] || "is too large (max #{max / 1.megabyte}MB)")
    end
  end
end

# Usage:
validates :avatar, file_size: { max: 5.megabytes }
```

### Validator class (cross-field)

```ruby
class DateRangeValidator < ActiveModel::Validator
  def validate(record)
    start_field = options[:start] || :start_date
    end_field = options[:end] || :end_date

    start_val = record.send(start_field)
    end_val = record.send(end_field)

    return if start_val.blank? || end_val.blank?

    if end_val <= start_val
      record.errors.add(end_field, "must be after #{start_field.to_s.humanize.downcase}")
    end

    if options[:max_span] && (end_val - start_val) > options[:max_span]
      record.errors.add(end_field, "range cannot exceed #{options[:max_span].inspect}")
    end
  end
end

# Usage:
validates_with DateRangeValidator, start: :check_in, end: :check_out, max_span: 30.days
```

### PORO validator (needs instance state)

When your validator needs per-validation instance variables:

```ruby
class Invoice < ApplicationRecord
  validate do |invoice|
    TaxValidator.new(invoice).validate
  end
end

class TaxValidator
  def initialize(invoice)
    @invoice = invoice
    @tax_rules = TaxRule.for_region(invoice.region)
  end

  def validate
    return if @invoice.tax_exempt?
    validate_tax_rate
    validate_tax_amount
  end

  private

  def validate_tax_rate
    unless @tax_rules.valid_rate?(@invoice.tax_rate)
      @invoice.errors.add(:tax_rate, "is not valid for #{@invoice.region}")
    end
  end

  def validate_tax_amount
    expected = @invoice.subtotal * @invoice.tax_rate
    unless (@invoice.tax_amount - expected).abs < 0.01
      @invoice.errors.add(:tax_amount, "doesn't match expected tax calculation")
    end
  end
end
```

---

## Errors API Complete Reference

### Reading errors

```ruby
record.valid?                              # Run validations, return bool
record.invalid?                            # Inverse of valid?

record.errors                              # ActiveModel::Errors instance
record.errors.any?                         # Any errors?
record.errors.empty?                       # No errors? (alias: .none?)
record.errors.count                        # Total error count (alias: .size)
record.errors.full_messages                # ["Name can't be blank", ...]
record.errors.full_messages_for(:name)     # ["Name can't be blank", "Name is too short..."]

record.errors[:name]                       # ["can't be blank", "is too short..."]
record.errors[:name].any?                  # Errors on this attribute?

record.errors.messages                     # { name: ["can't be blank"], email: ["is invalid"] }
record.errors.details                      # { name: [{error: :blank}], email: [{error: :invalid}] }
record.errors.to_hash                      # Same as messages but frozen
```

### Filtering errors

```ruby
# where — returns array of ActiveModel::Error objects
record.errors.where(:name)                         # all errors on :name
record.errors.where(:name, :too_short)             # specific type
record.errors.where(:name, :too_short, minimum: 3) # with options match

# Error object properties
error = record.errors.where(:name).first
error.attribute   # :name
error.type        # :too_short
error.options     # { count: 3 }
error.message     # "is too short (minimum is 3 characters)"
error.full_message # "Name is too short (minimum is 3 characters)"
error.details     # { error: :too_short, count: 3 }
```

### Adding errors

```ruby
errors.add(:name, :blank)                                # Standard type
errors.add(:name, :too_short, count: 3)                  # With options for i18n
errors.add(:name, "must be unique within the team")      # Custom string message
errors.add(:base, "This record is invalid")              # Record-level error
errors.add(:name, :custom_type, message: "is not ok")    # Custom type + message
```

### Checking errors

```ruby
errors.added?(:name, :blank)           # Was this exact error added?
errors.added?(:name, :too_short, count: 3)  # With option match
errors.of_kind?(:name, :blank)         # Like added? but ignores options
errors.include?(:name)                 # Any errors on this attribute?

# Aliases
errors.has_key?(:name)                 # same as include?
errors.key?(:name)                     # same as include?
```

### Clearing

```ruby
errors.clear          # Remove all errors (they come back on next valid? call)
errors.delete(:name)  # Remove errors for one attribute
```

---

## Conditional Validations Patterns

### Symbol (method name) — preferred for named logic

```ruby
class Order < ApplicationRecord
  validates :shipping_address, presence: true, if: :physical_product?
  validates :download_url, presence: true, unless: :physical_product?

  private

  def physical_product?
    product_type == "physical"
  end
end
```

### Proc/Lambda — for simple one-liners

```ruby
validates :company, presence: true, if: -> { account_type == "business" }
validates :parent_email, presence: true, if: -> { age.present? && age < 18 }
```

**Rule of thumb:** If the lambda has `&&`, `||`, or is longer than ~40 chars, extract to a method.

### Array of conditions

```ruby
# ALL conditions must be true for validation to run
validates :bio, presence: true,
  if: [:profile_complete?, :public_profile?]

# Mix symbols and lambdas
validates :vat_number, presence: true,
  if: [:business_account?, -> { country.in?(%w[DE FR IT]) }]
```

### with_options grouping

```ruby
class User < ApplicationRecord
  with_options if: :admin? do
    validates :permissions, presence: true
    validates :security_clearance, numericality: { greater_than: 0 }
    validates :admin_notes, length: { maximum: 5000 }
  end

  with_options unless: :admin? do
    validates :terms_accepted, acceptance: true
  end
end
```

### :on context for multi-step flows

```ruby
class Registration < ApplicationRecord
  # Step 1: Basic info
  validates :email, presence: true  # always (no :on)
  validates :password, presence: true, on: :create

  # Step 2: Profile
  validates :name, presence: true, on: :profile_step
  validates :bio, length: { maximum: 500 }, on: :profile_step

  # Step 3: Payment
  validates :card_number, presence: true, on: :payment_step
  validates :billing_address, presence: true, on: :payment_step
end

# Controller:
def update_profile
  if @registration.save(context: :profile_step)
    redirect_to payment_step_path
  else
    render :profile_step
  end
end
```

---

## Uniqueness Race Conditions

### The problem

```
Request A:  SELECT COUNT(*) WHERE email = 'foo@bar.com'  →  0
Request B:  SELECT COUNT(*) WHERE email = 'foo@bar.com'  →  0
Request A:  INSERT INTO users (email) VALUES ('foo@bar.com')  →  ✓
Request B:  INSERT INTO users (email) VALUES ('foo@bar.com')  →  ✓  DUPLICATE!
```

### The solution: DB unique index + rescue

```ruby
# Migration:
add_index :users, :email, unique: true

# Model:
validates :email, uniqueness: true  # still useful for nice error messages

# Controller — handle the race condition:
def create
  @user = User.new(user_params)
  @user.save!
  redirect_to @user
rescue ActiveRecord::RecordNotUnique
  @user.errors.add(:email, :taken)
  render :new, status: :unprocessable_entity
end
```

### Scoped uniqueness

```ruby
# Model:
validates :slug, uniqueness: { scope: :blog_id }

# Migration — composite unique index:
add_index :posts, [:blog_id, :slug], unique: true
```

### Soft-delete aware uniqueness

```ruby
# Model:
validates :email, uniqueness: { conditions: -> { where(deleted_at: nil) } }

# Migration (PostgreSQL partial index):
add_index :users, :email, unique: true, where: "deleted_at IS NULL"
```

### Multi-tenant uniqueness

```ruby
# With acts_as_tenant or manual scoping:
validates :name, uniqueness: { scope: :tenant_id }

# DB index must match:
add_index :projects, [:tenant_id, :name], unique: true
```

---

## Numericality Edge Cases

### nil handling

```ruby
# Rejects nil by default:
validates :age, numericality: true
User.new(age: nil).valid?  # => false

# Allow nil:
validates :age, numericality: { allow_nil: true }
```

### Empty string conversion

On `integer` and `float` columns, Active Record converts `""` to `nil` before validation.

```ruby
# Column: t.integer :age
user.age = ""
user.age  # => nil (Active Record type cast)
# Then numericality rejects nil unless allow_nil: true
```

### Float precision

```ruby
validates :price, numericality: { greater_than: 0 }

# Be careful with float comparisons:
(0.1 + 0.2) == 0.3  # => false in Ruby!

# For money, use decimal columns:
# t.decimal :price, precision: 10, scale: 2
# Or store as integer cents and validate that
```

### Combining constraints

```ruby
validates :score, numericality: {
  only_integer: true,
  greater_than_or_equal_to: 0,
  less_than_or_equal_to: 100
}

# Equivalent with :in (Rails 7+):
validates :score, numericality: { only_integer: true, in: 0..100 }
```

---

## I18n and Error Messages

### Error message lookup order

For `validates :name, presence: true` on `User`:

1. `activerecord.errors.models.user.attributes.name.blank`
2. `activerecord.errors.models.user.blank`
3. `activerecord.errors.messages.blank`
4. `errors.attributes.name.blank`
5. `errors.messages.blank`

### Locale file structure

```yaml
# config/locales/en.yml
en:
  activerecord:
    errors:
      models:
        user:
          attributes:
            email:
              blank: "is required"
              taken: "is already registered"
              invalid: "doesn't look right"
            name:
              too_short: "needs at least %{count} characters"
          # Model-level (any attribute):
          invalid: "has problems"
      # Global (any model):
      messages:
        blank: "cannot be empty"
  errors:
    # Fallback for ActiveModel (non-AR):
    messages:
      blank: "is required"
```

### Message interpolation variables

Available in all messages:
- `%{value}` — the attribute value
- `%{attribute}` — humanized attribute name
- `%{model}` — humanized model name

Additional for specific validators:
- `%{count}` — for length, numericality constraints

### Custom error types with i18n

```ruby
# Model:
errors.add(:discount, :too_high, count: 50)

# Locale:
# activerecord.errors.models.order.attributes.discount.too_high: "cannot exceed %{count}%"
```

### Proc messages

```ruby
validates :username, uniqueness: {
  message: ->(object, data) {
    "#{data[:value]} is taken. Try #{data[:value]}#{object.id || rand(100)}"
  }
}
```

---

## Testing Validations

### Pattern: In-memory validation tests

```ruby
class UserTest < ActiveSupport::TestCase
  setup do
    @user = users(:valid_user)  # fixture
  end

  test "valid fixture" do
    assert @user.valid?
  end

  test "requires email" do
    @user.email = nil
    refute @user.valid?
    assert_includes @user.errors[:email], "can't be blank"
  end

  test "requires unique email" do
    other = users(:other_user)
    @user.email = other.email
    refute @user.valid?
    assert_includes @user.errors[:email], "has already been taken"
  end

  test "rejects invalid email format" do
    @user.email = "not-an-email"
    refute @user.valid?
    assert_includes @user.errors[:email], "is not a valid email"
  end

  test "validates numericality of age" do
    @user.age = "abc"
    refute @user.valid?
    assert_includes @user.errors[:age], "is not a number"
  end

  test "rejects negative price" do
    product = Product.new(price: -1)
    refute product.valid?
    assert_includes product.errors[:price], "must be greater than 0"
  end
end
```

### Pattern: Test conditional validations

```ruby
test "requires card number only when paying by card" do
  order = Order.new(payment_type: "cash")
  order.valid?
  assert_empty order.errors[:card_number]

  order.payment_type = "card"
  order.valid?
  assert_includes order.errors[:card_number], "can't be blank"
end
```

### Pattern: Test custom validators

```ruby
test "rejects end date before start date" do
  event = Event.new(start_date: Date.tomorrow, end_date: Date.today)
  refute event.valid?
  assert_includes event.errors[:end_date], "must be after start date"
end
```

### Pattern: Test DB constraints survive

```ruby
test "DB constraint prevents duplicate email" do
  User.create!(email: "dup@test.com", name: "First")
  assert_raises ActiveRecord::RecordNotUnique do
    User.connection.execute(
      "INSERT INTO users (email, name, created_at, updated_at) VALUES ('dup@test.com', 'Second', NOW(), NOW())"
    )
  end
end
```

### Pattern: Test normalizations

```ruby
test "normalizes email on assignment" do
  user = User.new(email: "  FOO@BAR.COM  ")
  assert_equal "foo@bar.com", user.email
end

test "normalizes email in queries" do
  user = users(:valid_user)  # email: "alice@example.com"
  assert_equal user, User.find_by(email: "  ALICE@EXAMPLE.COM  ")
end
```

---

## DB Constraint Pairing Patterns

### Complete model + migration examples

```ruby
# Model:
class Product < ApplicationRecord
  normalizes :sku, with: ->(sku) { sku.strip.upcase }

  validates :name, presence: true, length: { maximum: 255 }
  validates :sku, presence: true, uniqueness: true, format: { with: /\A[A-Z0-9\-]+\z/ }
  validates :price_cents, presence: true, numericality: { only_integer: true, greater_than: 0 }
  validates :category, inclusion: { in: %w[electronics clothing food] }
  validates :weight_kg, numericality: { greater_than: 0 }, allow_nil: true
end

# Migration:
class CreateProducts < ActiveRecord::Migration[8.1]
  def change
    create_table :products do |t|
      t.string :name, null: false, limit: 255
      t.string :sku, null: false
      t.integer :price_cents, null: false
      t.string :category, null: false
      t.decimal :weight_kg, precision: 8, scale: 2
      t.timestamps
    end

    add_index :products, :sku, unique: true
    add_check_constraint :products, "price_cents > 0", name: "products_price_positive"
    add_check_constraint :products, "category IN ('electronics', 'clothing', 'food')", name: "products_category_valid"
  end
end
```

### PostgreSQL CHECK constraints

```ruby
# In migration:
add_check_constraint :orders, "total_cents >= 0", name: "orders_total_non_negative"
add_check_constraint :events, "end_date > start_date", name: "events_date_range_valid"
add_check_constraint :users, "age >= 0 AND age <= 150", name: "users_age_reasonable"

# Remove:
remove_check_constraint :orders, name: "orders_total_non_negative"
```

---

## Multi-Step Form Validations

### Custom context approach

```ruby
class Application < ApplicationRecord
  # Always validated:
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }

  # Step 1: Personal info
  validates :first_name, presence: true, on: :personal_info
  validates :last_name, presence: true, on: :personal_info
  validates :date_of_birth, presence: true, on: :personal_info

  # Step 2: Address
  validates :street, presence: true, on: :address
  validates :city, presence: true, on: :address
  validates :zip_code, presence: true, on: :address

  # Step 3: Documents
  validates :id_document, presence: true, on: :documents
  validates :proof_of_address, presence: true, on: :documents
end
```

```ruby
# Controller:
class ApplicationsController < ApplicationController
  def update_step
    context = params[:step].to_sym
    if @application.save(context: context)
      redirect_to next_step_path
    else
      render current_step_template, status: :unprocessable_entity
    end
  end
end
```

**Note:** When triggered with a custom context, validations with `:on` matching that context run, PLUS all validations without any `:on` option. So the `email` validation above runs on every step.

---

## Performance Considerations

### Uniqueness queries

Every `uniqueness` validation fires a `SELECT` query. On high-traffic models:

```ruby
# This generates N+1 queries if validating many records:
records.each { |r| r.valid? }

# Consider batch validation or DB-level only for bulk imports
```

### validates_associated cost

```ruby
# If library has 1000 books, this calls valid? on ALL of them:
validates_associated :books

# Consider limiting or skipping for large collections:
validates_associated :books, if: -> { books.size < 100 }
```

### Conditional validation performance

```ruby
# If the :if method is expensive, it runs on every validation:
validates :extra_field, presence: true, if: :expensive_api_check?

# Cache the result:
validates :extra_field, presence: true, if: :needs_extra_field?

def needs_extra_field?
  @_needs_extra_field ||= expensive_api_check?
end
```

### Normalization on every assignment

Normalizations run on every attribute write. Keep them cheap:

```ruby
# Good — fast string operations:
normalizes :email, with: ->(e) { e.strip.downcase }

# Bad — expensive operation on every assignment:
normalizes :address, with: ->(a) { GeocodingService.normalize(a) }
# Move this to a before_validation callback instead
```

---

## Miscellaneous Patterns

### Validating array/JSON columns (PostgreSQL)

```ruby
class Product < ApplicationRecord
  validate :valid_tags

  private

  def valid_tags
    return if tags.blank?
    unless tags.is_a?(Array) && tags.all? { |t| t.is_a?(String) && t.length <= 50 }
      errors.add(:tags, "must be an array of strings (max 50 chars each)")
    end
    if tags.length > 10
      errors.add(:tags, "cannot have more than 10 tags")
    end
  end
end
```

### Validating nested attributes

```ruby
class Order < ApplicationRecord
  has_many :line_items
  accepts_nested_attributes_for :line_items, reject_if: :all_blank, allow_destroy: true
  validates_associated :line_items
  validate :at_least_one_line_item

  private

  def at_least_one_line_item
    if line_items.reject(&:marked_for_destruction?).empty?
      errors.add(:base, "must have at least one line item")
    end
  end
end
```

### Validation outside Active Record

```ruby
class ContactForm
  include ActiveModel::Validations

  attr_accessor :name, :email, :message

  validates :name, presence: true
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :message, presence: true, length: { minimum: 10 }
end

form = ContactForm.new
form.name = "Alice"
form.valid?  # => false (email and message missing)
form.errors.full_messages  # ["Email can't be blank", ...]
```

### Listing validators on a model

```ruby
User.validators                    # All validators
User.validators_on(:email)        # Validators for :email
User.validators_on(:email).map(&:class)  # [PresenceValidator, UniquenessValidator, ...]
```
