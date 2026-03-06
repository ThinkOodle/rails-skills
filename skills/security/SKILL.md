---
name: security
description: Expert guidance for writing secure Rails applications. Use when dealing with security, CSRF protection, XSS prevention, SQL injection, authentication, authorization, sanitize, html_safe, credentials, secrets, content security policy, session security, mass assignment, strong parameters, secure headers, file uploads, open redirects, or vulnerability remediation. Covers every major attack vector and the Rails-idiomatic defenses.
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(bin/rails credentials:*), Bash(bin/rails secret), Bash(bundle audit), Bash(brakeman)
---

# Rails Security Expert

Write secure Rails code by default. Security mistakes are the most dangerous mistakes an agent can make — an XSS hole or SQL injection can compromise every user instantly. When in doubt, choose the safer path.

## Philosophy

**Core Principles:**
1. **Secure by default** — Rails has excellent built-in protections. Never disable them without explicit, documented justification
2. **Defense in depth** — Layer protections. Don't rely on a single mechanism
3. **Permit-list over deny-list** — Always prefer allowed lists over blocked lists for input validation, URL schemes, HTML tags, redirect targets
4. **Never trust user input** — Every param, header, cookie, and URL segment is attacker-controlled until proven otherwise
5. **Fail closed** — When authorization is ambiguous, deny access. When input is suspicious, reject it
6. **Minimize exposure** — Log less, expose less, store less. Filter sensitive params, avoid leaking stack traces, encrypt at rest

## When To Use This Skill

- Adding authentication or authorization logic
- Writing controllers that accept user input
- Rendering user-generated content in views
- Building API endpoints
- Working with file uploads or downloads
- Configuring sessions, cookies, or CSRF protection
- Managing secrets and credentials
- Setting HTTP security headers or CSP
- Reviewing code for security vulnerabilities
- Fixing reported CVEs or security issues

## Instructions

### Step 1: Audit the Existing Security Posture

**ALWAYS check what's already in place before changing anything:**

```bash
# Check CSRF configuration
rg "protect_from_forgery" app/controllers/

# Check strong params usage
rg "params\.permit\|params\.require" app/controllers/

# Check for dangerous patterns
rg "\.html_safe|raw\(|raw " app/views/ app/helpers/
rg "\.where\(\".*#\{" app/ --type ruby
rg "redirect_to.*params" app/controllers/

# Check security headers
rg "content_security_policy" config/
rg "force_ssl" config/

# Check filter_parameters
cat config/initializers/filter_parameter_logging.rb

# Check credentials setup
ls config/credentials* config/master.key 2>/dev/null
```

### Step 2: CSRF Protection

**Never disable CSRF protection.** Rails enables it by default. Respect it.

```ruby
# ApplicationController — this should ALREADY be here
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
end
```

**Rules:**
- Every non-GET request requires an authenticity token
- Include `<%= csrf_meta_tags %>` in your layout `<head>` for Turbo/Ajax
- API controllers using token auth may skip CSRF — but ONLY if they don't use cookie-based sessions
- Never use `skip_forgery_protection` on controllers that serve browser sessions

```ruby
# API-only controllers with token auth — acceptable to skip
class Api::BaseController < ActionController::API
  # No cookie session = no CSRF needed
  # But MUST authenticate via Authorization header
end

# WRONG — disabling CSRF on a browser-facing controller
class PaymentsController < ApplicationController
  skip_forgery_protection  # NEVER DO THIS
end
```

### Step 3: XSS Prevention

**Rails auto-escapes all ERB output by default. Never bypass it without justification.**

```ruby
# SAFE — auto-escaped (default)
<%= user.name %>
<%= user.bio %>

# DANGEROUS — bypasses escaping. Only use for trusted, sanitized HTML
<%= raw(content) %>              # AVOID
<%= content.html_safe %>         # AVOID
<%== content %>                  # AVOID

# CORRECT way to render trusted HTML — sanitize first
<%= sanitize(user.bio, tags: %w[p br strong em a ul ol li], attributes: %w[href title]) %>
```

**html_safe Rules:**
- NEVER call `.html_safe` on user input
- NEVER call `.html_safe` on strings built with interpolation of user data
- Only use `.html_safe` on string literals you fully control
- When building HTML in helpers, use `tag` builder or `content_tag` instead of string concat

```ruby
# WRONG — XSS vulnerability
def greeting(user)
  "<h1>Hello, #{user.name}</h1>".html_safe
end

# CORRECT — use tag helpers
def greeting(user)
  tag.h1("Hello, #{user.name}")
end

# CORRECT — escape manually if you must build strings
def greeting(user)
  "<h1>Hello, #{ERB::Util.html_escape(user.name)}</h1>".html_safe
end
```

**sanitize() usage:**
- Always specify allowed tags and attributes explicitly
- Default allowlist is reasonable but review it for your context
- For rich text, prefer Action Text which handles sanitization

### Step 4: SQL Injection Prevention

**Never interpolate user input into SQL strings.**

