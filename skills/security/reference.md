# Rails Security Reference

Detailed patterns, edge cases, and examples for every major security concern in Rails applications.

---

## CSRF Deep Dive

### How CSRF Works

Rails embeds a unique `authenticity_token` in every form and verifies it on POST/PATCH/PUT/DELETE requests. The token proves the request originated from your app, not a malicious third-party site.

### Token Flow

1. Rails generates a per-session CSRF token
2. `<%= csrf_meta_tags %>` embeds it in the layout (for Turbo/JS)
3. `form_with` / `form_for` automatically include a hidden `authenticity_token` field
4. Server verifies the token on every non-GET request

### Common Mistakes

```ruby
# MISTAKE: Skipping CSRF for webhook endpoints that also serve browsers
class WebhooksController < ApplicationController
  skip_forgery_protection  # Fine ONLY if this never uses cookie auth
end

# MISTAKE: Using protect_from_forgery with: :null_session globally
# This silently clears the session instead of raising — attacker gets a blank session
class ApplicationController < ActionController::Base
  protect_from_forgery with: :null_session  # BAD for browser apps
end

# CORRECT: Use :exception for browser apps, :null_session only for pure APIs
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end

class Api::BaseController < ActionController::API
  # Inherits from API — no session, no CSRF needed
  # BUT must authenticate via token header, not cookies
end
```

### CSRF with JavaScript/Turbo

Turbo automatically reads the CSRF token from meta tags. For custom fetch/XHR:

```javascript
// Read token from meta tag
const token = document.head.querySelector("meta[name=csrf-token]")?.content;

// Include in fetch requests
fetch("/posts", {
  method: "POST",
  headers: {
    "X-CSRF-Token": token,
    "Content-Type": "application/json"
  },
  body: JSON.stringify({ post: { title: "New Post" } })
});
```

### API-Only Controllers

If your API uses token authentication (Bearer tokens, API keys) and never reads cookies, CSRF protection is unnecessary. But if your API shares cookies with the browser app, you MUST keep CSRF.

```ruby
# Safe: Token-only API
class Api::V1::BaseController < ActionController::API
  before_action :authenticate_api_token

  private

  def authenticate_api_token
    token = request.headers["Authorization"]&.remove("Bearer ")
    @current_api_user = ApiToken.find_by(token: token)&.user
    head :unauthorized unless @current_api_user
  end
end
```

---

## XSS Deep Dive

### How Rails Auto-Escaping Works

ERB's `<%= %>` calls `html_escape` on the output unless the string is already marked `html_safe`. This is why `<%= user.name %>` is safe — it escapes `<`, `>`, `&`, `"`, `'`.

### The html_safe Trap

Calling `.html_safe` marks a string as "already escaped" — Rails will NOT escape it again. This is the #1 way agents introduce XSS:

```ruby
# String marked html_safe is NEVER escaped again
"<script>alert('XSS')</script>".html_safe
# => Rendered as literal script tag. XSS!

# Concatenation with + preserves html_safe status of the LEFT operand
safe = "<b>Bold</b>".html_safe
unsafe = params[:name]
result = safe + unsafe  # unsafe part IS escaped — safe
result = unsafe + safe  # NEITHER is escaped — XSS if unsafe has HTML
```

### Safe Patterns for Building HTML in Helpers

```ruby
# BEST: Use tag helpers
def status_badge(status)
  tag.span(status, class: "badge badge-#{status}")
end

# GOOD: Use content_tag
def icon_link(text, url)
  content_tag(:a, href: url) do
    content_tag(:i, "", class: "icon") + " " + text
  end
end

# ACCEPTABLE: Manual escaping with string building
def labeled_value(label, value)
  "<dt>#{ERB::Util.html_escape(label)}</dt><dd>#{ERB::Util.html_escape(value)}</dd>".html_safe
end

# WRONG: Interpolating user data into html_safe string
def labeled_value(label, value)
  "<dt>#{label}</dt><dd>#{value}</dd>".html_safe  # XSS!
