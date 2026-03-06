# Rails Generators Reference

Detailed patterns, examples, and edge cases for Rails generators.

## Complete Built-In Generator Reference

### model

Creates model class, migration, test, and fixture.

```bash
bin/rails g model User name email:string:uniq age:integer admin:boolean

# With references
bin/rails g model Comment body:text post:references user:references

# With polymorphic reference
bin/rails g model Comment body:text commentable:references{polymorphic}

# With specific index
bin/rails g model Product name sku:string:uniq price:decimal{8,2}

# Skip migration (model class only)
bin/rails g model Calculation --skip-migration

# With namespace
bin/rails g model Admin::User name email
# Creates: app/models/admin/user.rb, db/migrate/..._create_admin_users.rb

# With parent class (STI)
bin/rails g model AdminUser --parent=User
# Creates model inheriting from User, no migration
```

**Files created:**
- `app/models/[name].rb`
- `db/migrate/TIMESTAMP_create_[table].rb`
- `test/models/[name]_test.rb`
- `test/fixtures/[table].yml`

**Common flags:**
| Flag | Effect |
|------|--------|
| `--skip-migration` | No migration file |
| `--no-test-framework` | No test or fixture |
| `--no-fixture` | No fixture file |
| `--primary-key-type=uuid` | UUID primary key |
| `--parent=ClassName` | STI — inherit from parent, skip migration |
| `--database=animals` | Multi-DB: generate for specific database |

---

### migration

Creates a standalone migration file. The migration name determines content.

```bash
# Add columns — name pattern: AddXxxToYyy
bin/rails g migration AddEmailToUsers email:string
# => def change; add_column :users, :email, :string; end

bin/rails g migration AddIndexToUsersEmail
# => empty change (no columns specified, just the name)

bin/rails g migration AddDetailsToProducts price:decimal{8,2} supplier:references
# => adds both columns

# Remove columns — name pattern: RemoveXxxFromYyy
bin/rails g migration RemoveAgeFromUsers age:integer
# => def change; remove_column :users, :age, :integer; end

# Create table — name pattern: CreateXxx
bin/rails g migration CreateProducts name:string price:decimal
# => create_table with those columns

# Create join table — name pattern: CreateJoinTableXxxYyy
bin/rails g migration CreateJoinTableProductsCategories products categories
# => create_join_table :products, :categories

# Generic name (empty change method)
bin/rails g migration ConsolidateUserData
# => empty def change; end — fill in manually
```

**Name pattern inference rules:**
| Pattern | Inferred Action |
|---------|----------------|
| `AddXxxToYyy field:type` | `add_column :yyy, :field, :type` |
| `RemoveXxxFromYyy field:type` | `remove_column :yyy, :field, :type` |
| `CreateXxx field:type` | `create_table` with columns |
| `CreateJoinTableXxxYyy` | `create_join_table` |
| Anything else | Empty `change` method |

**Column modifiers in migration generator:**
```bash
name:string{40}              # limit: 40
price:decimal{10,2}          # precision: 10, scale: 2
email:string:index           # adds add_index
slug:string:uniq             # adds add_index unique: true
user:references              # adds foreign key + belongs_to
commentable:references{polymorphic}  # polymorphic reference
```

---

### controller

Creates controller, views, route entries, test, and helper.

```bash
bin/rails g controller Posts index show new create

# Namespaced
bin/rails g controller Admin::Posts index show

# API-only (no views)
bin/rails g controller Api::V1::Posts index show --no-helper --skip-routes
```

**Files created:**
- `app/controllers/[name]_controller.rb` (with empty action methods)
- `app/views/[name]/[action].html.erb` (for each action)
- `test/controllers/[name]_controller_test.rb`
- `app/helpers/[name]_helper.rb`
- Route entries for each action

**Common flags:**
| Flag | Effect |
|------|--------|
| `--no-helper` | Skip helper file |
| `--skip-routes` | Don't add routes |
| `--no-test-framework` | Skip test file |
| `--api` | Skip views and helpers |

**Key difference from scaffold:** Controller generator creates individual route entries (`get 'posts/index'`), NOT `resources :posts`. Use `resource` or `scaffold` for RESTful routes.

---

### scaffold

Creates everything for a full CRUD resource.