```ruby
# DANGEROUS — SQL injection
User.where("name = '#{params[:name]}'")
User.where("email LIKE '%#{params[:q]}%'")
Order.where("status = '#{status}' AND user_id = #{user_id}")

# SAFE — parameterized queries
User.where("name = ?", params[:name])
User.where("email LIKE ?", "%#{User.sanitize_sql_like(params[:q])}%")
Order.where(status: status, user_id: user_id)

# SAFE — named parameters
User.where("name = :name AND role = :role", name: params[:name], role: params[:role])

# SAFE — hash conditions (preferred)
User.where(name: params[:name])
```

**LIKE queries need special handling:**

```ruby
# WRONG — user can inject % and _ wildcards
User.where("name LIKE ?", "%#{params[:q]}%")

# CORRECT — sanitize LIKE wildcards
User.where("name LIKE ?", "%#{User.sanitize_sql_like(params[:q])}%")
```

**Also watch for injection in:**
- `order()` — never pass raw user input
- `pluck()` with string args
- `select()` with string args
- `group()`, `having()`, `joins()` with string args
- `find_by_sql()`, `connection.execute()`
- `Arel.sql()` — only use with trusted strings

```ruby
# WRONG — order injection
Post.order(params[:sort])

# CORRECT — permit-list of allowed columns
allowed = %w[created_at title updated_at]
direction = %w[asc desc].include?(params[:dir]) ? params[:dir] : "asc"
column = allowed.include?(params[:sort]) ? params[:sort] : "created_at"
Post.order("#{column} #{direction}")
```

### Step 5: Mass Assignment Protection

**Always use strong parameters. Never pass raw params to model methods.**

```ruby
# WRONG — mass assignment vulnerability
User.create(params[:user])
@user.update(params[:user])
User.new(params.permit!)  # permit! allows everything

# CORRECT — strong parameters
def user_params
  params.require(:user).permit(:name, :email, :bio)
end

@user.update(user_params)
```

**Strong params rules:**
- Define a private `*_params` method in every controller
- Only permit attributes the user should be able to set
- NEVER permit `:role`, `:admin`, `:is_admin`, `:verified` or similar privilege attributes from user input
- Nested attributes need explicit permitting

```ruby
def post_params
  params.require(:post).permit(
    :title, :body, :published,
    tags: [],
    images_attributes: [:id, :url, :caption, :_destroy]
  )
end
```

### Step 6: Authentication

**Use the Rails 8+ authentication generator as your starting point:**

```bash
bin/rails generate authentication
```

**Key authentication rules:**
- Use `has_secure_password` — it handles bcrypt hashing
- Use `authenticate_by` — it's timing-safe (prevents enumeration)
- Always `reset_session` after login to prevent session fixation
- Require current password for password/email changes
- Use generic error messages: "Invalid email or password" (not "User not found")

```ruby
# Session creation — correct pattern
class SessionsController < ApplicationController
  def create
    if user = User.authenticate_by(
      email_address: params[:email_address],
      password: params[:password]
    )
      reset_session  # Prevent session fixation
      start_new_session_for user
      redirect_to root_path
    else
      redirect_to new_session_path, alert: "Invalid email or password."
    end
  end
end
```

**Rate limiting login attempts:**

```ruby
class SessionsController < ApplicationController
  rate_limit to: 10, within: 3.minutes, only: :create
end
```

### Step 7: Authorization

**Always scope queries to the current user. Never trust IDs from params.**

```ruby
# WRONG — any user can access any project by changing the ID
@project = Project.find(params[:id])

# CORRECT — scope to current user
@project = Current.user.projects.find(params[:id])

# CORRECT — use policy objects (Pundit, Action Policy, etc.)
@project = authorize Project.find(params[:id])
```

**Authorization checklist:**
- Scope every query to the authenticated user's permissions
- Check authorization on every action (before_action)
- Use `before_action :authenticate` for all protected resources
- Prefer deny-by-default: use `before_action` with `except:` not `only:`

```ruby
class ApplicationController < ActionController::Base
  before_action :require_authentication
  # All controllers require auth unless explicitly skipped
end

class PublicPagesController < ApplicationController
  skip_before_action :require_authentication, only: [:home, :about]
end
```

### Step 8: Session Security

```ruby
# config/environments/production.rb
config.force_ssl = true  # HSTS + secure cookies + HTTPS redirect

# Session cookie settings (Rails defaults are good, but verify)
Rails.application.config.session_store :cookie_store,
  key: "_myapp_session",
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax
```

**Session rules:**
- Store only IDs in the session, never sensitive data
- Call `reset_session` after login (prevents session fixation)
- Implement session expiry (absolute + idle timeout)
- `force_ssl = true` in production — always

### Step 9: Secrets and Credentials

**Never hardcode secrets. Use Rails credentials or environment variables.**

```bash
# Edit credentials
EDITOR=vim bin/rails credentials:edit

# Environment-specific credentials
EDITOR=vim bin/rails credentials:edit --environment production
```