end
```

### sanitize() In Depth

```ruby
# Default allowed tags/attributes — check ActionView::Helpers::SanitizeHelper
sanitize(user_html)

# Explicit allowlist — always preferred
sanitize(user_html,
  tags: %w[p br strong em a ul ol li h2 h3 blockquote code pre],
  attributes: %w[href title class]
)

# Strip ALL HTML
strip_tags(user_html)

# Strip links only
strip_links(user_html)
```

### Action Text

Action Text sanitizes rich text content automatically through its own sanitizer. If you use Action Text for user-generated rich content, you get XSS protection for free — but review the allowed tags if you customize.

### XSS in JavaScript Context

Auto-escaping only protects HTML context. Data injected into JavaScript needs different handling:

```erb
<!-- WRONG — XSS in JS context -->
<script>
  var name = "<%= @user.name %>";  // Escapes HTML, not JS
</script>

<!-- CORRECT — use json_escape or to_json -->
<script>
  var name = <%= @user.name.to_json %>;
  var data = <%= json_escape(@data.to_json) %>;
</script>

<!-- BEST — use data attributes -->
<div data-name="<%= @user.name %>" data-config="<%= @config.to_json %>">
</div>
```

### Content-Type Matters

```ruby
# Rendering user content as HTML is dangerous
render html: user_input  # XSS if not escaped

# Rendering as plain text is safe
render plain: user_input

# JSON responses auto-escape HTML entities
render json: { name: user_input }  # Safe — JSON-encoded
```

---

## SQL Injection Deep Dive

### Safe vs. Unsafe ActiveRecord Methods

**Always safe (handle escaping internally):**
```ruby
User.find(id)
User.find_by(name: value)
User.where(name: value)
User.where(name: [val1, val2])
User.order(name: :asc)
User.pluck(:name, :email)
User.select(:name, :email)
```

**Safe when used correctly (with placeholders):**
```ruby
User.where("name = ?", value)
User.where("name = :name", name: value)
User.having("count(*) > ?", count)
User.joins("INNER JOIN posts ON posts.user_id = users.id")  # Safe if no interpolation
```

**Dangerous (string interpolation):**
```ruby
User.where("name = '#{value}'")         # SQL injection
User.order("#{params[:sort]}")           # SQL injection
User.select("#{params[:fields]}")        # SQL injection
User.group("#{params[:group_by]}")       # SQL injection
User.find_by_sql("SELECT * FROM users WHERE name = '#{value}'")  # SQL injection
User.pluck(Arel.sql(params[:column]))    # SQL injection
connection.execute("UPDATE users SET name = '#{value}'")  # SQL injection
```

### LIKE Query Injection

Users can inject SQL wildcards (`%` matches any string, `_` matches any char):

```ruby
# Without sanitization — user enters "%" and gets ALL records
User.where("name LIKE ?", "%#{params[:q]}%")

# With sanitization — wildcards in user input are escaped
User.where("name LIKE ?", "%#{User.sanitize_sql_like(params[:q])}%")
```

### Order Clause Injection

```ruby
# DANGEROUS — attacker can inject: "name; DROP TABLE users--"
Post.order(params[:sort])

# SAFE — permit-list approach
ALLOWED_SORT = {
  "newest" => "created_at DESC",
  "oldest" => "created_at ASC",
  "title"  => "title ASC",
  "popular" => "views_count DESC"
}.freeze

Post.order(ALLOWED_SORT.fetch(params[:sort], "created_at DESC"))
```

### Arel.sql

`Arel.sql()` tells ActiveRecord "trust this string as raw SQL." Only use with strings you fully control:

```ruby
# SAFE — literal string
Post.order(Arel.sql("COALESCE(published_at, created_at) DESC"))

# DANGEROUS — user input
Post.order(Arel.sql(params[:order_clause]))
```

---

## Mass Assignment Deep Dive

### Strong Parameters Patterns

```ruby
# Basic permit
def user_params
  params.require(:user).permit(:name, :email, :avatar)