```bash
bin/rails g scaffold Post title body:text published:boolean user:references

# API-only (no views, no view tests)
bin/rails g scaffold Post title body:text --api

# With specific test framework
bin/rails g scaffold Post title --test-framework=rspec

# With UUID primary keys
bin/rails g scaffold Post title --primary-key-type=uuid
```

**Files created:**
- Model + migration + fixture
- Controller with all 7 RESTful actions
- Views (index, show, new, edit, _form, _resource)
- Route (`resources :posts`)
- Helper
- Test files (controller + system)
- Jbuilder templates (if jbuilder is present)

**Common flags:**
| Flag | Effect |
|------|--------|
| `--api` | API-only (no views, no helpers) |
| `--no-helper` | Skip helper |
| `--no-jbuilder` | Skip JSON builder views |
| `--no-assets` | Skip asset files |
| `--no-test-framework` | Skip all tests |
| `--force` | Overwrite existing files |

---

### resource

Like scaffold but generates an **empty** controller (no actions, no views).

```bash
bin/rails g resource Post title body:text
```

**Creates:** model + migration + empty controller + `resources :posts` route + test + fixture

**Use when:** You want the model, route, and controller file but will write controller actions yourself.

---

### scaffold_controller

Creates controller + views for an **existing** model. No model or migration.

```bash
bin/rails g scaffold_controller Post title body:text

# API-only
bin/rails g scaffold_controller Post title body:text --api

# Specify model name explicitly
bin/rails g scaffold_controller Admin::Post title body:text
```

**Use when:** Model already exists (maybe from a previous `bin/rails g model`), and you now need the controller + views.

---

### job

```bash
bin/rails g job ProcessPayment

# With queue name
bin/rails g job ProcessPayment --queue=payments
```

**Creates:**
- `app/jobs/process_payment_job.rb`
- `test/jobs/process_payment_job_test.rb`

---

### mailer

```bash
bin/rails g mailer UserMailer welcome password_reset

# Namespaced
bin/rails g mailer Admin::NotificationMailer alert digest
```

**Creates:**
- `app/mailers/user_mailer.rb` (with methods for each action)
- `app/views/user_mailer/welcome.html.erb`
- `app/views/user_mailer/welcome.text.erb`
- `app/views/user_mailer/password_reset.html.erb`
- `app/views/user_mailer/password_reset.text.erb`
- `test/mailers/user_mailer_test.rb`
- `test/mailers/previews/user_mailer_preview.rb`

---

### channel (Action Cable)

```bash
bin/rails g channel Chat speak

bin/rails g channel Notifications
```

**Creates:**
- `app/channels/chat_channel.rb`
- `test/channels/chat_channel_test.rb`

---

### mailbox (Action Mailbox)

```bash
bin/rails g mailbox Forwards
```

**Creates:**
- `app/mailboxes/forwards_mailbox.rb`
- `test/mailboxes/forwards_mailbox_test.rb`

---

### task (Rake)

```bash
bin/rails g task data_migration migrate_users cleanup_orphans
```

**Creates:**
- `lib/tasks/data_migration.rake` (with `data_migration:migrate_users` and `data_migration:cleanup_orphans` tasks)

---

### integration_test / system_test

```bash
bin/rails g integration_test UserRegistration
bin/rails g system_test UserRegistration
```

**Creates:** Test file in `test/integration/` or `test/system/`.

---

### helper

```bash
bin/rails g helper Posts
```

**Creates:** `app/helpers/posts_helper.rb` + test.

---

### generator

Creates a custom generator skeleton.

```bash
bin/rails g generator service

# Namespaced (for overriding Rails generators)
bin/rails g generator rails/my_helper
```

**Creates:**
- `lib/generators/service/service_generator.rb`
- `lib/generators/service/USAGE`
- `lib/generators/service/templates/`
- `test/lib/generators/service_generator_test.rb`

---

## Customizing Generator Defaults

### config.generators

Set in `config/application.rb` to change defaults for ALL generator invocations:

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    config.generators do |g|
      # ORM settings
      g.orm :active_record, primary_key_type: :uuid

      # Test framework
      g.test_framework :test_unit, fixture: true
      # Or: g.test_framework :rspec, fixture: true
      # Or: g.test_framework nil  # disable test generation

      # Template engine
      g.template_engine :erb
      # Or: g.template_engine :slim
      # Or: g.template_engine :haml

      # Skip files globally
      g.helper false
      g.assets false
      g.stylesheets false
      g.javascripts false
      g.jbuilder false
      g.system_tests nil

      # Custom frameworks
      g.scaffold_controller :scaffold_controller
      g.resource_route :resource_route
    end
  end
