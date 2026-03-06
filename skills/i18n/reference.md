# I18n Reference — Detailed Patterns, Examples, and Edge Cases

## Table of Contents

1. [YAML Locale File Patterns](#yaml-locale-file-patterns)
2. [Translation Lookup Deep Dive](#translation-lookup-deep-dive)
3. [Interpolation Patterns](#interpolation-patterns)
4. [Pluralization Rules by Language](#pluralization-rules-by-language)
5. [Date, Time, and Number Localization](#date-time-and-number-localization)
6. [Active Record Translations](#active-record-translations)
7. [Error Message Resolution Chain](#error-message-resolution-chain)
8. [Locale Switching Strategies](#locale-switching-strategies)
9. [Fallback Configuration](#fallback-configuration)
10. [Testing I18n](#testing-i18n)
11. [Localized Views](#localized-views)
12. [Custom Backends](#custom-backends)
13. [Common Gems](#common-gems)
14. [Edge Cases and Gotchas](#edge-cases-and-gotchas)

---

## YAML Locale File Patterns

### Basic Structure

```yaml
# ALWAYS start with the locale key
en:
  key: "value"
  nested:
    key: "nested value"
```

### Quoting Rules — CRITICAL

YAML treats certain values as booleans/nulls. You MUST quote these when used as keys:

```yaml
en:
  # BROKEN — YAML interprets these as booleans/null
  statuses:
    true: "Active"       # ❌ parsed as boolean true
    false: "Inactive"    # ❌ parsed as boolean false
    yes: "Confirmed"     # ❌ parsed as boolean true
    no: "Denied"         # ❌ parsed as boolean false
    on: "Enabled"        # ❌ parsed as boolean true
    off: "Disabled"      # ❌ parsed as boolean false
    null: "None"         # ❌ parsed as null

  # CORRECT — quote the keys
  statuses:
    'true': "Active"
    'false': "Inactive"
    'yes': "Confirmed"
    'no': "Denied"
    'on': "Enabled"
    'off': "Disabled"
    'null': "None"
```

### Multi-line Strings

```yaml
en:
  # Literal block (preserves newlines)
  terms: |
    By using this service, you agree to our terms.
    Please read carefully before proceeding.

  # Folded block (joins lines with spaces)
  description: >
    This is a long description that will be
    joined into a single line with spaces.

  # Quoted with explicit newlines
  message: "Line one\nLine two"
```

### ERB in YAML (Avoid Unless Necessary)

If you must use dynamic values, name the file `.yml.erb`:

```yaml
# config/locales/en.yml — NOT .yml.erb, this won't work
en:
  app_name: "MyApp"

# Only use .yml.erb if truly needed (rare):
en:
  build: "<%= ENV['BUILD_VERSION'] %>"
```

### File Organization for Large Apps

```
config/locales/
├── en.yml                    # Simple/shared keys
├── es.yml
├── models/
│   ├── en.yml                # activerecord.models, .attributes, .errors
│   └── es.yml
├── views/
│   ├── books/
│   │   ├── en.yml            # books.index.*, books.show.*, etc.
│   │   └── es.yml
│   └── users/
│       ├── en.yml
│       └── es.yml
├── mailers/
│   ├── en.yml
│   └── es.yml
└── defaults/
    ├── en.yml                # date.formats, time.formats, number.*
    └── es.yml
```

**Loading nested files:**

```ruby
# config/application.rb
config.i18n.load_path += Dir[Rails.root.join("config", "locales", "**", "*.{rb,yml}")]
```

---

## Translation Lookup Deep Dive

### Lookup Methods Comparison

```ruby
# All equivalent:
I18n.t("activerecord.errors.messages.blank")
I18n.t(:blank, scope: [:activerecord, :errors, :messages])
I18n.t(:blank, scope: "activerecord.errors.messages")
I18n.translate("activerecord.errors.messages.blank")
```

### Bulk Lookup

```ruby
# Multiple keys at once
I18n.t([:odd, :even], scope: "errors.messages")
# => ["must be odd", "must be even"]

# Namespace lookup (returns hash)
I18n.t("errors.messages")
# => { blank: "can't be blank", invalid: "is invalid", ... }
```

### Deep Interpolation

```yaml
en:
  welcome:
    title: "Welcome!"
    content: "Welcome to %{app_name}, %{user}!"
```

```ruby
# Without deep_interpolation — nested values NOT interpolated
I18n.t("welcome", app_name: "MyApp", user: "Ryan")
# => { title: "Welcome!", content: "Welcome to %{app_name}, %{user}!" }

# With deep_interpolation — all values interpolated
I18n.t("welcome", deep_interpolation: true, app_name: "MyApp", user: "Ryan")
# => { title: "Welcome!", content: "Welcome to MyApp, Ryan!" }
```

### Lazy Lookup Scoping Rules

**Views** — scope is `controller_name.action_name`:

```
app/views/books/index.html.erb    → t(".key") resolves to books.index.key
app/views/books/_sidebar.html.erb → t(".key") resolves to books.sidebar.key
app/views/shared/_nav.html.erb    → t(".key") resolves to shared.nav.key
```

**Controllers** — scope is `controller_name.action_name`:

```ruby
# BooksController#create
t(".success")  # → books.create.success

# Admin::BooksController#create
t(".success")  # → admin/books.create.success
# In YAML:
# en:
#   admin/books:
#     create:
#       success: "Created!"
# OR (nested):
# en:
#   admin:
#     books:
#       create:
#         success: "Created!"
```

**Partials** — scope includes the partial name (without underscore):

```
app/views/books/_form.html.erb → t(".submit") resolves to books.form.submit
```

### Default Values

```ruby
# String default
t("missing", default: "Not found")

# Symbol default (tries another key)
t("missing", default: :fallback_key)

# Chain of defaults
t("missing", default: [:try_this, :then_this, "Final string"])

# Proc default (lazy evaluation)
t("missing", default: -> { expensive_computation })
```

### Handling Missing Translations

```ruby
# Returns nil instead of raising/showing "translation missing"
I18n.t("maybe.exists", default: nil)

# Check if translation exists
I18n.exists?("some.key")
I18n.exists?("some.key", :es)
```

---

## Interpolation Patterns

### Basic Interpolation

```yaml
en:
  greeting: "Hello, %{name}!"
  welcome: "Welcome back, %{name}. You have %{count} messages."
```

```ruby
t("greeting", name: "Ryan")
# => "Hello, Ryan!"

t("welcome", name: "Ryan", count: 5)
# => "Welcome back, Ryan. You have 5 messages."
```

### Reserved Variables

These CANNOT be used as interpolation variable names:
- `scope` → raises `I18n::ReservedInterpolationKey`
- `default` → raises `I18n::ReservedInterpolationKey`

### Missing Interpolation Arguments

```ruby
# If a required variable is not passed:
t("greeting")  # greeting expects %{name}
# => raises I18n::MissingInterpolationArgument

# To handle gracefully:
t("greeting", name: nil)  # => "Hello, !"
```

### HTML-Safe Interpolation

```yaml
en:
  # _html suffix = HTML safe
  alert_html: "<strong>Warning:</strong> %{message}"
```

```erb
<%# message is auto-escaped (XSS safe), but the translation markup is preserved %>
<%= t("alert_html", message: user_input) %>
```

The translation itself is marked `html_safe`, but interpolated values are escaped. This is secure by default.

---

## Pluralization Rules by Language

### English (default)

```yaml
en:
  items:
    zero: "No items"     # optional
    one: "1 item"        # count == 1
    other: "%{count} items"  # everything else
```

### Russian (requires rails-i18n gem)

```yaml
ru:
  items:
    one: "%{count} элемент"      # 1, 21, 31...
    few: "%{count} элемента"     # 2-4, 22-24...
    many: "%{count} элементов"   # 5-20, 25-30...
    other: "%{count} элемента"   # 1.5, 2.5...
```

### Arabic

```yaml
ar:
  items:
    zero: "لا عناصر"
    one: "عنصر واحد"
    two: "عنصران"
    few: "%{count} عناصر"
    many: "%{count} عنصراً"
    other: "%{count} عنصر"
```

### Japanese / Chinese / Korean (no plural forms)

```yaml
ja:
  items:
    other: "%{count}個のアイテム"
```

### Enabling Locale-Specific Pluralization

```ruby
# Gemfile
gem "rails-i18n"  # Includes pluralization rules for 100+ locales

# Or manually:
# config/initializers/pluralization.rb
I18n::Backend::Simple.include(I18n::Backend::Pluralization)
```

---

## Date, Time, and Number Localization

### Full Date/Time Locale File

```yaml
en:
  date:
    abbr_day_names:
      - Sun
      - Mon
      - Tue
      - Wed
      - Thu
      - Fri
      - Sat
    abbr_month_names:
      -            # index 0 is nil
      - Jan
      - Feb
      - Mar
      - Apr
      - May
      - Jun
      - Jul
      - Aug
      - Sep
      - Oct
      - Nov
      - Dec
    day_names:
      - Sunday
      - Monday
      - Tuesday
      - Wednesday
      - Thursday
      - Friday
      - Saturday
    formats:
      default: "%Y-%m-%d"
      long: "%B %d, %Y"
      short: "%b %d"
    month_names:
      -            # index 0 is nil
      - January
      - February
      - March
      - April
      - May
      - June
      - July
      - August
      - September
      - October
      - November
      - December
    order:
      - :year
      - :month
      - :day

  time:
    am: "am"
    pm: "pm"
    formats:
      default: "%a, %d %b %Y %H:%M:%S %z"
      long: "%B %d, %Y %H:%M"
      short: "%d %b %H:%M"
      time_only: "%H:%M"

  number:
    format:
      separator: "."
      delimiter: ","
      precision: 3
      significant: false
      strip_insignificant_zeros: false
    currency:
      format:
        format: "%u%n"
        unit: "$"
        separator: "."
        delimiter: ","
        precision: 2
    percentage:
      format:
        delimiter: ""
        format: "%n%"
    precision:
      format:
        delimiter: ""
    human:
      format:
        delimiter: ""
        precision: 3
      storage_units:
        format: "%n %u"
        units:
          byte:
            one: "Byte"
            other: "Bytes"
          kb: "KB"
          mb: "MB"
          gb: "GB"
          tb: "TB"
          pb: "PB"
          eb: "EB"
```

### Usage Examples

```ruby
# Dates
l(Date.today)                         # => "2026-03-06"
l(Date.today, format: :long)          # => "March 06, 2026"
l(Date.today, format: :short)         # => "Mar 06"

# Times
l(Time.current)                       # Default format
l(Time.current, format: :short)       # Short format

# Custom format on the fly
l(Time.current, format: "%B %Y")      # => "March 2026"

# Numbers
number_to_currency(1234.50)           # => "$1,234.50"
number_with_delimiter(1234567)        # => "1,234,567"
number_to_percentage(75.5)            # => "75.500%"
number_to_human_size(1234567)         # => "1.18 MB"
```

### Spanish Example

```yaml
es:
  date:
    formats:
      default: "%d/%m/%Y"
      long: "%d de %B de %Y"
      short: "%d %b"
    day_names:
      - domingo
      - lunes
      - martes
      - miércoles
      - jueves
      - viernes
      - sábado
    month_names:
      -
      - enero
      - febrero
      - marzo
      - abril
      - mayo
      - junio
      - julio
      - agosto
      - septiembre
      - octubre
      - noviembre
      - diciembre
  number:
    format:
      separator: ","
      delimiter: "."
    currency:
      format:
        unit: "€"
        format: "%n %u"
        separator: ","
        delimiter: "."
```

---

## Active Record Translations

### Complete Model Translation Example

```yaml
en:
  activerecord:
    models:
      user:
        one: "User"
        other: "Users"
      order:
        one: "Order"
        other: "Orders"

    attributes:
      user:
        email: "Email address"
        first_name: "First name"
        last_name: "Last name"
        role: "Role"
      order:
        total: "Total amount"
        status: "Status"
        placed_at: "Date placed"

    errors:
      models:
        user:
          attributes:
            email:
              blank: "is required"
              taken: "is already registered — did you mean to sign in?"
              invalid: "doesn't look right — check for typos"
            password:
              too_short: "must be at least %{count} characters"
        order:
          attributes:
            total:
              greater_than: "must be more than %{count}"
```

### Using ActiveModel (non-AR classes)

For classes that include `ActiveModel` but don't inherit from `ActiveRecord::Base`:

```yaml
en:
  activemodel:     # NOT activerecord
    models:
      contact_form: "Contact Form"
    attributes:
      contact_form:
        email: "Your email"
    errors:
      models:
        contact_form:
          attributes:
            email:
              blank: "is required"
```

### STI (Single Table Inheritance) Lookups

```ruby
class Admin < User; end
```

Error lookup order for `Admin` model, `name` attribute, `:blank` error:

```
activerecord.errors.models.admin.attributes.name.blank
activerecord.errors.models.admin.blank
activerecord.errors.models.user.attributes.name.blank   # ← walks up inheritance
activerecord.errors.models.user.blank
activerecord.errors.messages.blank
errors.attributes.name.blank
errors.messages.blank
```

### Nested Attributes

```yaml
en:
  activerecord:
    attributes:
      user/role:
        admin: "Administrator"
        member: "Member"
```

```ruby
User.human_attribute_name("role.admin")  # => "Administrator"
```

---

## Error Message Resolution Chain

### Full Resolution Order

For model `User`, attribute `email`, error `:blank`:

```
1. activerecord.errors.models.user.attributes.email.blank
2. activerecord.errors.models.user.blank
3. activerecord.errors.messages.blank
4. errors.attributes.email.blank
5. errors.messages.blank
```

### Available Interpolation Variables

All error messages have access to:
- `%{model}` — translated model name
- `%{attribute}` — translated attribute name
- `%{value}` — the current attribute value

Some validations also provide:
- `%{count}` — the validation option value (length, numericality thresholds)

### Full Message Format

```yaml
en:
  errors:
    # Default: "%{attribute} %{message}"
    format: "%{attribute} %{message}"

  # Per-model/attribute override (requires config):
  activerecord:
    errors:
      models:
        user:
          format: "%{attribute}: %{message}"
          attributes:
            email:
              format: "Email → %{message}"
```

Enable per-model format customization:

```ruby
# config/application.rb
config.active_model.i18n_customize_full_message = true
```

### Validation-to-Error-Key Mapping

| Validation | Option | Error Key | Extra Interpolation |
|-----------|--------|-----------|-------------------|
| `presence` | — | `:blank` | — |
| `uniqueness` | — | `:taken` | — |
| `format` | — | `:invalid` | — |
| `inclusion` | — | `:inclusion` | — |
| `exclusion` | — | `:exclusion` | — |
| `confirmation` | — | `:confirmation` | `attribute` |
| `acceptance` | — | `:accepted` | — |
| `absence` | — | `:present` | — |
| `length` | `:minimum` | `:too_short` | `count` |
| `length` | `:maximum` | `:too_long` | `count` |
| `length` | `:is` | `:wrong_length` | `count` |
| `length` | `:in` / `:within` | `:too_short` / `:too_long` | `count` |
| `numericality` | — | `:not_a_number` | — |
| `numericality` | `:only_integer` | `:not_an_integer` | — |
| `numericality` | `:greater_than` | `:greater_than` | `count` |
| `numericality` | `:greater_than_or_equal_to` | `:greater_than_or_equal_to` | `count` |
| `numericality` | `:equal_to` | `:equal_to` | `count` |
| `numericality` | `:less_than` | `:less_than` | `count` |
| `numericality` | `:less_than_or_equal_to` | `:less_than_or_equal_to` | `count` |
| `numericality` | `:other_than` | `:other_than` | `count` |
| `numericality` | `:odd` | `:odd` | — |
| `numericality` | `:even` | `:even` | — |
| `associated` | — | `:invalid` | — |

---

## Locale Switching Strategies

### Strategy 1: URL Path Scope (Most Common)

```ruby
# config/routes.rb
scope "(:locale)", locale: /en|es|fr/ do
  resources :books
  resources :users
  root "home#index"
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  around_action :switch_locale

  private

  def switch_locale(&action)
    locale = params[:locale] || I18n.default_locale
    I18n.with_locale(locale, &action)
  end

  def default_url_options
    { locale: I18n.locale }
  end
end
```

URLs: `/en/books`, `/es/books`, `/books` (uses default)

### Strategy 2: Subdomain

```ruby
def switch_locale(&action)
  locale = extract_locale_from_subdomain || I18n.default_locale
  I18n.with_locale(locale, &action)
end

def extract_locale_from_subdomain
  parsed = request.subdomains.first
  I18n.available_locales.map(&:to_s).include?(parsed) ? parsed.to_sym : nil
end
```

URLs: `en.example.com/books`, `es.example.com/books`

### Strategy 3: TLD (Top-Level Domain)

```ruby
def extract_locale_from_tld
  parsed = request.host.split(".").last
  I18n.available_locales.map(&:to_s).include?(parsed) ? parsed.to_sym : nil
end
```

URLs: `example.com/books` (en), `example.es/books` (es)

### Strategy 4: Accept-Language Header

```ruby
def extract_locale_from_header
  return nil unless request.env["HTTP_ACCEPT_LANGUAGE"]

  preferred = request.env["HTTP_ACCEPT_LANGUAGE"]
    .split(",")
    .map { |l| l.strip.split(";").first.split("-").first }
    .find { |l| I18n.available_locales.include?(l.to_sym) }

  preferred&.to_sym
end
```

### Strategy 5: User Preference (Database)

```ruby
# Migration
add_column :users, :locale, :string, default: "en", null: false

# Controller
def switch_locale(&action)
  locale = current_user&.locale&.to_sym || I18n.default_locale
  I18n.with_locale(locale, &action)
end
```

### Combined Strategy (Recommended)

```ruby
around_action :switch_locale

private

def switch_locale(&action)
  locale = resolve_locale
  I18n.with_locale(locale, &action)
end

def resolve_locale
  locale = params[:locale] ||
           current_user&.locale ||
           locale_from_header ||
           I18n.default_locale.to_s

  if I18n.available_locales.include?(locale.to_sym)
    locale.to_sym
  else
    I18n.default_locale
  end
end

def locale_from_header
  request.env["HTTP_ACCEPT_LANGUAGE"]
    &.scan(/^[a-z]{2}/)
    &.first
end
```

---

## Fallback Configuration

### Basic Fallbacks

```ruby
# config/application.rb

# All locales fall back to default_locale
config.i18n.fallbacks = true

# Specific fallback chains
config.i18n.fallbacks = {
  "es-MX": :es,      # Mexican Spanish → Spanish → default
  "pt-BR": :pt,      # Brazilian Portuguese → Portuguese → default
  fr: :en             # French → English (if default is something else)
}

# Disable fallbacks (recommended for test)
config.i18n.fallbacks = false
```

### Fallback in Initializer

```ruby
# config/initializers/locale.rb
Rails.application.config.after_initialize do
  I18n.fallbacks = I18n::Locale::Fallbacks.new(
    "es-MX": [:es, :en],
    "pt-BR": [:pt, :en]
  )
end
```

---

## Testing I18n

### Configuration for Tests

```ruby
# config/environments/test.rb
config.i18n.raise_on_missing_translations = true
```

This makes `MissingTranslationData` raise an exception instead of returning a "translation missing" span — you'll catch missing keys immediately.

For strict mode (also raises in models):

```ruby
config.i18n.raise_on_missing_translations = :strict
```

### Testing Translation Keys Exist

```ruby
# test/i18n_test.rb
require "test_helper"

class I18nTest < ActiveSupport::TestCase
  test "all locale files are valid YAML" do
    locale_files = Dir[Rails.root.join("config", "locales", "**", "*.yml")]
    locale_files.each do |file|
      assert YAML.load_file(file), "Invalid YAML: #{file}"
    end
  end

  test "all locales have the same keys" do
    en_keys = flatten_keys(I18n.backend.translations[:en])
    I18n.available_locales.reject { |l| l == :en }.each do |locale|
      locale_keys = flatten_keys(I18n.backend.translations[locale])
      missing = en_keys - locale_keys
      assert missing.empty?, "#{locale} missing keys: #{missing.join(', ')}"
    end
  end

  private

  def flatten_keys(hash, prefix = nil)
    hash.each_with_object([]) do |(k, v), keys|
      key = [prefix, k].compact.join(".")
      if v.is_a?(Hash)
        keys.concat(flatten_keys(v, key))
      else
        keys << key
      end
    end
  end
end
```

### Testing with Specific Locales

```ruby
test "shows Spanish content" do
  I18n.with_locale(:es) do
    get root_path, params: { locale: :es }
    assert_response :success
    assert_select "h1", I18n.t("home.index.title")
  end
end
```

### i18n-tasks Gem (Recommended)

```ruby
# Gemfile
gem "i18n-tasks", group: [:development, :test]
```

```bash
# Find missing translations
bundle exec i18n-tasks missing

# Find unused translations
bundle exec i18n-tasks unused

# Normalize/sort locale files
bundle exec i18n-tasks normalize

# Check consistency
bundle exec i18n-tasks health
```

---

## Localized Views

Rails supports locale-specific view templates:

```
app/views/books/
  index.html.erb       # Default (any locale)
  index.es.html.erb    # Spanish-specific
  index.ja.html.erb    # Japanese-specific
```

When `I18n.locale = :es`, Rails renders `index.es.html.erb`. Falls back to `index.html.erb` if locale-specific template doesn't exist.

**Use sparingly.** Localized views are useful for:
- Significantly different layouts per locale (RTL vs LTR)
- Large blocks of static content
- Legal pages that differ by region

For most cases, YAML translations with `t()` are more maintainable.

---

## Custom Backends

### Chain Backend (Most Common)

Combine multiple backends — look up in order, first match wins:

```ruby
# config/initializers/i18n.rb
I18n.backend = I18n::Backend::Chain.new(
  I18n::Backend::ActiveRecord.new,  # Check DB first
  I18n.backend                       # Fall back to YAML files
)
```

### Key-Value Backend

Store translations in Redis, database, etc.:

```ruby
# Using i18n-active_record gem
gem "i18n-active_record"

# config/initializers/i18n.rb
I18n.backend = I18n::Backend::Chain.new(
  I18n::Backend::ActiveRecord.new,
  I18n::Backend::Simple.new
)
```

### Cache Backend

Wrap any backend with caching:

```ruby
I18n::Backend::Simple.include(I18n::Backend::Cache)
I18n.cache_store = ActiveSupport::Cache.lookup_store(:memory_store)
```

---

## Common Gems

### rails-i18n

Locale data for Rails (date/time formats, pluralization rules, etc.) for 100+ languages.

```ruby
gem "rails-i18n", "~> 8.0"  # Match your Rails version
```

This gives you pre-built translations for:
- Date/time format strings
- Day/month names
- Number formatting (separators, delimiters)
- Pluralization rules
- Distance of time in words
- Active Record error messages

### i18n-tasks

Static analysis for translation keys.

```ruby
gem "i18n-tasks", group: [:development, :test]
```

### mobility

Translate model content (database columns).

```ruby
gem "mobility"

# In model:
class Post < ApplicationRecord
  extend Mobility
  translates :title, :body, backend: :key_value
end

# Usage:
I18n.with_locale(:es) { post.title = "Título" }
```

### route_translator

Auto-translate routes.

```ruby
gem "route_translator"

# config/routes.rb
localized do
  resources :books  # /en/books, /es/libros, etc.
end
```

---

## Edge Cases and Gotchas

### 1. Thread Safety — NEVER Use `I18n.locale =`

```ruby
# ❌ DANGEROUS — leaks to other requests in threaded servers (Puma)
I18n.locale = :es
do_something
I18n.locale = I18n.default_locale

# ✅ SAFE — scoped to block
I18n.with_locale(:es) do
  do_something
end
```

### 2. New Locale Files Require Server Restart

Locale files are loaded at boot. Adding a new `.yml` file requires restarting the server. Editing an existing file works with Spring/reloader in development.

### 3. YAML Indentation Matters

```yaml
# ❌ Wrong indentation — "name" is NOT under "attributes"
en:
  activerecord:
    attributes:
    user:
      name: "Name"

# ✅ Correct
en:
  activerecord:
    attributes:
      user:
        name: "Name"
```

### 4. Lazy Lookup Doesn't Work Everywhere

Lazy lookup (`.key`) works in:
- ✅ Views (action templates and partials)
- ✅ Controllers (action methods)

Lazy lookup does NOT work in:
- ❌ Models
- ❌ Service objects
- ❌ Mailer bodies (works in subjects via `default_i18n_subject`)
- ❌ Jobs
- ❌ Decorators/presenters

### 5. Pluralization Requires `:count`

```ruby
# ❌ Returns the HASH, not a string
t("notifications")
# => { one: "1 notification", other: "%{count} notifications" }

# ✅ Pass count to trigger pluralization
t("notifications", count: 5)
# => "5 notifications"
```

### 6. HTML Safety is View-Only

The `_html` suffix auto-marks translations as `html_safe` ONLY when using the `t` view helper. In models/services, `I18n.t("key_html")` returns a plain string — you'd need to call `.html_safe` manually (be careful with XSS).

### 7. Scope Separator is "."

Dots in translation keys require array syntax:

```yaml
en:
  "3.5_inch_floppy": "Floppy disk"  # ❌ Confusing — looks like nested key
```

```ruby
I18n.t("3.5_inch_floppy")  # Looks up key "5_inch_floppy" under "3"!

# Use array scope instead:
I18n.t(:"3.5_inch_floppy")  # Still broken

# Best: avoid dots in key names entirely
```

### 8. Default Locale Fallback for Missing Keys

```ruby
# With fallbacks enabled:
# config.i18n.fallbacks = true

# If es.yml has no "greeting" key, falls back to en.yml
I18n.with_locale(:es) { I18n.t("greeting") }
# => English greeting (if es translation missing)
```

### 9. Locale Files Merge, Not Replace

Multiple YAML files for the same locale are deep-merged. This means you can split translations across files:

```yaml
# config/locales/models/en.yml
en:
  activerecord:
    models:
      user: "User"

# config/locales/views/en.yml
en:
  users:
    index:
      title: "All Users"
```

Both are available. If the same key exists in multiple files, the last-loaded file wins (load order depends on file system + load_path order).

### 10. Percent Signs in YAML

```yaml
en:
  # ❌ Breaks — %{} triggers interpolation
  discount: "50% off"

  # ✅ Works — not a valid interpolation variable name
  discount: "50% off"  # Actually fine since "% " isn't "%{...}"

  # Only "%{variable_name}" triggers interpolation
  # Literal "%" followed by anything except "{" is safe
```

### 11. Testing Tip: Temporarily Force Locale

```ruby
# In test setup
setup do
  @original_locale = I18n.locale
end

teardown do
  I18n.locale = @original_locale
end

# Or better: use I18n.with_locale in each test
test "Spanish page" do
  I18n.with_locale(:es) do
    # ...
  end
end
```
