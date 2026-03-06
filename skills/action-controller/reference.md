# Action Controller Reference

Detailed patterns, examples, and edge cases for Rails Action Controller. Companion to SKILL.md.

## Table of Contents

- [Strong Parameters Deep Dive](#strong-parameters-deep-dive)
- [Callback Ordering and Edge Cases](#callback-ordering-and-edge-cases)
- [Rendering — Full Options](#rendering--full-options)
- [Redirect Patterns](#redirect-patterns)
- [Flash Message Patterns](#flash-message-patterns)
- [Sessions — Configuration and Stores](#sessions--configuration-and-stores)
- [Cookies — All Jar Types](#cookies--all-jar-types)
- [rescue_from Patterns](#rescue_from-patterns)
- [CSRF Protection Details](#csrf-protection-details)
- [HTTP Authentication](#http-authentication)
- [Streaming and Downloads](#streaming-and-downloads)
- [Request and Response Objects](#request-and-response-objects)
- [Log Filtering](#log-filtering)
- [Force SSL](#force-ssl)
- [Content Security Policy](#content-security-policy)
- [Browser Version Control](#browser-version-control)
- [Health Check Endpoint](#health-check-endpoint)
- [API Controllers](#api-controllers)

---

## Strong Parameters Deep Dive

### expect vs require + permit

Rails 8 introduced `expect` which combines `require` and `permit` into one call. Prefer `expect`.

```ruby
# Rails 8+ (preferred)
params.expect(article: [:title, :body])

# Equivalent older style
params.require(:article).permit(:title, :body)
```

`expect` raises `ActionController::ParameterMissing` (returns 400) if the root key is missing.

### Scalar values from params

```ruby
# Extract a single scalar (e.g., from URL params)
id = params.expect(:id)    # raises if :id missing

# Old style
id = params.require(:id)
```

### All Permitting Patterns

#### Simple flat attributes
```ruby
params.expect(user: [:name, :email, :age, :active])
```

#### Array of scalars (tag_ids, category_ids, etc.)
```ruby
# { article: { title: "Hi", tag_ids: [1, 2, 3] } }
params.expect(article: [:title, tag_ids: []])
```

#### Single nested hash (belongs_to / has_one style)
```ruby
# { user: { name: "Jo", profile: { bio: "Hi", website: "example.com" } } }
params.expect(user: [:name, profile: [:bio, :website]])
```

#### Deeply nested hashes
```ruby
# { company: { name: "Acme", address: { street: "123", geo: { lat: 1.0, lng: 2.0 } } } }
params.expect(company: [:name, address: [:street, :city, geo: [:lat, :lng]]])
```

#### Array of hashes (has_many nested attributes) — DOUBLE ARRAY SYNTAX
```ruby
# { project: { name: "X", tasks_attributes: [{ title: "A" }, { title: "B" }] } }
params.expect(project: [:name, tasks_attributes: [[:title, :done, :id, :_destroy]]])
```

The `[[...]]` (double array) is critical. Single `[...]` means "array of scalars." Double `[[...]]` means "array of hashes with these permitted keys."

#### Hash with integer keys (nested attributes from forms)
```ruby
# Forms send: { book: { chapters_attributes: { "0" => { title: "Ch1" }, "1" => { title: "Ch2" } } } }
# Same syntax — Rails treats integer-keyed hashes like arrays
params.expect(book: [:title, chapters_attributes: [[:title, :id, :_destroy]]])
```

#### Arbitrary/dynamic hash
```ruby
# When keys are truly unpredictable (settings, metadata, etc.)
params.expect(product: [:name, settings: {}])
# ⚠️ settings: {} permits ANY scalar values inside. Use only when necessary.
```

#### Multiple file uploads
```ruby
# { post: { title: "Hi", images: [<file1>, <file2>] } }
params.expect(post: [:title, images: []])
```

#### fetch with defaults (for optional root keys)
```ruby
# Useful in #new where the root key might not exist yet
params.fetch(:blog, {}).permit(:title, :author)
```

#### accepts_nested_attributes_for — complete example
```ruby
# Model:
class Project < ApplicationRecord
  has_many :tasks
  accepts_nested_attributes_for :tasks, allow_destroy: true
end

# Controller:
def project_params
  params.expect(project: [
    :name,
    :description,
    tasks_attributes: [[:title, :completed, :position, :id, :_destroy]]
  ])
end
```

Always include `:id` and `:_destroy` for nested attributes if `allow_destroy: true`.

### Multiple return values from expect

```ruby
name, emails, friends = params.expect(
  :name,
  emails: [],
  friends: [[:name, family: [:name], hobbies: []]]
)
```

### Composite key parameters

```ruby
# URL: /books/4_2
id = params.extract_value(:id)  # => ["4", "2"]
@book = Book.find(id)
```

---

## Callback Ordering and Edge Cases

### Execution order

Callbacks run in declaration order, top to bottom:

```ruby
class PostsController < ApplicationController
  before_action :first    # runs 1st
  before_action :second   # runs 2nd
  before_action :third    # runs 3rd
end
```

Inherited callbacks from parent classes run first, then the child's.

### Halting the chain

A `before_action` that calls `render` or `redirect_to` halts all remaining callbacks AND the action:

```ruby
before_action :require_admin

private
  def require_admin
    redirect_to root_path, alert: "Nope" unless current_user.admin?
    # If redirect happens, the action and remaining callbacks are skipped
  end
```

### only / except

```ruby
before_action :authenticate, except: [:index, :show]
before_action :set_record, only: [:show, :edit, :update, :destroy]
```

### skip_before_action

```ruby
class ApplicationController < ActionController::Base
  before_action :authenticate
end

class PublicController < ApplicationController
  skip_before_action :authenticate, only: [:index, :show]
end
```

`skip_before_action` raises if the callback doesn't exist. Use `raise: false` if it might not be defined:

```ruby
skip_before_action :some_callback, raise: false
```

### around_action

```ruby
around_action :wrap_in_monitoring

private
  def wrap_in_monitoring
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    yield  # REQUIRED — executes the action
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start
    Rails.logger.info "[Perf] #{controller_name}##{action_name}: #{duration.round(3)}s"
  end
```

`around_action` code after `yield` runs even if the action raises (if you use `ensure`). `after_action` does NOT run on exceptions.

### after_action

```ruby
after_action :track_response, only: [:create, :update]

private
  def track_response
    # Has access to response object
    Analytics.track(action: action_name, status: response.status)
  end
```

### Callback classes

```ruby
class AuthenticationCallback
  def self.before(controller)
    unless controller.send(:current_user)
      controller.redirect_to controller.login_path
    end
  end
end

class SecureController < ApplicationController
  before_action AuthenticationCallback
end
```

The class must implement a class method matching the callback type (`before`, `after`, `around`).

### Block callbacks

```ruby
before_action do |controller|
  redirect_to root_path unless controller.send(:current_user)&.admin?
end
```

---

## Rendering — Full Options

### Implicit rendering

If an action doesn't call `render` or `redirect_to`, Rails renders `app/views/<controller>/<action>.html.erb` automatically.

### Explicit render options

```ruby
render :new                              # template for action :new in current controller
render "articles/new"                    # cross-controller template
render template: "articles/new"          # explicit template option
render plain: "OK"                       # plain text
render html: "<h1>Hi</h1>".html_safe     # raw HTML
render json: @post                       # calls .to_json
render json: { error: "bad" }, status: 422
render xml: @post                        # calls .to_xml
render js: "alert('hi')"                 # JavaScript
render body: "raw"                       # raw body
render file: Rails.root.join("public/404.html"), layout: false
render inline: "<%= 'hi' %>"             # inline ERB (avoid in production code)
render nothing: true                     # deprecated — use head
```

### head — no body

```ruby
head :ok                  # 200
head :no_content          # 204
head :created, location: post_url(@post)
head :not_found           # 404
```

### Status codes

Use symbols, not numbers:

| Symbol | Code |
|--------|------|
| `:ok` | 200 |
| `:created` | 201 |
| `:no_content` | 204 |
| `:moved_permanently` | 301 |
| `:found` | 302 |
| `:see_other` | 303 |
| `:not_modified` | 304 |
| `:bad_request` | 400 |
| `:unauthorized` | 401 |
| `:forbidden` | 403 |
| `:not_found` | 404 |
| `:unprocessable_entity` | 422 |
| `:too_many_requests` | 429 |
| `:internal_server_error` | 500 |

### Layout control

```ruby
class AdminController < ApplicationController
  layout "admin"  # uses app/views/layouts/admin.html.erb
end

# Per-action layout
def show
  render layout: "minimal"
end

# Conditional layout
layout :determine_layout

private
  def determine_layout
    current_user&.admin? ? "admin" : "application"
  end
```

### respond_to for format negotiation

```ruby
def show
  @post = Post.find(params.expect(:id))
  respond_to do |format|
    format.html
    format.json { render json: @post }
    format.csv { send_data @post.to_csv, filename: "post.csv" }
    format.pdf { render pdf: generate_pdf(@post) }
  end
end
```

Register custom MIME types in `config/initializers/mime_types.rb`:

```ruby
Mime::Type.register "application/rtf", :rtf
```

### Variant-based rendering

```ruby
# Set variant in before_action
before_action :set_variant

def set_variant
  request.variant = :mobile if request.user_agent.match?(/Mobile|Android|iPhone/)
  request.variant = :tablet if request.user_agent.match?(/iPad/)
end

# In action
def show
  respond_to do |format|
    format.html do |html|
      html.mobile   # renders show.html+mobile.erb
      html.tablet   # renders show.html+tablet.erb
      html.none     # renders show.html.erb (default)
    end
  end
end
```

Template naming: `show.html+mobile.erb`, `show.html+tablet.erb`.

### Double render protection

```ruby
# ❌ This raises AbstractController::DoubleRenderError
def show
  render :one
  render :two
end

# ✅ Use early return
def show
  if condition
    render :special and return
  end
  render :normal
end

# ✅ Or use if/else
def show
  if condition
    render :special
  else
    render :normal
  end
end
```

A controller action can only render or redirect ONCE.

---

## Redirect Patterns

```ruby
redirect_to action: :show, id: 5
redirect_to @post                                  # polymorphic URL
redirect_to posts_url                              # named route
redirect_to "https://example.com"                  # external URL
redirect_to posts_path, notice: "Done!"            # with flash
redirect_to posts_path, alert: "Error!"
redirect_to posts_path, flash: { custom: "hi" }   # custom flash key
redirect_to posts_path, status: :see_other         # 303 for DELETE
redirect_to posts_path, allow_other_host: true      # allow external redirect

redirect_back fallback_location: root_path         # go back or fallback
redirect_back_or_to root_path                       # Rails 7.1+ shorthand
```

### After DELETE — always use :see_other

```ruby
def destroy
  @post.destroy!
  redirect_to posts_path, notice: "Deleted.", status: :see_other
end
```

Without `:see_other` (303), the browser may replay the DELETE request on redirect.

---

## Flash Message Patterns

### Types

```ruby
flash[:notice]  # success messages (green)
flash[:alert]   # error/warning messages (red)
flash[:custom]  # any custom key you want
```

### When to use flash vs flash.now

| Situation | Use | Why |
|-----------|-----|-----|
| Before `redirect_to` | `flash[:notice]` | Flash persists to next request |
| Before `render` | `flash.now[:error]` | No redirect = same request; regular flash would leak to NEXT request |
| Multiple redirects | `flash.keep` | Carry flash through an extra redirect |

### flash.keep

```ruby
def index
  flash.keep  # carries ALL flash values through this redirect
  redirect_to dashboard_path
end

# Or keep specific key
flash.keep(:notice)
```

### Layout display

```erb
<!-- app/views/layouts/application.html.erb -->
<% flash.each do |type, message| %>
  <div class="flash flash-<%= type %>">
    <%= message %>
  </div>
<% end %>
```

---

## Sessions — Configuration and Stores

### Available stores

| Store | Description | Pros | Cons |
|-------|-------------|------|------|
| `CookieStore` (default) | All data in the cookie | Zero setup, no server state | 4KB limit, all data sent every request |
| `CacheStore` | Uses Rails.cache | Leverages existing cache infra | Sessions lost on cache eviction |
| `ActiveRecordStore` | Database storage | Unlimited size, persistent | Requires gem + migration, slower |

### Configuration

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  expire_after: 12.hours

# Cache store
Rails.application.config.session_store :cache_store,
  key: "_myapp_session",
  expire_after: 30.minutes

# Domain sharing
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  domain: ".example.com"  # share session across subdomains
```

### reset_session — ALWAYS on login

```ruby
def create
  user = User.authenticate_by(email: params[:email], password: params[:password])
  if user
    reset_session  # prevents session fixation attacks
    session[:current_user_id] = user.id
    redirect_to root_path
  else
    flash.now[:alert] = "Invalid credentials"
    render :new, status: :unprocessable_entity
  end
end
```

---

## Cookies — All Jar Types

| Jar | Method | Tamper-proof? | Encrypted? | Use case |
|-----|--------|---------------|------------|----------|
| Basic | `cookies[:key]` | No | No | Non-sensitive preferences |
| Permanent | `cookies.permanent[:key]` | No | No | Long-lived preferences |
| Signed | `cookies.signed[:key]` | Yes | No | User IDs, tokens (readable but can't be faked) |
| Encrypted | `cookies.encrypted[:key]` | Yes | Yes | Sensitive data |

### Chaining jars

```ruby
cookies.permanent.signed[:user_id] = current_user.id   # permanent + signed
cookies.permanent.encrypted[:token] = session_token      # permanent + encrypted
```

### Options

```ruby
cookies[:theme] = {
  value: "dark",
  expires: 1.year,
  path: "/admin",       # only sent for /admin paths
  domain: ".example.com", # shared across subdomains
  secure: true,          # HTTPS only
  httponly: true,         # not accessible via JavaScript
  same_site: :lax        # CSRF protection (:strict, :lax, :none)
}
```

### Serialization

Default serializer is `:json`. Be aware that `Date`, `Time`, and `Symbol` are serialized as strings:

```ruby
cookies.encrypted[:date] = Date.today
cookies.encrypted[:date]  # => "2024-03-20" (String, not Date!)
```

---

## rescue_from Patterns

### Order matters — most specific first

```ruby
class ApplicationController < ActionController::Base
  # Most specific first
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActiveRecord::RecordInvalid, with: :unprocessable
  rescue_from ActionController::ParameterMissing, with: :bad_request
  rescue_from Pundit::NotAuthorizedError, with: :forbidden

  # ⚠️ NEVER do this — breaks Rails internals:
  # rescue_from Exception, with: :handle_all
  # rescue_from StandardError, with: :handle_all
end
```

### Handler receives the exception

```ruby
rescue_from ActiveRecord::RecordInvalid, with: :unprocessable

private
  def unprocessable(exception)
    render json: {
      error: "Validation failed",
      details: exception.record.errors.full_messages
    }, status: :unprocessable_entity
  end
```

### Block syntax

```ruby
rescue_from ActiveRecord::RecordNotFound do |exception|
  render json: { error: exception.message }, status: :not_found
end
```

### Format-aware handlers

```ruby
def not_found
  respond_to do |format|
    format.html { render "errors/not_found", status: :not_found }
    format.json { render json: { error: "Not found" }, status: :not_found }
    format.any  { head :not_found }
  end
end
```

### Custom exception classes

```ruby
# app/errors/authorization_error.rb
class AuthorizationError < StandardError; end

# In controller
class ApplicationController < ActionController::Base
  rescue_from AuthorizationError do
    redirect_back fallback_location: root_path, alert: "Access denied."
  end
end

# Raise it
def update
  raise AuthorizationError unless current_user.can_edit?(@post)
  # ...
end
```

---

## CSRF Protection Details

### How it works

1. Rails generates an authenticity token for each session
2. The token is embedded in forms (via `form_with`) and in a `<meta>` tag
3. On non-GET requests, Rails verifies the submitted token matches the session's token
4. Mismatch → `ActionController::InvalidAuthenticityToken` (or session reset)

### Configuration

```ruby
class ApplicationController < ActionController::Base
  # Default — raises exception
  protect_from_forgery with: :exception

  # Alternative — just resets session (less disruptive)
  protect_from_forgery with: :reset_session

  # Null session — useful for APIs that may receive cookies
  protect_from_forgery with: :null_session
end
```

### Skipping for APIs

```ruby
# Option 1: Skip in specific controller
class Api::V1::BaseController < ApplicationController
  skip_forgery_protection
end

# Option 2: Inherit from API base (better)
class Api::V1::BaseController < ActionController::API
  # No CSRF protection included by default
end
```

### Custom AJAX with CSRF

If you're making fetch/XHR calls that don't use Turbo or rails-ujs, include the token manually:

```javascript
// Read from meta tag
const token = document.querySelector('meta[name="csrf-token"]').content;

fetch('/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': token
  },
  body: JSON.stringify({ post: { title: 'Hi' } })
});
```

### Authenticity token in meta tag

```erb
<!-- In layout head -->
<%= csrf_meta_tags %>
<!-- Renders: -->
<!-- <meta name="csrf-token" content="..."> -->
<!-- <meta name="csrf-param" content="authenticity_token"> -->
```

---

## HTTP Authentication

### Basic Auth

```ruby
# Class-level (simplest)
class AdminController < ApplicationController
  http_basic_authenticate_with name: "admin", password: ENV["ADMIN_PW"]
end

# Method-level (more control)
class AdminController < ApplicationController
  before_action :authenticate

  private
    def authenticate
      authenticate_or_request_with_http_basic("Admin") do |name, password|
        ActiveSupport::SecurityUtils.secure_compare(name, "admin") &
          ActiveSupport::SecurityUtils.secure_compare(password, ENV["ADMIN_PW"])
      end
    end
end
```

Always use `secure_compare` to prevent timing attacks.

### Digest Auth

```ruby
class AdminController < ApplicationController
  USERS = { "admin" => Digest::MD5.hexdigest("admin:realm:password") }

  before_action :authenticate

  private
    def authenticate
      authenticate_or_request_with_http_digest("Application") do |username|
        USERS[username]
      end
    end
end
```

### Token Auth

```ruby
class Api::BaseController < ActionController::API
  before_action :authenticate

  private
    def authenticate
      authenticate_or_request_with_http_token do |token, _options|
        @current_user = User.find_by(api_token: token)
      end
    end
end
```

Client sends: `Authorization: Token token="abc123"`

---

## Streaming and Downloads

### send_data — generated content

```ruby
send_data content,
  filename: "report.csv",
  type: "text/csv",
  disposition: "attachment"  # "attachment" (download) or "inline" (display in browser)
```

### send_file — existing file on disk

```ruby
send_file path_to_file,
  filename: "download.pdf",
  type: "application/pdf",
  disposition: "attachment",
  stream: true,              # default: true, streams in 4KB chunks
  buffer_size: 4096          # chunk size
```

⚠️ Never use user input directly in `send_file` paths — directory traversal risk.

### Live streaming (SSE — Server-Sent Events)

```ruby
class EventsController < ApplicationController
  include ActionController::Live

  def stream
    response.headers["Content-Type"] = "text/event-stream"
    response.headers["Cache-Control"] = "no-cache"
    response.headers["X-Accel-Buffering"] = "no"  # disable Nginx buffering

    sse = SSE.new(response.stream, event: "message")

    10.times do |i|
      sse.write({ count: i })
      sleep 1
    end
  rescue ActionController::Live::ClientDisconnected
    # Client disconnected, that's fine
  ensure
    response.stream.close  # ALWAYS close
  end
end
```

**Important streaming caveats:**
- Each stream creates a new thread — don't create too many
- WEBrick buffers everything — use Puma for streaming
- Always `ensure` the stream is closed
- Headers must be set BEFORE first `write`

---

## Request and Response Objects

### Request properties

```ruby
request.method              # "GET", "POST", "PUT", "PATCH", "DELETE"
request.get?                # true/false (also post?, put?, patch?, delete?, head?)
request.xhr?                # AJAX request?
request.format              # Mime::HTML, Mime::JSON, etc.
request.content_type        # "application/json", etc.

# URLs
request.url                 # "https://example.com/posts?page=2"
request.original_url        # same, but before any rewrites
request.host                # "example.com"
request.domain              # "example.com"
request.port                # 443
request.protocol            # "https://"
request.path                # "/posts"
request.query_string        # "page=2"

# Client info
request.remote_ip           # client IP (respects X-Forwarded-For)
request.user_agent          # browser user agent string
request.headers["Accept"]   # any header

# Parameter sources
request.query_parameters    # from URL query string only
request.request_parameters  # from POST body only
request.path_parameters     # from route matching only
```

### Response properties

```ruby
response.body               # response body string
response.status             # numeric status code
response.headers            # header hash
response.content_type       # "text/html"
response.charset            # "utf-8"
response.location           # redirect URL, if any

# Set custom headers
response.headers["X-Request-Id"] = SecureRandom.uuid
response.headers["Cache-Control"] = "no-store"
```

---

## Log Filtering

### Filter sensitive params from logs

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt, :certificate, :otp, :ssn,
  :credit_card, :card_number
]
```

Uses partial matching — `:passw` filters `password`, `password_confirmation`, etc.

### Filter redirect URLs

```ruby
# config/application.rb
config.filter_redirect << "s3.amazonaws.com"
config.filter_redirect << /private_path/
```

---

## Force SSL

```ruby
# config/environments/production.rb
config.force_ssl = true
```

This enables the `ActionDispatch::SSL` middleware which:
- Redirects HTTP to HTTPS
- Sets `Strict-Transport-Security` header
- Marks cookies as secure

---

## Content Security Policy

Rails has built-in CSP support:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src :self, :https
    policy.font_src    :self, :https, :data
    policy.img_src     :self, :https, :data
    policy.object_src  :none
    policy.script_src  :self, :https
    policy.style_src   :self, :https, :unsafe_inline

    # Nonce for inline scripts (works with Turbo)
    policy.script_src  :self, :https, :strict_dynamic
  end

  # Generate nonce for script tags
  config.content_security_policy_nonce_generator = ->(request) {
    request.session.id.to_s
  }
  config.content_security_policy_nonce_directives = %w[script-src style-src]
end
```

Per-controller override:

```ruby
class AdminController < ApplicationController
  content_security_policy do |policy|
    policy.script_src :self, :unsafe_eval  # needed for some admin tools
  end
end

# Disable for specific actions
content_security_policy false, only: :legacy_page
```

---

## Browser Version Control

Rails 7.2+ includes `allow_browser` for blocking outdated browsers:

```ruby
class ApplicationController < ActionController::Base
  # Block anything older than Safari 17.2, Chrome 120, Firefox 121, Opera 106
  allow_browser versions: :modern
end

# Custom versions
class ApplicationController < ActionController::Base
  allow_browser versions: { safari: 16.4, firefox: 121, ie: false }
  # ie: false blocks ALL IE versions
  # Unlisted browsers (Chrome, Opera) are allowed
end

# Per-action
class MessagesController < ApplicationController
  allow_browser versions: { opera: 104, chrome: 119 }, only: :show
end
```

Blocked browsers get `public/406-unsupported-browser.html` with 406 status.

---

## Health Check Endpoint

Built into Rails — available at `/up` by default:
- Returns **200** if app booted successfully
- Returns **500** if an exception occurred during boot

### Customize the path

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get "health" => "rails/health#show", as: :rails_health_check
end
```

### Custom health check with dependency checks

```ruby
class HealthController < ApplicationController
  skip_before_action :authenticate_user!

  def show
    checks = {
      database: database_connected?,
      redis: redis_connected?,
      migrations: migrations_current?
    }

    status = checks.values.all? ? :ok : :service_unavailable
    render json: checks, status: status
  end

  private
    def database_connected?
      ActiveRecord::Base.connection.active?
    rescue
      false
    end

    def redis_connected?
      Redis.current.ping == "PONG"
    rescue
      false
    end

    def migrations_current?
      !ActiveRecord::Base.connection.migration_context.needs_migration?
    rescue
      false
    end
end
```

---

## API Controllers

### Inheriting from ActionController::API

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  # No CSRF protection (APIs use token auth)
  # No cookie/session middleware
  # No flash
  # No asset helpers

  before_action :authenticate_token

  private
    def authenticate_token
      authenticate_or_request_with_http_token do |token, _options|
        @current_user = User.find_by(api_token: token)
      end
    end
end
```

`ActionController::API` is lighter than `ActionController::Base` — it excludes middleware for cookies, sessions, flash, CSRF, and views.

### Wrap Parameters (automatic JSON wrapping)

When `Content-Type: application/json`, Rails can auto-wrap params under the controller's model name:

```ruby
# POST /api/users with body: { "name": "Jo", "email": "jo@ex.com" }
# params becomes: { "name" => "Jo", "email" => "jo@ex.com", "user" => { "name" => "Jo", "email" => "jo@ex.com" } }
```

This is enabled by default (`config.action_controller.wrap_parameters_by_default = true`). Disable with:

```ruby
config.action_controller.wrap_parameters_by_default = false
```

### Standard API response pattern

```ruby
class Api::V1::PostsController < Api::BaseController
  def index
    posts = Post.all
    render json: posts
  end

  def show
    post = Post.find(params.expect(:id))
    render json: post
  end

  def create
    post = Post.new(post_params)
    if post.save
      render json: post, status: :created, location: api_v1_post_url(post)
    else
      render json: { errors: post.errors.full_messages }, status: :unprocessable_entity
    end
  end

  def update
    post = Post.find(params.expect(:id))
    if post.update(post_params)
      render json: post
    else
      render json: { errors: post.errors.full_messages }, status: :unprocessable_entity
    end
  end

  def destroy
    post = Post.find(params.expect(:id))
    post.destroy!
    head :no_content
  end

  private
    def post_params
      params.expect(post: [:title, :body, :published])
    end
end
```

---

## default_url_options

Set global defaults for URL generation:

```ruby
class ApplicationController < ActionController::Base
  def default_url_options
    { locale: I18n.locale }
  end
end
```

Now all URL helpers include the locale:
```ruby
posts_path          # => "/posts?locale=en"
posts_path(locale: :fr)  # => "/posts?locale=fr"
```