end
```

### Per-Environment Generator Config

```ruby
# config/environments/development.rb
config.generators do |g|
  g.test_framework nil  # Skip tests in development generators
end
```

### Checking Current Config

```bash
# From Rails console
rails runner "pp Rails.application.config.generators"

# Or grep config files
grep -rn "config.generators" config/
```

---

## Creating Custom Generators — In Depth

### Generator Base Classes

| Base Class | Requires Argument | Use For |
|------------|-------------------|---------|
| `Rails::Generators::Base` | No | Generators that don't take a NAME argument |
| `Rails::Generators::NamedBase` | Yes (NAME) | Generators that create something named (most common) |

### Full Custom Generator Example

```ruby
# lib/generators/service/service_generator.rb
class ServiceGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  # Arguments (positional, after NAME)
  argument :methods, type: :array, default: [], banner: "method method"

  # Options (--flag style)
  class_option :module, type: :string, default: nil,
    desc: "Wrap service in a module"
  class_option :skip_test, type: :boolean, default: false,
    desc: "Skip test file generation"

  def create_service_file
    template "service.rb.tt", File.join("app/services", class_path, "#{file_name}_service.rb")
  end

  def create_test_file
    return if options[:skip_test]
    template "service_test.rb.tt", File.join("test/services", class_path, "#{file_name}_service_test.rb")
  end

  # Hook into test framework (like Rails built-in generators do)
  hook_for :test_framework, as: :service

  private

  def module_name
    options[:module]
  end

  def service_class_name
    "#{class_name}Service"
  end
end
```

### Template (.tt) Files

Templates use ERB and have access to all generator methods:

```ruby
# lib/generators/service/templates/service.rb.tt
<% if module_name -%>
module <%= module_name %>
  <% end -%>
class <%= service_class_name %>
  def initialize(<%= file_name %>)
    @<%= file_name %> = <%= file_name %>
  end

<% methods.each do |method| -%>
  def <%= method %>
    # TODO: implement
  end

<% end -%>
  def call
    # TODO: implement
  end

  private

  attr_reader :<%= file_name %>
<% if module_name -%>
end
<% end -%>
end
```

### USAGE File

```
# lib/generators/service/USAGE
Description:
    Creates a service object class and test.

Example:
    bin/rails generate service ProcessPayment charge refund

    This will create:
        app/services/process_payment_service.rb
        test/services/process_payment_service_test.rb
```

---

## Thor Actions Reference

Generators inherit from Thor and have access to these file manipulation methods:

### File Creation & Copying

```ruby
# Create a file with content
create_file "config/initializers/my_init.rb", <<~RUBY
  MyApp.configure do |config|
    config.setting = true
  end
RUBY

# Copy a template (ERB processed)
template "service.rb.tt", "app/services/#{file_name}_service.rb"

# Copy a file without ERB processing
copy_file "config.yml", "config/my_config.yml"

# Create empty directory
empty_directory "app/services"
```

### File Modification

```ruby
# Append to file
append_to_file "Gemfile", "\ngem 'devise'"

# Prepend to file
prepend_to_file "app/models/user.rb", "# frozen_string_literal: true\n\n"

# Insert at specific point
inject_into_file "config/routes.rb", after: "Rails.application.routes.draw do\n" do
  "  root to: 'home#index'\n"
end

# Insert before a line
inject_into_file "app/models/user.rb", before: "end\n" do
  "  has_many :posts\n"
end

# Global search and replace
gsub_file "config/database.yml", /sqlite3/, "postgresql"

# Comment/uncomment lines
comment_lines "Gemfile", /gem 'sqlite3'/
uncomment_lines "config/environments/production.rb", /config.force_ssl/
```

### Running Commands

```ruby
# Run shell command
run "bundle install"

# Run Rails command
rails_command "db:migrate"

# Run Rake task
rake "db:seed"

# Run inside a directory
inside("lib") do
  run "touch my_file.rb"