end

# Array values
def post_params
  params.require(:post).permit(:title, :body, tag_ids: [])
end

# Nested attributes
def order_params
  params.require(:order).permit(
    :customer_name,
    line_items_attributes: [:id, :product_id, :quantity, :_destroy]
  )
end

# Nested hash (JSON column or serialized attribute)
def settings_params
  params.require(:user).permit(
    settings: [:theme, :locale, :notifications_enabled]
  )
end
```

### Dangerous Attributes to Never Permit from Users

```ruby
# These should NEVER appear in user-facing strong params:
# :role, :admin, :is_admin, :superadmin, :verified, :email_verified,
# :approved, :banned, :suspended, :credits, :balance,
# :password_digest (use :password and :password_confirmation instead)
# :created_at, :updated_at (set by Rails)
# :id (set by database)

# WRONG
params.require(:user).permit(:name, :email, :role, :admin)

# CORRECT — admin sets role through a separate admin-only controller
# app/controllers/admin/users_controller.rb
def user_params
  params.require(:user).permit(:name, :email, :role)  # Admin-only controller
end

# app/controllers/users_controller.rb (user-facing)
def user_params
  params.require(:user).permit(:name, :email, :bio)  # No role!
end
```

### params.permit! — The Nuclear Option

```ruby
# NEVER use this
params.permit!  # Permits EVERYTHING — defeats the entire purpose
User.create(params.permit!)  # Mass assignment on steroids

# If you think you need permit!, you're wrong. List every attribute explicitly.
```

---

## Session Security Deep Dive

### Session Fixation Attack

1. Attacker gets a valid session ID from your app
2. Tricks victim into using that session ID
3. Victim logs in — session is now authenticated
4. Attacker uses the same session ID — they're now logged in as victim

**Defense: Always reset the session on login.**

```ruby
def create
  if user = User.authenticate_by(email_address: params[:email], password: params[:password])
    reset_session  # Invalidates old session, creates new one
    session[:user_id] = user.id
    redirect_to root_path
  end
end
```

### Session Storage

```ruby
# Default: CookieStore (encrypted, 4KB limit)
# Good for: session IDs, CSRF tokens, flash messages
# Bad for: large data, data that needs server-side invalidation

# For server-side sessions (better control):
# gem "redis-session-store"
Rails.application.config.session_store :redis_session_store,
  key: "_myapp_session",
  expire_after: 1.day
```

### Session Data Rules

```ruby
# GOOD — store only IDs
session[:user_id] = user.id
session[:cart_id] = cart.id

# BAD — store actual data (replay attacks, size limits)
session[:cart_items] = [{ product: "Widget", price: 9.99 }]
session[:user_role] = "admin"  # Attacker can replay old cookie with admin role
```

### Cookie Security Flags

```ruby
# In production, Rails sets these when force_ssl = true:
# Secure: true (only sent over HTTPS)
# HttpOnly: true (not accessible to JavaScript — prevents XSS cookie theft)
# SameSite: Lax (prevents CSRF from most cross-site contexts)

# You can set custom cookies securely:
cookies.encrypted[:preference] = {
  value: "dark_mode",
  httponly: true,
  secure: Rails.env.production?,
  same_site: :lax,
  expires: 1.year
}
```

---

## Authentication Patterns

### Rails 8 Authentication Generator

```bash
bin/rails generate authentication
bin/rails db:migrate
```

This creates:
- `User` model with `has_secure_password`
- `Session` model for tracking active sessions
- `SessionsController` for login/logout
- `PasswordsController` for reset flow
- `Authentication` concern for `ApplicationController`

### Timing-Safe Authentication

```ruby
# WRONG — timing attack reveals which field is wrong
user = User.find_by(email: params[:email])
if user && user.authenticate(params[:password])
  # ...
end
# If email doesn't exist, responds faster (no bcrypt comparison)

