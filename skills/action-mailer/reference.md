# Action Mailer Reference

Detailed patterns, examples, and edge cases for Action Mailer in Rails 8.1.

## Table of Contents

- [Mailer Class Patterns](#mailer-class-patterns)
- [Template Patterns](#template-patterns)
- [Attachments Deep Dive](#attachments-deep-dive)
- [Delivery Configuration](#delivery-configuration)
- [Callbacks In Depth](#callbacks-in-depth)
- [Interceptors and Observers](#interceptors-and-observers)
- [Testing Patterns](#testing-patterns)
- [Preview Patterns](#preview-patterns)
- [I18n for Emails](#i18n-for-emails)
- [Layouts and Shared Partials](#layouts-and-shared-partials)
- [URLs and Assets in Emails](#urls-and-assets-in-emails)
- [Error Handling](#error-handling)
- [Edge Cases and Gotchas](#edge-cases-and-gotchas)
- [Production Recipes](#production-recipes)

---

## Mailer Class Patterns

### Basic Mailer

```ruby
class UserMailer < ApplicationMailer
  def welcome_email
    @user = params[:user]
    mail(to: @user.email, subject: "Welcome!")
  end
end
```

### Mailer with Multiple Actions

```ruby
class UserMailer < ApplicationMailer
  default from: "noreply@example.com"

  def welcome_email
    @user = params[:user]
    mail(to: @user.email, subject: "Welcome to Our App")
  end

  def password_reset
    @user = params[:user]
    @token = params[:token]
    mail(to: @user.email, subject: "Reset Your Password")
  end

  def account_confirmation
    @user = params[:user]
    @confirmation_url = confirm_account_url(token: params[:token])
    mail(to: @user.email, subject: "Confirm Your Account")
  end

  def goodbye_email
    @user = params[:user]
    mail(to: @user.email, subject: "We're sorry to see you go")
  end
end
```

### Parameterized Mailer with before_action

This is the preferred pattern for mailers with shared context across actions:

```ruby
class OrderMailer < ApplicationMailer
  before_action :set_order_and_customer

  default to: -> { @customer.email }

  def confirmation
    mail(subject: "Order ##{@order.number} confirmed")
  end

  def shipped
    @tracking_url = params[:tracking_url]
    mail(subject: "Order ##{@order.number} has shipped!")
  end

  def delivered
    mail(subject: "Order ##{@order.number} delivered")
  end

  def refunded
    @refund_amount = params[:refund_amount]
    mail(subject: "Refund for Order ##{@order.number}")
  end

  private

  def set_order_and_customer
    @order = params[:order]
    @customer = @order.customer
  end
end

# Calling:
OrderMailer.with(order: @order).confirmation.deliver_later
OrderMailer.with(order: @order, tracking_url: url).shipped.deliver_later
```

### Mailer with Dynamic Default Recipients

```ruby
class AdminMailer < ApplicationMailer
  default to: -> { Admin.pluck(:email) },
          from: "system@example.com"

  def new_registration(user)
    @user = user
    mail(subject: "New User Signup: #{@user.email}")
  end

  def daily_report
    @stats = params[:stats]
    mail(subject: "Daily Report — #{Date.current}")
  end
end
```

### Display Names in From/To

```ruby
class NotificationMailer < ApplicationMailer
  def notify
    @user = params[:user]
    mail(
      to: email_address_with_name(@user.email, @user.full_name),
      from: email_address_with_name("support@example.com", "MyApp Support"),
      reply_to: email_address_with_name("help@example.com", "MyApp Help Desk"),
      subject: "You have a new notification"
    )
  end
end
```

### Multiple Recipients, CC, BCC

```ruby
def team_update
  @team = params[:team]
  mail(
    to: @team.members.pluck(:email),
    cc: @team.managers.pluck(:email),
    bcc: ["analytics@example.com"],
    subject: "Team Update: #{@team.name}"
  )
end
```

---

## Template Patterns

### Standard HTML Template

```html+erb
<%# app/views/user_mailer/welcome_email.html.erb %>
<h1>Welcome, <%= @user.name %>!</h1>

<p>Thanks for joining. Here's what you can do next:</p>

<ul>
  <li><%= link_to "Complete your profile", edit_profile_url %></li>
  <li><%= link_to "Explore the dashboard", dashboard_url %></li>
  <li><%= link_to "Read the getting started guide", guide_url %></li>
</ul>

<p>
  If you have any questions, just reply to this email.
</p>

<p>
  — The <%= @company_name %> Team
</p>
```

### Standard Text Template

```erb
<%# app/views/user_mailer/welcome_email.text.erb %>
Welcome, <%= @user.name %>!

Thanks for joining. Here's what you can do next:

* Complete your profile: <%= edit_profile_url %>
* Explore the dashboard: <%= dashboard_url %>
* Read the getting started guide: <%= guide_url %>

If you have any questions, just reply to this email.

— The <%= @company_name %> Team
```

### Template with Conditional Content

```html+erb
<h1>Order #<%= @order.number %> Update</h1>

<% if @order.shipped? %>
  <p>Your order has shipped!</p>
  <p>Tracking number: <strong><%= @order.tracking_number %></strong></p>
  <p><%= link_to "Track your package", @tracking_url %></p>
<% elsif @order.confirmed? %>
  <p>Your order has been confirmed and is being prepared.</p>
<% end %>

<h2>Order Summary</h2>
<table>
  <% @order.line_items.each do |item| %>
    <tr>
      <td><%= item.name %></td>
      <td><%= number_to_currency(item.price) %></td>
    </tr>
  <% end %>
  <tr>
    <td><strong>Total</strong></td>
    <td><strong><%= number_to_currency(@order.total) %></strong></td>
  </tr>
</table>
```

### Using Partials in Mailer Views

```html+erb
<%# app/views/user_mailer/welcome_email.html.erb %>
<h1>Welcome!</h1>
<%= render "shared/mailer_header" %>
<p>Content here...</p>
<%= render partial: "order_summary", locals: { order: @order } %>
<%= render "shared/mailer_footer" %>
```

### Rendering Without Templates

```ruby
def simple_notification
  mail(
    to: params[:user].email,
    subject: "Quick Update",
    body: "Your request has been processed.",
    content_type: "text/plain"
  )
end
```

### Explicit Format Block

```ruby
def welcome_email
  @user = params[:user]
  mail(to: @user.email, subject: "Welcome") do |format|
    format.html { render "custom_welcome_template" }
    format.text { render plain: "Welcome, #{@user.name}!" }
  end
end
```

---

## Attachments Deep Dive

### Simple File Attachment

```ruby
def invoice_email
  @invoice = params[:invoice]
  attachments["invoice-#{@invoice.number}.pdf"] = @invoice.generate_pdf
  mail(to: @invoice.customer_email, subject: "Invoice ##{@invoice.number}")
end
```

### Attachment with Explicit MIME Type

```ruby
def report_email
  attachments["report.csv"] = {
    mime_type: "text/csv",
    content: generate_csv(@report_data)
  }
  mail(to: params[:user].email, subject: "Your Report")
end
```

### Multiple Attachments

```ruby
def document_bundle
  @documents = params[:documents]
  @documents.each do |doc|
    attachments[doc.filename] = doc.download
  end
  mail(to: params[:user].email, subject: "Your Documents")
end
```

### Inline Attachment (Image in Email Body)

```ruby
# In mailer
def branded_email
  attachments.inline["logo.png"] = File.read(
    Rails.root.join("app/assets/images/logo.png")
  )
  attachments.inline["banner.jpg"] = File.read(
    Rails.root.join("app/assets/images/email-banner.jpg")
  )
  mail(to: params[:user].email, subject: "Newsletter")
end
```

```html+erb
<%# In template %>
<div style="text-align: center;">
  <%= image_tag attachments["logo.png"].url, alt: "Company Logo", width: "150" %>
</div>

<%= image_tag attachments["banner.jpg"].url, alt: "Banner", style: "width: 100%;" %>

<p>Hello <%= @user.name %>,</p>
<p>Here's your weekly newsletter...</p>
```

### Active Storage Attachment

```ruby
def receipt_email
  @receipt = params[:receipt]
  if @receipt.pdf_file.attached?
    attachments["receipt.pdf"] = {
      mime_type: @receipt.pdf_file.content_type,
      content: @receipt.pdf_file.download
    }
  end
  mail(to: @receipt.user.email, subject: "Your Receipt")
end
```

---

## Delivery Configuration

### SMTP (Production)

```ruby
# config/environments/production.rb
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address:         "smtp.example.com",
  port:            587,
  domain:          "example.com",
  user_name:       Rails.application.credentials.dig(:smtp, :user_name),
  password:        Rails.application.credentials.dig(:smtp, :password),
  authentication:  "plain",
  enable_starttls: true,
  open_timeout:    5,
  read_timeout:    5
}
config.action_mailer.default_url_options = { host: "www.example.com", protocol: "https" }
config.action_mailer.asset_host = "https://www.example.com"
```

### Gmail SMTP

```ruby
config.action_mailer.smtp_settings = {
  address:         "smtp.gmail.com",
  port:            587,
  domain:          "example.com",
  user_name:       Rails.application.credentials.dig(:smtp, :user_name),
  password:        Rails.application.credentials.dig(:smtp, :password),
  authentication:  "plain",
  enable_starttls: true,
  open_timeout:    5,
  read_timeout:    5
}
```

### Sendmail

```ruby
config.action_mailer.delivery_method = :sendmail
config.action_mailer.sendmail_settings = {
  location:  "/usr/sbin/sendmail",
  arguments: %w[-i]
}
```

### Development — Letter Opener

```ruby
# Gemfile
gem "letter_opener", group: :development

# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```

### Test Environment

```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test
config.action_mailer.default_url_options = { host: "www.example.com" }
```

### Per-Email Delivery Overrides

```ruby
def welcome_email
  @user = params[:user]
  delivery_options = {
    user_name: params[:company].smtp_user,
    password:  params[:company].smtp_password,
    address:   params[:company].smtp_host
  }
  mail(
    to: @user.email,
    subject: "Welcome",
    delivery_method_options: delivery_options
  )
end
```

### Key Configuration Options

```ruby
# Raise errors if delivery fails (recommended in development)
config.action_mailer.raise_delivery_errors = true

# Actually send emails (set false to suppress all delivery)
config.action_mailer.perform_deliveries = true

# Enable caching in mailer views
config.action_mailer.perform_caching = true

# Queue name for deliver_later jobs
config.action_mailer.deliver_later_queue_name = :mailers

# Default from address (app-wide fallback)
config.action_mailer.default_options = { from: "noreply@example.com" }
```

---

## Callbacks In Depth

### Callback Order

When sending an email, callbacks fire in this order:

1. `before_action` — Set up instance vars, populate defaults
2. **Mailer action executes** — Your method runs
3. `after_action` — Modify message, set headers
4. `before_deliver` — Last chance to modify or abort
5. **Email delivered**
6. `after_deliver` — Log, record, notify

### before_action — Shared Setup

```ruby
class InvitationsMailer < ApplicationMailer
  before_action :set_inviter_and_invitee
  before_action { @account = params[:inviter].account }

  default to:       -> { @invitee.email },
          from:     -> { email_address_with_name("invites@example.com", @inviter.name) },
          reply_to: -> { @inviter.email }

  def account_invitation
    mail(subject: "#{@inviter.name} invited you to #{@account.name}")
  end

  private

  def set_inviter_and_invitee
    @inviter = params[:inviter]
    @invitee = params[:invitee]
  end
end
```

### after_action — Modify Before Delivery

```ruby
class UserMailer < ApplicationMailer
  after_action :set_delivery_options
  after_action :prevent_delivery_to_guests
  after_action :set_custom_headers

  def feedback_message
    @feedback = params[:feedback]
    @user = params[:user]
    mail(to: @user.email, subject: "Feedback received")
  end

  private

  def set_delivery_options
    if @business&.has_smtp_settings?
      mail.delivery_method.settings.merge!(@business.smtp_settings)
    end
  end

  def prevent_delivery_to_guests
    mail.perform_deliveries = false if @user&.guest?
  end

  def set_custom_headers
    headers["X-Mailer-Category"] = action_name
    headers["X-App-Version"] = Rails.application.config.version
  end
end
```

### before_deliver — Intercept/Abort

```ruby
class ApplicationMailer < ActionMailer::Base
  before_deliver :sandbox_staging

  private

  def sandbox_staging
    if Rails.env.staging?
      message.to = ["staging-inbox@example.com"]
      message.subject = "[STAGING] #{message.subject}"
    end
  end
end
```

Abort delivery with `throw :abort`:

```ruby
before_deliver :check_recipient_preferences

def check_recipient_preferences
  throw :abort if recipient_unsubscribed?
end
```

### after_deliver — Post-Delivery Actions

```ruby
class NotificationMailer < ApplicationMailer
  after_deliver :log_delivery
  after_deliver :update_notification_status

  def notify
    @notification = params[:notification]
    mail(to: @notification.user.email, subject: @notification.subject)
  end

  private

  def log_delivery
    EmailLog.create!(
      mailer: self.class.name,
      action: action_name,
      recipient: message.to.first,
      subject: message.subject,
      sent_at: Time.current
    )
  end

  def update_notification_status
    params[:notification].update!(emailed_at: Time.current)
  end
end
```

### Aborting on Body Set

Mailer callbacks abort further processing if `body` is set to a non-nil value. This means if a `before_action` sets the body, the mailer action won't run its template rendering.

---

## Interceptors and Observers

### Email Interceptor — Modify Before Sending

Interceptors run before the email is handed to the delivery agent:

```ruby
# app/interceptors/sandbox_email_interceptor.rb
class SandboxEmailInterceptor
  def self.delivering_email(message)
    message.to = ["sandbox@example.com"]
    message.cc = nil
    message.bcc = nil
    message.subject = "[SANDBOX] #{message.subject}"
  end
end
```

### Email Observer — React After Sending

Observers run after successful delivery:

```ruby
# app/observers/email_delivery_observer.rb
class EmailDeliveryObserver
  def self.delivered_email(message)
    EmailDelivery.create!(
      to: message.to&.join(", "),
      from: message.from&.join(", "),
      subject: message.subject,
      message_id: message.message_id,
      delivered_at: Time.current
    )
  end
end
```

### Registration

```ruby
# config/initializers/mail_interceptors.rb
Rails.application.configure do
  if Rails.env.staging?
    config.action_mailer.interceptors = %w[SandboxEmailInterceptor]
  end

  config.action_mailer.observers = %w[EmailDeliveryObserver]
end
```

### When to Use Callbacks vs Interceptors

| Feature | Callbacks | Interceptors/Observers |
|---------|-----------|----------------------|
| Scope | Per-mailer class | All mailers globally |
| Access to mailer context | Yes (params, instance vars) | No (only `Mail::Message`) |
| Can abort delivery | Yes (`throw :abort`) | Yes (set `perform_deliveries = false`) |
| Registration | In mailer class | In initializer |

**Use callbacks** when logic is specific to a mailer. **Use interceptors/observers** for global concerns (sandboxing, logging, analytics).

---

## Testing Patterns

### Basic Mailer Test

```ruby
require "test_helper"

class UserMailerTest < ActionMailer::TestCase
  test "welcome_email sends to user" do
    user = users(:active_user)
    email = UserMailer.with(user: user).welcome_email

    assert_emails 1 do
      email.deliver_now
    end

    assert_equal [user.email], email.to
    assert_equal ["notifications@example.com"], email.from
    assert_equal "Welcome to MyApp", email.subject
  end

  test "welcome_email includes user name in body" do
    user = users(:active_user)
    email = UserMailer.with(user: user).welcome_email

    assert_match user.name, email.html_part.body.to_s
    assert_match user.name, email.text_part.body.to_s
  end

  test "welcome_email has both HTML and text parts" do
    user = users(:active_user)
    email = UserMailer.with(user: user).welcome_email

    assert_not_nil email.html_part
    assert_not_nil email.text_part
  end
end
```

### Testing deliver_later (Enqueued Jobs)

```ruby
test "welcome_email is enqueued for later delivery" do
  user = users(:active_user)

  assert_enqueued_emails 1 do
    UserMailer.with(user: user).welcome_email.deliver_later
  end
end

test "no email enqueued for invalid user" do
  assert_no_enqueued_emails do
    # action that shouldn't enqueue
  end
end
```

### Testing Attachments

```ruby
test "invoice_email includes PDF attachment" do
  invoice = invoices(:paid_invoice)
  email = InvoiceMailer.with(invoice: invoice).invoice_email

  assert_equal 1, email.attachments.size
  attachment = email.attachments.first
  assert_equal "invoice-#{invoice.number}.pdf", attachment.filename
  assert_equal "application/pdf", attachment.content_type.split(";").first
end
```

### Testing with assert_emails

```ruby
# Exactly N emails sent
assert_emails 2 do
  UserMailer.with(user: user1).welcome_email.deliver_now
  UserMailer.with(user: user2).welcome_email.deliver_now
end

# No emails sent
assert_no_emails do
  # nothing happens
end

# Check deliveries array
UserMailer.with(user: user).welcome_email.deliver_now
email = ActionMailer::Base.deliveries.last
assert_equal "Welcome", email.subject
```

### Integration Test — Email from Controller Action

```ruby
require "test_helper"

class UsersControllerTest < ActionDispatch::IntegrationTest
  test "creating user sends welcome email" do
    assert_emails 1 do
      post users_path, params: {
        user: { name: "New User", email: "new@example.com", login: "newuser" }
      }
    end
  end

  test "creating user enqueues welcome email" do
    assert_enqueued_emails 1 do
      post users_path, params: {
        user: { name: "New User", email: "new@example.com", login: "newuser" }
      }
    end
  end
end
```

---

## Preview Patterns

### Basic Preview

```ruby
# test/mailers/previews/user_mailer_preview.rb
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    UserMailer.with(user: User.first).welcome_email
  end
end
```

### Preview with Fallback Data

```ruby
class UserMailerPreview < ActionMailer::Preview
  def welcome_email
    user = User.first || User.new(
      name: "Jane Doe",
      email: "jane@example.com",
      login: "janedoe"
    )
    UserMailer.with(user: user).welcome_email
  end
end
```

### Preview All Actions

```ruby
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    OrderMailer.with(order: sample_order).confirmation
  end

  def shipped
    OrderMailer.with(
      order: sample_order,
      tracking_url: "https://track.example.com/ABC123"
    ).shipped
  end

  def delivered
    OrderMailer.with(order: sample_order).delivered
  end

  private

  def sample_order
    Order.includes(:line_items, :customer).first || build_sample_order
  end

  def build_sample_order
    Order.new(
      number: "ORD-PREVIEW-001",
      customer: Customer.new(name: "Preview Customer", email: "preview@example.com")
    )
  end
end
```

### Custom Preview Path

```ruby
# config/application.rb
config.action_mailer.preview_paths << "#{Rails.root}/lib/mailer_previews"
```

### Viewing Previews

- **All previews:** `http://localhost:3000/rails/mailers`
- **Specific mailer:** `http://localhost:3000/rails/mailers/user_mailer`
- **Specific action:** `http://localhost:3000/rails/mailers/user_mailer/welcome_email`

Previews auto-reload when templates change. No server restart needed.

---

## I18n for Emails

### Subject Translation (Automatic)

When you omit `subject:` from `mail()`, Action Mailer looks up the translation automatically:

```ruby
def welcome_email
  @user = params[:user]
  mail(to: @user.email) # Subject auto-looked up from I18n
end
```

```yaml
# config/locales/en.yml
en:
  user_mailer:
    welcome_email:
      subject: "Welcome to Our App!"
```

### Subject with Interpolation

```yaml
en:
  user_mailer:
    welcome_email:
      subject: "Welcome to %{app_name}, %{user_name}!"
```

```ruby
def welcome_email
  @user = params[:user]
  mail(
    to: @user.email,
    subject: default_i18n_subject(app_name: "MyApp", user_name: @user.name)
  )
end
```

### Full I18n Templates

```yaml
# config/locales/en.yml
en:
  user_mailer:
    welcome_email:
      subject: "Welcome!"
      greeting: "Hello %{name},"
      body: "Thanks for joining our platform."
      cta: "Get Started"

# config/locales/es.yml
es:
  user_mailer:
    welcome_email:
      subject: "¡Bienvenido!"
      greeting: "Hola %{name},"
      body: "Gracias por unirte a nuestra plataforma."
      cta: "Comenzar"
```

```html+erb
<h1><%= t(".greeting", name: @user.name) %></h1>
<p><%= t(".body") %></p>
<p><%= link_to t(".cta"), @login_url %></p>
```

### Setting Locale per Email

```ruby
def welcome_email
  @user = params[:user]
  I18n.with_locale(@user.locale || :en) do
    mail(to: @user.email)
  end
end
```

---

## Layouts and Shared Partials

### Default Mailer Layout

```html+erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
      body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; line-height: 1.6; color: #333; }
      a { color: #0066cc; }
    </style>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```

```erb
<%# app/views/layouts/mailer.text.erb %>
<%= yield %>

--
Sent by MyApp | https://www.example.com
```

### Custom Layout per Mailer

```ruby
class MarketingMailer < ApplicationMailer
  layout "marketing_email"
end
```

### Custom Layout per Action

```ruby
def welcome_email
  mail(to: @user.email, subject: "Welcome") do |format|
    format.html { render layout: "branded_email" }
    format.text
  end
end
```

---

## URLs and Assets in Emails

### URL Configuration

```ruby
# config/environments/production.rb
config.action_mailer.default_url_options = { host: "www.example.com", protocol: "https" }
config.action_mailer.asset_host = "https://cdn.example.com"
```

### Always Use _url, Never _path

```erb
<%# CORRECT — absolute URL %>
<%= link_to "View", order_url(@order) %>
<%= edit_profile_url %>

<%# WRONG — relative path, will break %>
<%= link_to "View", order_path(@order) %>
```

### Images

```ruby
# Set asset host
config.action_mailer.asset_host = "https://www.example.com"
```

```html+erb
<%# Then use image_tag normally %>
<%= image_tag "logo.png", alt: "Logo" %>
```

### url_for in Emails

```erb
<%= url_for(controller: "welcome", action: "greeting", host: "example.com") %>
```

---

## Error Handling

### rescue_from in Mailers

```ruby
class ApplicationMailer < ActionMailer::Base
  rescue_from ActiveJob::DeserializationError do |exception|
    Rails.logger.warn("Mailer deserialization error: #{exception.message}")
    # Record was deleted between enqueue and delivery — skip silently
  end

  rescue_from Net::SMTPAuthenticationError do |exception|
    Rails.logger.error("SMTP auth failed: #{exception.message}")
    ErrorNotifier.notify(exception)
  end

  rescue_from Net::SMTPServerBusy do |exception|
    Rails.logger.warn("SMTP busy: #{exception.message}")
    # Active Job will retry automatically
    raise # Re-raise to trigger retry
  end
end
```

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ActionView::MissingTemplate` | No template file for mailer action | Create both `.html.erb` and `.text.erb` |
| `ArgumentError: missing host` | `default_url_options` not set | Add `config.action_mailer.default_url_options` |
| `Net::SMTPAuthenticationError` | Bad SMTP credentials | Check credentials in `rails credentials:edit` |
| `ActiveJob::DeserializationError` | Record deleted between enqueue and delivery | Add `rescue_from` in mailer |
| `NoMethodError` on `params` | Called mailer without `.with()` | Use `UserMailer.with(user: @user).action` |

---

## Edge Cases and Gotchas

### 1. params is nil Without .with()

```ruby
# This WILL blow up if you access params[:user]
UserMailer.welcome_email.deliver_later  # WRONG

# Always use .with()
UserMailer.with(user: @user).welcome_email.deliver_later  # CORRECT
```

### 2. deliver_later Serializes Arguments

Active Job serializes params for `deliver_later`. This means:
- ActiveRecord objects are serialized by class + ID and re-fetched at delivery time
- If the record is deleted between enqueue and delivery, you get `ActiveJob::DeserializationError`
- Non-serializable objects (Procs, IO) cannot be passed via params

```ruby
# WRONG — Proc can't be serialized
UserMailer.with(user: @user, callback: -> { do_something }).action.deliver_later

# CORRECT — Pass only serializable data
UserMailer.with(user: @user, callback_type: "signup").action.deliver_later
```

### 3. Template Lookup Order

Action Mailer looks for templates in:
1. `app/views/{mailer_name}/{action_name}.{format}.erb`
2. Custom `template_path` if specified
3. `prepend_view_path` directories

### 4. Multipart Email Part Order

The order of parts in multipart emails is determined by `:parts_order` in defaults:

```ruby
class ApplicationMailer < ActionMailer::Base
  default parts_order: ["text/plain", "text/html"]
end
```

### 5. Email with No Recipients is Silently Skipped

```ruby
mail(to: nil, subject: "Test")  # No error, no email sent
mail(to: [], subject: "Test")   # No error, no email sent
```

### 6. Headers Method vs mail() Options

```ruby
# Both work — use mail() options for standard headers
mail(to: "user@example.com", subject: "Hi")

# Use headers() for custom/non-standard headers
headers["X-Custom-Header"] = "value"
headers["List-Unsubscribe"] = "<mailto:unsubscribe@example.com>"
```

### 7. Mailer Method Names

Mailer method names don't need to end in `_email`. Any public method works:

```ruby
class UserMailer < ApplicationMailer
  def welcome        # Fine
  def welcome_email  # Also fine
  def notify         # Also fine
end
```

### 8. Caching in Mailer Views

```ruby
# Enable in environment config
config.action_mailer.perform_caching = true
```

```html+erb
<%# Then use cache helper in views %>
<% cache @user do %>
  <h1>Hello, <%= @user.name %></h1>
<% end %>
```

---

## Production Recipes

### Unsubscribe Header

```ruby
class MarketingMailer < ApplicationMailer
  after_action :add_unsubscribe_header

  private

  def add_unsubscribe_header
    headers["List-Unsubscribe"] = "<#{unsubscribe_url(token: @user.unsubscribe_token)}>"
    headers["List-Unsubscribe-Post"] = "List-Unsubscribe=One-Click"
  end
end
```

### Conditional Delivery (Preferences)

```ruby
class NotificationMailer < ApplicationMailer
  before_deliver :check_preferences

  def activity_update
    @user = params[:user]
    @activity = params[:activity]
    mail(to: @user.email, subject: "New activity")
  end

  private

  def check_preferences
    throw :abort unless params[:user].email_notifications_enabled?
  end
end
```

### Rate-Limited Delivery

```ruby
class DigestMailer < ApplicationMailer
  before_deliver :rate_limit

  def weekly_digest
    @user = params[:user]
    mail(to: @user.email, subject: "Your Weekly Digest")
  end

  private

  def rate_limit
    last_sent = EmailLog.where(
      mailer: self.class.name,
      action: action_name,
      recipient: params[:user].email
    ).maximum(:sent_at)

    throw :abort if last_sent && last_sent > 6.days.ago
  end
end
```

### Multi-Tenant SMTP

```ruby
class TenantMailer < ApplicationMailer
  after_action :set_tenant_smtp

  private

  def set_tenant_smtp
    tenant = params[:tenant]
    if tenant&.custom_smtp?
      mail.delivery_method.settings.merge!(
        address:   tenant.smtp_address,
        port:      tenant.smtp_port,
        user_name: tenant.smtp_username,
        password:  tenant.smtp_password
      )
    end
  end
end
```

### Premailer for Inline CSS

```ruby
# Gemfile
gem "premailer-rails"

# That's it — premailer-rails automatically inlines CSS from <style> tags
# and linked stylesheets in your mailer layouts before sending.
```