end
```

### User Interaction

```ruby
# Ask a question
name = ask("What's the project name?")

# Yes/no question
if yes?("Install Devise?")
  gem "devise"
end

# Say something
say "Creating service object...", :green
say_status "create", "app/services/foo_service.rb", :green
```

### Conflict Resolution

```ruby
# Check if file exists
if File.exist?(destination)
  say "File already exists, skipping", :yellow
  return
end

# Or use built-in behavior_on_conflict
# Generator automatically asks: Ynaqdh (Yes/No/All/Quit/Diff/Help)
```

---

## Overriding Built-In Templates — In Depth

### Template Override Locations

Place templates in `lib/templates/` mirroring the generator's namespace:

```
lib/templates/
├── active_record/
│   ├── migration/
│   │   ├── create_table_migration.rb.tt
│   │   └── migration.rb.tt
│   └── model/
│       └── model.rb.tt
├── erb/
│   ├── controller/
│   │   └── view.html.erb.tt
│   └── scaffold/
│       ├── _form.html.erb.tt
│       ├── _resource.html.erb.tt
│       ├── edit.html.erb.tt
│       ├── index.html.erb.tt
│       ├── new.html.erb.tt
│       └── show.html.erb.tt
├── rails/
│   └── scaffold_controller/
│       └── controller.rb.tt
└── test_unit/
    ├── model/
    │   └── unit_test.rb.tt
    ├── scaffold/
    │   └── functional_test.rb.tt
    └── integration/
        └── integration_test.rb.tt
```

### Finding Original Templates

To find the original template to copy and modify:

```bash
# Find where railties gem is installed
bundle show railties

# Browse generator templates
find $(bundle show railties)/lib/rails/generators -name "*.tt" | sort

# Copy a template to override it
cp $(bundle show railties)/lib/rails/generators/erb/scaffold/templates/index.html.erb.tt \
   lib/templates/erb/scaffold/index.html.erb.tt
```

### Example: Custom Model Template

```ruby
# lib/templates/active_record/model/model.rb.tt
<% module_namespacing do -%>
class <%= class_name %> < <%= parent_class_name.classify %>
<% attributes.select(&:reference?).each do |attribute| -%>
  belongs_to :<%= attribute.name %><%= ", polymorphic: true" if attribute.polymorphic? %>
<% end -%>
<% attributes.select(&:token?).each do |attribute| -%>
  has_secure_token<%= ":#{attribute.name}" unless attribute.name == "token" %>
<% end -%>
<% if attributes.any?(&:password_digest?) -%>
  has_secure_password
<% end -%>

  # Validations
<% attributes.reject(&:reference?).each do |attribute| -%>
  validates :<%= attribute.name %>, presence: true
<% end -%>
end
<% end -%>
```

### Example: Custom Scaffold Views

```erb
<%# lib/templates/erb/scaffold/index.html.erb.tt %>
<div class="container mx-auto px-4">
  <h1 class="text-2xl font-bold mb-4"><%%= @<%= plural_table_name %>.count %> <%= human_name.pluralize %></h1>

  <%%= link_to "New <%= human_name %>", new_<%= singular_route_name %>_path, class: "btn btn-primary" %>

  <div class="mt-4 space-y-4">
    <%% @<%= plural_table_name %>.each do |<%= singular_table_name %>| %>
      <%%= render <%= singular_table_name %> %>
    <%% end %>
  </div>
</div>
```

---

## Generator Hooks and Composition

### hook_for

Generators can delegate parts of their work to other generators:

```ruby
class ScaffoldGenerator < Rails::Generators::NamedBase
  hook_for :orm, required: true                    # Invokes active_record:model
  hook_for :resource_route, required: true         # Invokes resource_route
  hook_for :scaffold_controller, required: true    # Invokes scaffold_controller
  hook_for :test_framework, as: :scaffold          # Invokes test_unit:scaffold
end
```

### Overriding Hooked Generators

```ruby
# config/application.rb
config.generators do |g|
  g.helper :my_helper              # Replace helper generator
  g.test_framework :my_test_unit   # Replace test generator
  g.orm :my_orm                    # Replace ORM generator
  g.template_engine :slim          # Use Slim instead of ERB
end
```

### Fallbacks

When you want to override only SOME generators in a namespace:

```ruby
# config/application.rb
config.generators do |g|
  g.test_framework :my_test_unit
  g.fallbacks[:my_test_unit] = :test_unit