# CORRECT — authenticate_by is timing-safe
if user = User.authenticate_by(email_address: params[:email], password: params[:password])
  # authenticate_by always performs bcrypt comparison, even if email not found
end
```

### Password Validation

```ruby
class User < ApplicationRecord
  has_secure_password

  # has_secure_password gives you:
  # - password presence on create
  # - password <= 72 bytes
  # - password_confirmation match

  # Add your own:
  validates :password, length: { minimum: 12 }, if: :password_digest_changed?
  # Consider checking against breached password databases (have_i_been_pwned gem)
end
```

### Multi-Device Session Management

```ruby
# The authentication generator creates a Session model
# Users can have multiple active sessions (one per device)

class Session < ApplicationRecord
  belongs_to :user

  # Expire old sessions
  def self.sweep(time = 30.days)
    where(updated_at: ...time.ago).delete_all
  end
end

# Logout from all devices
Current.user.sessions.delete_all
```

---

## Authorization Patterns

### Scoping Queries (Most Important)

```ruby
# CRITICAL — never use unscoped finds for user-owned resources
# WRONG
@project = Project.find(params[:id])

# CORRECT
@project = Current.user.projects.find(params[:id])

# For admin viewing others' resources, use policy objects
@project = authorize(Project.find(params[:id]))
```

### before_action for Access Control

```ruby
class ApplicationController < ActionController::Base
  before_action :require_authentication

  private

  def require_authentication
    redirect_to new_session_path unless authenticated?
  end
end

# Admin check
class Admin::BaseController < ApplicationController
  before_action :require_admin

  private

  def require_admin
    head :forbidden unless Current.user&.admin?
  end
end
```

### Pundit Pattern

```ruby
class PostPolicy
  attr_reader :user, :post

  def initialize(user, post)
    @user = user
    @post = post
  end

  def update?
    post.user == user || user.admin?
  end

  def destroy?
    post.user == user || user.admin?
  end

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      if @user.admin?
        @scope.all
      else
        @scope.where(user: @user)
      end
    end
  end
end
```

---

## Content Security Policy Deep Dive

### Starter Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
  policy.connect_src :self, :https
  policy.frame_ancestors :none
  policy.base_uri    :self
  policy.form_action :self
end
```

### Nonce-Based CSP (Recommended over 'unsafe-inline')

```ruby
# Generate nonce per request
Rails.application.config.content_security_policy_nonce_generator =
  ->(request) { SecureRandom.base64(16) }

# Apply to script-src (and optionally style-src)
Rails.application.config.content_security_policy_nonce_directives = %w[script-src]
```

```erb
<!-- Tags automatically get nonce attribute -->
<%= javascript_include_tag "application", nonce: true %>
<%= javascript_tag nonce: true do %>
  console.log("This is allowed by CSP");
<% end %>
```

### Per-Controller Overrides

```ruby
class PaymentsController < ApplicationController
  content_security_policy do |policy|
    policy.script_src :self, "https://js.stripe.com"
    policy.frame_src "https://js.stripe.com"
  end
end
```

### Report-Only Mode for Migration

```ruby
# Start with report-only to find violations without breaking your app
Rails.application.config.content_security_policy_report_only = true
```

---

## Redirect Security

### url_from (Rails 7+)

```ruby
# url_from returns nil if the URL doesn't point to your app
redirect_to url_from(params[:return_to]) || root_path

# It checks: same host, same port, same protocol
url_from("https://myapp.com/dashboard")    # => "https://myapp.com/dashboard"
url_from("https://evil.com/phishing")      # => nil
url_from("javascript:alert(1)")            # => nil
```

### Manual Validation

```ruby
def safe_redirect_path(path)
  return root_path if path.blank?

  uri = URI.parse(path.to_s)
  # Only allow relative paths starting with /
  if uri.relative? && uri.path.start_with?("/") && !uri.path.start_with?("//")
    uri.path
  else
    root_path
  end
rescue URI::InvalidURIError
  root_path
end
```