```ruby
# Access credentials
Rails.application.credentials.secret_key_base
Rails.application.credentials.dig(:aws, :access_key_id)

# Use bang version to fail loud if missing
Rails.application.credentials.some_api_key!
```

**Rules:**
- NEVER commit `config/master.key` or `config/credentials/*.key`
- Add `config/master.key` and `config/credentials/*.key` to `.gitignore`
- Use `ENV["RAILS_MASTER_KEY"]` in production
- Never log or expose credentials in error messages
- Rotate secrets if they may have been exposed

### Step 10: Content Security Policy

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
  policy.report_uri "/csp-violation-report-endpoint"
end

# Use nonces instead of 'unsafe-inline'
Rails.application.config.content_security_policy_nonce_generator =
  ->(request) { SecureRandom.base64(16) }
Rails.application.config.content_security_policy_nonce_directives = %w[script-src]
```

**Layout must include:**
```erb
<head>
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= javascript_include_tag "application", nonce: true %>
  <%= stylesheet_link_tag "application", nonce: true %>
</head>
```

### Step 11: Logging Security

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :passw, :secret, :token, :_key, :crypt, :salt, :certificate, :otp,
  :ssn, :credit_card, :card_number, :cvv, :authorization
]
```

**Never log:**
- Passwords, tokens, API keys, secrets
- Credit card numbers, SSNs, PII
- Full request bodies that may contain sensitive data

### Step 12: Redirect Security

```ruby
# WRONG — open redirect vulnerability
redirect_to params[:return_to]

# CORRECT — only allow internal paths
redirect_to safe_redirect_path(params[:return_to])

# Helper implementation
def safe_redirect_path(path)
  uri = URI.parse(path.to_s)
  if uri.relative? && uri.path.start_with?("/")
    uri.path
  else
    root_path
  end
rescue URI::InvalidURIError
  root_path
end

# SIMPLEST — use url_from (Rails 7+)
redirect_to url_from(params[:return_to]) || root_path
```

### Step 13: File Upload Security

```ruby
# Validate content type
class Document < ApplicationRecord
  has_one_attached :file

  validates :file, content_type: %w[application/pdf image/png image/jpeg],
                   size: { less_than: 10.megabytes }
end

# NEVER serve uploads from your app domain in production
# Use a separate domain/CDN or Active Storage's redirect mode

# NEVER use user-provided filenames for storage
# Active Storage handles this — it uses random keys
```

**File upload rules:**
- Validate content type AND file extension
- Set maximum file size limits
- Store uploads outside the web root (Active Storage does this)
- Never execute uploaded files
- Process images/media asynchronously
- Scan for malware in sensitive applications

## Quick Reference

### Dangerous Patterns to Grep For

```bash
# XSS: rg "\.html_safe|raw\(|<%==" app/
# SQL injection: rg '\.where\(".*#\{' app/ --type ruby
# Order injection: rg '\.order\(params' app/ --type ruby
# Mass assignment: rg 'params\.permit!' app/controllers/
# Open redirects: rg 'redirect_to.*params' app/controllers/
# CSRF disabled: rg 'skip_forgery_protection' app/controllers/
# Hardcoded secrets: rg '(api_key|secret|password|token)\s*=' app/ config/ -i
# Command injection: rg 'system\(|exec\(|`.*#\{' app/ --type ruby
```

### Regex Gotcha

```ruby
# WRONG — ^ and $ match LINE boundaries in Ruby, not string boundaries
validates :url, format: { with: /^https?:\/\/.+$/ }

# CORRECT — use \A and \z for string boundaries
validates :url, format: { with: /\Ahttps?:\/\/.+\z/ }
```

## Anti-Patterns to NEVER Introduce

1. **`html_safe` on user input** — Instant XSS. No exceptions
2. **String interpolation in SQL** — Use `?` placeholders or hash conditions
3. **`params.permit!`** — Permits everything, defeats strong params entirely
4. **`skip_forgery_protection`** on browser controllers — Opens CSRF attacks
5. **`redirect_to params[:url]`** — Open redirect. Validate with `url_from` or permit-list
6. **Hardcoded secrets in source** — Use credentials or ENV vars
7. **`Kernel#open` with user input** — Use `File.open`, `URI.open`, or `IO.open`
8. **Disabling `force_ssl` in production** — Cookies become stealable
9. **Logging sensitive data** — Filter parameters properly
10. **Trusting client-side validation** — Always validate server-side

## Security Audit Checklist

Before shipping any feature, verify:

- [ ] CSRF protection enabled (not skipped)
- [ ] Strong parameters used in every controller action
- [ ] No `.html_safe` or `raw()` on user-supplied content
- [ ] All SQL uses parameterized queries
- [ ] Queries scoped to current user (authorization)
- [ ] Redirects validated (no open redirects)
- [ ] Sensitive params filtered from logs
- [ ] File uploads validated (type, size)
- [ ] No hardcoded secrets in source
- [ ] `force_ssl = true` in production
- [ ] CSP headers configured
- [ ] Authentication uses timing-safe comparison
- [ ] Session reset after login
- [ ] Rate limiting on login/sensitive endpoints