end
```

This uses `my_test_unit:model` if it exists, but falls back to `test_unit:controller` etc.

---

## Generator Testing

### Test Structure

```ruby
# test/lib/generators/service_generator_test.rb
require "test_helper"
require "generators/service/service_generator"

class ServiceGeneratorTest < Rails::Generators::TestCase
  tests ServiceGenerator
  destination Rails.root.join("tmp/generators")
  setup :prepare_destination

  test "creates service file" do
    run_generator ["user_signup"]
    assert_file "app/services/user_signup_service.rb" do |content|
      assert_match(/class UserSignupService/, content)
      assert_match(/def call/, content)
    end
  end

  test "creates test file" do
    run_generator ["user_signup"]
    assert_file "test/services/user_signup_service_test.rb" do |content|
      assert_match(/class UserSignupServiceTest/, content)
    end
  end

  test "respects --skip-test flag" do
    run_generator ["user_signup", "--skip-test"]
    assert_no_file "test/services/user_signup_service_test.rb"
  end

  test "handles namespaced names" do
    run_generator ["admin/user_cleanup"]
    assert_file "app/services/admin/user_cleanup_service.rb" do |content|
      assert_match(/module Admin/, content)
      assert_match(/class UserCleanupService/, content)
    end
  end
end
```

### Available Test Assertions

```ruby
# File existence
assert_file "path/to/file.rb"
assert_no_file "path/to/file.rb"
assert_directory "app/services"

# File content
assert_file "path/to/file.rb" do |content|
  assert_match(/expected_pattern/, content)
  assert_instance_of String, content
end

# File content (shorthand)
assert_file "path/to/file.rb", /class MyService/

# Migration existence (handles timestamps)
assert_migration "db/migrate/create_posts.rb"
assert_migration "db/migrate/create_posts.rb" do |content|
  assert_match(/t.string :title/, content)
end

# Route assertions
assert_file "config/routes.rb", /resources :posts/
```

### Running Generator Tests

```bash
# Run specific generator test
RAILS_LOG_TO_STDOUT=true bin/rails test test/lib/generators/service_generator_test.rb

# Run all generator tests
RAILS_LOG_TO_STDOUT=true bin/rails test test/lib/generators/
```

---

## Edge Cases and Gotchas

### Namespaced Models

```bash
# Creates Admin::User model
bin/rails g model Admin::User name email

# Files created:
# app/models/admin.rb (module)
# app/models/admin/user.rb
# db/migrate/..._create_admin_users.rb
# test/models/admin/user_test.rb
# test/fixtures/admin/users.yml
```

### STI (Single Table Inheritance)

```bash
# Create base model
bin/rails g model Vehicle type name

# Create STI subclasses (no migration needed)
bin/rails g model Car --parent=Vehicle
bin/rails g model Truck --parent=Vehicle
```

### API-Only Applications

In API-only apps (`config.api_only = true`):
- Scaffold generates API controllers (no views)
- No helper generation
- JSON responses by default
- `--api` flag is implicit

### UUID Primary Keys

```ruby
# config/application.rb
config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end

# Also need in config/initializers/generators.rb or migration:
# enable_extension 'pgcrypto' (PostgreSQL)
```

### Multi-Database

```bash
# Generate migration for specific database
bin/rails g migration AddNameToAnimals name --database=animals

# Generate model for specific database
bin/rails g model Dog name --database=animals
```

### Token Attributes

```bash
bin/rails g model User token:token
# Generates: has_secure_token in model