### Open Redirect via Host Parameter

```ruby
# DANGEROUS — Rails redirect_to with hash can be exploited
redirect_to params.update(action: "show")
# Attacker adds ?host=evil.com → redirects to evil.com

# SAFE — never pass raw params to redirect_to with update/merge
redirect_to action: "show"
```

---

## File Upload Security

### Active Storage Validations

```ruby
class User < ApplicationRecord
  has_one_attached :avatar

  validates :avatar,
    content_type: ["image/png", "image/jpeg", "image/webp"],
    size: { less_than: 5.megabytes }
end

class Document < ApplicationRecord
  has_one_attached :file

  validates :file,
    content_type: {
      in: ["application/pdf", "text/plain"],
      message: "must be a PDF or text file"
    },
    size: { less_than: 25.megabytes }
end
```

### Serving User Uploads Safely

```ruby
# Active Storage default: redirect mode (safe — serves from storage service URL)
config.active_storage.resolve_model_to_route = :rails_storage_redirect

# AVOID proxy mode for user uploads on your main domain
# It serves files through your app — potential for stored XSS with HTML files
config.active_storage.resolve_model_to_route = :rails_storage_proxy

# BEST: Serve from a different domain/CDN to isolate cookies
# Configure your storage service to serve from a separate origin
```

### Filename Sanitization

Active Storage handles filename sanitization, but if you're doing manual file handling:

```ruby
# Active Storage does this internally
ActiveStorage::Filename.new(user_provided_name).sanitized

# Manual sanitization
def sanitize_filename(name)
  name.strip.gsub(/[^\w.\-]/, "_").gsub(/\.{2,}/, ".")
end
```

### Path Traversal Prevention

```ruby
# WRONG — path traversal: "../../../etc/passwd"
send_file("/uploads/#{params[:filename]}")

# CORRECT — verify the file is in the expected directory
basename = Rails.root.join("uploads").to_s
filepath = File.expand_path(File.join(basename, params[:filename]))
raise SecurityError unless filepath.start_with?(basename)
send_file(filepath)

# BEST — use database IDs, not filenames
document = Current.user.documents.find(params[:id])
send_file(document.file.path)
```

---

## Command Injection

### system() and Friends

```ruby
# DANGEROUS — shell injection via semicolons, pipes, backticks
system("convert #{params[:filename]} output.png")
# Attacker: "image.png; rm -rf /"

# SAFE — pass arguments as array (no shell interpretation)
system("convert", params[:filename], "output.png")

# DANGEROUS
`echo #{user_input}`
exec("ls #{user_input}")

# SAFE
IO.popen(["echo", user_input]) { |io| io.read }
```

### Kernel#open

```ruby
# DANGEROUS — Kernel#open executes commands with pipe prefix
open("| rm -rf /")  # Executes the command!
open(params[:url])   # If user sends "| malicious_command"

# SAFE alternatives
File.open(path)
URI.open(url)       # For URLs (requires 'open-uri')
IO.open(fd)
```

---

## CORS Configuration

```ruby
# Only needed if your frontend is on a different origin

# Gemfile
gem "rack-cors"

# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins "https://app.example.com"  # Be specific!

    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head],
      credentials: true,
      max_age: 86400
  end
end

# WRONG — allows any origin (defeats same-origin policy)
origins "*"  # NEVER use with credentials: true
```

---

## Rate Limiting

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create
end

class PasswordsController < ApplicationController
  rate_limit to: 5, within: 1.minute, only: :create
end

class Api::BaseController < ApplicationController
  rate_limit to: 100, within: 1.minute
end
```

---

## Security Scanning Tools

### Brakeman (Static Analysis)

```bash
# Install
gem install brakeman

# Run
brakeman -p /path/to/rails/app

# CI-friendly output
brakeman -p . -f json -o brakeman-report.json

# Check specific files
brakeman -p . --only-files app/controllers/
```