bin/rails g model User auth_token:token
# Generates: has_secure_token :auth_token
```

### Password Digest

```bash
bin/rails g model User password:digest
# Generates: has_secure_password in model
# Adds password_digest string column to migration
# Adds bcrypt gem to Gemfile
```

### Rich Text (Action Text)

```bash
bin/rails g model Article title body:rich_text
# Note: rich_text type doesn't create a column
# It adds has_rich_text :body to the model
```

### Attachments (Active Storage)

```bash
bin/rails g model User avatar:attachment
bin/rails g model Product images:attachments
# Adds has_one_attached :avatar or has_many_attached :images
```

---

## Generator Resolution Order

When you run `bin/rails g foo`, Rails searches for the generator in this order:

1. `rails/generators/foo/foo_generator.rb`
2. `generators/foo/foo_generator.rb`
3. `rails/generators/foo_generator.rb`
4. `generators/foo_generator.rb`

For namespaced generators like `bin/rails g my_gem:install`:

1. `rails/generators/my_gem/install/install_generator.rb`
2. `generators/my_gem/install/install_generator.rb`
3. `rails/generators/my_gem/install_generator.rb`
4. `generators/my_gem/install_generator.rb`

### Where to Place Custom Generators

| Location | When |
|----------|------|
| `lib/generators/` | App-specific generators |
| `lib/generators/rails/` | Override Rails built-in generators |
| Gem's `lib/generators/` | Gem-provided generators |

---

## Application Templates

Templates automate new Rails app setup. Used with `rails new -m template.rb`.

### Template API Methods

```ruby
# template.rb

# Add gems
gem "devise"
gem "sidekiq"
gem_group :development, :test do
  gem "debug"
  gem "brakeman"
end

# Run after bundle install
after_bundle do
  # Run generators
  generate "devise:install"
  generate "devise", "User"

  # Run Rails commands
  rails_command "db:migrate"

  # Add routes
  route 'root to: "home#index"'

  # Add environment config
  environment 'config.action_mailer.default_url_options = {host: "localhost:3000"}', env: "development"

  # Create initializer
  initializer "sidekiq.rb", <<~RUBY
    Sidekiq.configure_server do |config|
      config.redis = { url: ENV.fetch("REDIS_URL", "redis://localhost:6379/1") }
    end
  RUBY

  # Create files
  file "app/services/application_service.rb", <<~RUBY
    class ApplicationService
      def self.call(...)
        new(...).call
      end
    end
  RUBY

  # Git
  git :init
  git add: "."
  git commit: "-m 'Initial commit'"
end
```

### Using Templates

```bash
# With new app
rails new myapp -m ~/template.rb
rails new myapp -m https://example.com/template.rb

# Apply to existing app
bin/rails app:template LOCATION=~/template.rb
bin/rails app:template LOCATION=https://example.com/template.rb
```

---

## Recipes: Common Generator Patterns

### Service Object Generator

```bash
bin/rails g generator service
```

```ruby
# lib/generators/service/service_generator.rb
class ServiceGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  def create_service
    template "service.rb.tt", "app/services/#{file_path}_service.rb"
  end

  def create_test
    template "service_test.rb.tt", "test/services/#{file_path}_service_test.rb"
  end
end
```

### Form Object Generator

```ruby
# lib/generators/form/form_generator.rb
class FormGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  argument :attributes, type: :array, default: [], banner: "field:type field:type"

  def create_form
    template "form.rb.tt", "app/forms/#{file_path}_form.rb"
  end

  def create_test
    template "form_test.rb.tt", "test/forms/#{file_path}_form_test.rb"
  end
end
```

### Query Object Generator

```ruby
# lib/generators/query/query_generator.rb
class QueryGenerator < Rails::Generators::NamedBase
  source_root File.expand_path("templates", __dir__)

  def create_query
    template "query.rb.tt", "app/queries/#{file_path}_query.rb"
  end

  def create_test
    template "query_test.rb.tt", "test/queries/#{file_path}_query_test.rb"
  end
end
```

---

## Troubleshooting

### "Could not find generator"

```bash
# Check available generators
bin/rails g --help

# Check if the gem providing the generator is installed
bundle list | grep generator_gem_name

# Check load path
bin/rails runner "puts $LOAD_PATH"
```

### "Another migration is already named"

```bash
# Check existing migrations
ls db/migrate/*create_posts*

# Use a different name or destroy first
bin/rails d migration CreatePosts
bin/rails g migration CreatePosts title:string
```

### Generated File Has Wrong Content

```bash
# Check for template overrides
find lib/templates -name "*.tt" 2>/dev/null

# Check generator config
grep -A 20 "config.generators" config/application.rb

# Use --pretend to see what would happen
bin/rails g model Post title --pretend
```

### Scaffold Generating Too Many Files

```ruby
# Add to config/application.rb
config.generators do |g|
  g.helper false
  g.assets false
  g.jbuilder false
  g.system_tests nil
end
```

Then scaffold only generates what you actually need.