### bundle-audit (Dependency Vulnerabilities)

```bash
# Install
gem install bundler-audit

# Update vulnerability database
bundle-audit update

# Check Gemfile.lock
bundle-audit check

# CI: fail on vulnerabilities
bundle-audit check --update
```

### importmap:audit (JavaScript Dependencies)

```bash
# Rails 7+ with importmaps
bin/rails importmap:audit
```

---

## Regex Security in Ruby

```ruby
# Ruby's ^ and $ match LINE boundaries, not string boundaries
# This is a common security bug in format validators

# VULNERABLE
/^https?:\/\/[^\n]+$/
# Bypassed by: "javascript:alert(1)\nhttp://legit.com"

# SECURE
/\Ahttps?:\/\/[^\n]+\z/

# Rails validates_format_of raises if you use ^ or $
# unless you set multiline: true (acknowledging the risk)
validates :url, format: { with: /\Ahttps:\/\//, message: "must be HTTPS" }
```

---

## DNS Rebinding Protection

```ruby
# config/environments/production.rb
Rails.application.config.hosts << "myapp.com"
Rails.application.config.hosts << ".myapp.com"  # Allows subdomains

# Exclude health checks
Rails.application.config.host_authorization = {
  exclude: ->(request) { request.path.include?("/healthcheck") }
}
```

---

## Logging and Error Handling

### Filter Sensitive Parameters

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt,
  :certificate, :otp, :ssn, :credit_card, :cvv,
  :authorization, :api_key
]
```

### Don't Leak Stack Traces

```ruby
# config/environments/production.rb
config.consider_all_requests_local = false  # Default — don't change!
# Shows generic error page, not stack traces

# Custom error pages
# app/views/errors/internal_server_error.html.erb
# app/views/errors/not_found.html.erb
```

### Rescue Handlers

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionPolicy::Unauthorized, with: :forbidden

  private

  def not_found
    render "errors/not_found", status: :not_found
  end

  def forbidden
    render "errors/forbidden", status: :forbidden
  end
end

# NEVER expose detailed errors to users:
# WRONG
rescue => e
  render json: { error: e.message, backtrace: e.backtrace }

# CORRECT
rescue => e
  Rails.logger.error("Payment failed: #{e.message}")
  render json: { error: "Payment processing failed. Please try again." }, status: 500
end
```

---

## Credentials Management

### Editing Credentials

```bash
# Default credentials (all environments)
EDITOR=vim bin/rails credentials:edit

# Environment-specific
EDITOR=vim bin/rails credentials:edit --environment production
EDITOR=vim bin/rails credentials:edit --environment staging
```

### Structure

```yaml
# config/credentials.yml.enc (decrypted view)
secret_key_base: abc123...
aws:
  access_key_id: AKIAXXXXXXXX
  secret_access_key: xxxxxxxx
  bucket: my-app-production
stripe:
  publishable_key: pk_live_xxxx
  secret_key: sk_live_xxxx
sendgrid:
  api_key: SG.xxxx
```

### Access in Code

```ruby
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.dig(:stripe, :secret_key)

# Bang version — raises if missing
Rails.application.credentials.stripe![:secret_key]
```

### .gitignore Rules

```gitignore
# MUST be in .gitignore
config/master.key
config/credentials/*.key
```

---

## Quick Security Wins

1. **`force_ssl = true`** in production — one line, massive protection
2. **`config.filter_parameters`** — add all sensitive field names
3. **Run `brakeman`** — catches most static vulnerabilities
4. **Run `bundle-audit`** — catches known CVEs in gems
5. **Use `url_from`** for redirects — eliminates open redirects
6. **Use hash conditions** in `where()` — eliminates most SQL injection
7. **Never call `.html_safe`** on anything touching user input
8. **Add CSP headers** — even a basic policy helps
9. **Rate limit auth endpoints** — one line per controller
10. **Scope queries to current user** — prevents most authorization bugs
