# Action Cable Reference

Detailed patterns, examples, and edge cases for Action Cable in Rails.

## Table of Contents

- [Connection Authentication Patterns](#connection-authentication-patterns)
- [Channel Patterns](#channel-patterns)
- [Broadcasting Patterns](#broadcasting-patterns)
- [Client-Side Patterns](#client-side-patterns)
- [Solid Cable Configuration](#solid-cable-configuration)
- [Redis Adapter Configuration](#redis-adapter-configuration)
- [PostgreSQL Adapter](#postgresql-adapter)
- [Testing Patterns](#testing-patterns)
- [Deployment](#deployment)
- [Turbo Streams vs Action Cable Decision Matrix](#turbo-streams-vs-action-cable-decision-matrix)
- [Common Patterns](#common-patterns)
- [Error Handling](#error-handling)
- [Performance](#performance)
- [Security](#security)

---

## Connection Authentication Patterns

### Devise (Cookie-Based)

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private
      def find_verified_user
        if verified_user = env["warden"].user
          verified_user
        else
          reject_unauthorized_connection
        end
      end
  end
end
```

### Session-Based (Standard Rails Auth)

```ruby
def find_verified_user
  if user_id = cookies.encrypted["_session"]["user_id"]
    User.find_by(id: user_id)
  else
    reject_unauthorized_connection
  end
end
```

### Token-Based (API/SPA)

```ruby
# Server
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private
      def find_verified_user
        token = request.params[:token]
        user = User.find_by_auth_token(token)
        user || reject_unauthorized_connection
      end
  end
end
```

```js
// Client — pass token via URL
import { createConsumer } from "@rails/actioncable"

function getWebSocketURL() {
  const token = document.querySelector('meta[name="auth-token"]')?.content
  return `/cable?token=${token}`
}

export default createConsumer(getWebSocketURL)
```

### JWT Token Auth

```ruby
def find_verified_user
  token = request.params[:token]
  payload = JWT.decode(token, Rails.application.credentials.secret_key_base).first
  User.find(payload["user_id"])
rescue JWT::DecodeError, ActiveRecord::RecordNotFound
  reject_unauthorized_connection
end
```

### Multiple Identifiers

```ruby
class Connection < ActionCable::Connection::Base
  identified_by :current_user, :current_account

  def connect
    self.current_user = find_verified_user
    self.current_account = current_user.account
  end
end
```

Both identifiers are available in all channels and can be used to target disconnections:

```ruby
ActionCable.server.remote_connections
  .where(current_user: user, current_account: account)
  .disconnect
```

---

## Channel Patterns

### Basic Channel with Parameter Validation

```ruby
class ChatChannel < ApplicationCable::Channel
  def subscribed
    room = Room.find_by(id: params[:room_id])

    if room && room.accessible_by?(current_user)
      stream_for room
    else
      reject
    end
  end

  def unsubscribed
    # Clean up: remove from online users, release locks, etc.
  end

  def speak(data)
    room = Room.find(params[:room_id])
    message = room.messages.create!(
      user: current_user,
      body: data["body"]
    )
    # Broadcasting happens in model callback or job
  end
end
```

**Always validate params and authorize access in `subscribed`.** Never trust client-supplied params without verification.

### Channel with Callbacks

```ruby
class GameChannel < ApplicationCable::Channel
  before_subscribe :verify_game_access
  after_subscribe :announce_player_joined

  def subscribed
    @game = Game.find(params[:game_id])
    stream_for @game
  end

  def unsubscribed
    announce_player_left
  end

  private
    def verify_game_access
      game = Game.find_by(id: params[:game_id])
      reject unless game&.player?(current_user)
    end

    def announce_player_joined
      GameChannel.broadcast_to(@game, {
        type: "player_joined",
        player: current_user.display_name
      })
    end

    def announce_player_left
      game = Game.find_by(id: params[:game_id])
      return unless game

      GameChannel.broadcast_to(game, {
        type: "player_left",
        player: current_user.display_name
      })
    end
end
```

### Connection Callbacks

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    before_command :set_current_attributes
    after_command :clear_current_attributes

    def connect
      self.current_user = find_verified_user
    end

    private
      def set_current_attributes
        Current.user = current_user
      end

      def clear_current_attributes
        Current.reset
      end

      def find_verified_user
        User.find_by(id: cookies.encrypted[:user_id]) ||
          reject_unauthorized_connection
      end
  end
end
```

### Multiple Streams Per Channel

```ruby
class DashboardChannel < ApplicationCable::Channel
  def subscribed
    stream_from "dashboard_global"
    stream_from "dashboard_team_#{current_user.team_id}"
    stream_for current_user  # Personal notifications too
  end
end
```

### Dynamic Subscribe/Unsubscribe

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
    # Start with no streams — client will call follow/unfollow
  end

  def follow(data)
    room = Room.find(data["room_id"])
    stream_for room if room.accessible_by?(current_user)
  end

  def unfollow(data)
    room = Room.find(data["room_id"])
    stop_stream_for room
  end
end
```

---

## Broadcasting Patterns

### From Models (with Background Job)

```ruby
class Message < ApplicationRecord
  belongs_to :room
  belongs_to :user

  after_create_commit :broadcast_message

  private
    def broadcast_message
      MessageBroadcastJob.perform_later(self)
    end
end
```

```ruby
class MessageBroadcastJob < ApplicationJob
  queue_as :default

  def perform(message)
    html = ApplicationController.renderer.render(
      partial: "messages/message",
      locals: { message: message }
    )

    ChatChannel.broadcast_to(message.room, {
      type: "new_message",
      html: html,
      message_id: message.id,
      sent_by: message.user.display_name
    })
  end
end
```

### From Controllers

```ruby
class MessagesController < ApplicationController
  def create
    @message = @room.messages.create!(message_params.merge(user: current_user))

    # Broadcast inline (simple cases)
    ActionCable.server.broadcast("chat_#{@room.id}", {
      html: render_to_string(partial: "messages/message", locals: { message: @message })
    })

    head :created
  end
end
```

### Targeted Broadcasts

```ruby
# Broadcast to a specific user
NotificationsChannel.broadcast_to(user, {
  title: "New follower",
  body: "#{follower.name} started following you"
})

# Broadcast to all users in a team
team.users.each do |user|
  NotificationsChannel.broadcast_to(user, { ... })
end

# Better: use a team stream
ActionCable.server.broadcast("team_#{team.id}", { ... })
```

### Conditional Broadcasting

```ruby
class Comment < ApplicationRecord
  after_create_commit :broadcast_if_public

  private
    def broadcast_if_public
      return unless post.public?

      CommentsChannel.broadcast_to(post, {
        html: ApplicationController.renderer.render(
          partial: "comments/comment",
          locals: { comment: self }
        )
      })
    end
end
```

---

## Client-Side Patterns

### Stimulus Controller Integration

This is the modern Rails way — pair Action Cable subscriptions with Stimulus controllers.

```js
// app/javascript/controllers/notifications_controller.js
import { Controller } from "@hotwired/stimulus"
import consumer from "../channels/consumer"

export default class extends Controller {
  static targets = ["badge", "list"]

  connect() {
    this.subscription = consumer.subscriptions.create("NotificationsChannel", {
      received: this.handleNotification.bind(this)
    })
  }

  disconnect() {
    this.subscription?.unsubscribe()
  }

  handleNotification(data) {
    // Update badge count
    if (this.hasBadgeTarget) {
      const count = parseInt(this.badgeTarget.textContent || "0") + 1
      this.badgeTarget.textContent = count
      this.badgeTarget.classList.remove("hidden")
    }

    // Prepend notification to list
    if (this.hasListTarget) {
      this.listTarget.insertAdjacentHTML("afterbegin", data.html)
    }
  }
}
```

```erb
<!-- app/views/layouts/application.html.erb -->
<div data-controller="notifications">
  <span data-notifications-target="badge" class="hidden">0</span>
  <div data-notifications-target="list"></div>
</div>
```

### Reconnection Handling

Action Cable handles reconnection automatically with exponential backoff. Handle the UI:

```js
consumer.subscriptions.create("ChatChannel", {
  connected() {
    this.showStatus("connected")
    // Re-fetch missed messages on reconnect
    this.fetchMissedMessages()
  },

  disconnected() {
    this.showStatus("disconnected")
  },

  rejected() {
    this.showStatus("unauthorized")
    // Redirect to login
    window.location.href = "/login"
  },

  showStatus(status) {
    const el = document.getElementById("connection-status")
    if (el) {
      el.dataset.status = status
      el.textContent = status === "connected" ? "●" : "○"
    }
  },

  fetchMissedMessages() {
    const lastId = document.querySelector("[data-message-id]:last-child")?.dataset.messageId
    if (lastId) {
      fetch(`/messages?after=${lastId}`)
        .then(r => r.json())
        .then(messages => messages.forEach(m => this.received(m)))
    }
  }
})
```

### Multiple Subscriptions

```js
// Subscribe to multiple rooms
const rooms = [1, 2, 3]
const subscriptions = rooms.map(roomId =>
  consumer.subscriptions.create(
    { channel: "ChatChannel", room_id: roomId },
    {
      received(data) {
        document.querySelector(`#room-${roomId}-messages`)
          .insertAdjacentHTML("beforeend", data.html)
      }
    }
  )
)

// Unsubscribe from all
subscriptions.forEach(sub => sub.unsubscribe())
```

---

## Solid Cable Configuration

Solid Cable is the Rails 8 default — a database-backed adapter that eliminates Redis dependency.

### Installation

```bash
bin/rails solid_cable:install
```

This generates:
- Updates `config/cable.yml`
- Creates `db/cable_schema.rb`
- You must manually update `config/database.yml`

### Database Configuration

```yaml
# config/database.yml
production:
  primary:
    <<: *default
    database: myapp_production
  cable:
    <<: *default
    database: myapp_production_cable
    migrations_paths: db/cable_migrate
```

```yaml
# config/cable.yml
production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

### Solid Cable Options

| Option | Default | Description |
|--------|---------|-------------|
| `polling_interval` | `0.1.seconds` | How often to poll for new messages |
| `message_retention` | `nil` | How long to keep messages (set to prevent table bloat) |
| `connects_to` | — | Database connection config |
| `silence_polling` | `true` | Suppress polling query logs |

### When to Use Solid Cable vs Redis

**Solid Cable wins when:**
- You want fewer infrastructure dependencies
- Message volume is moderate (<1000/sec)
- You're already running a database
- Simplicity matters more than absolute lowest latency

**Redis wins when:**
- Very high message throughput (thousands/sec)
- Sub-millisecond latency matters
- You already have Redis infrastructure
- You're running multiple app servers that need shared state beyond just pub/sub

---

## Redis Adapter Configuration

### Basic Setup

```yaml
# config/cable.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL", "redis://localhost:6379/1") %>
  channel_prefix: myapp_production
```

### With SSL/TLS

```yaml
production:
  adapter: redis
  url: rediss://redis.example.com:6380
  channel_prefix: myapp_production
  ssl_params:
    ca_file: "/path/to/ca.crt"
```

### With Sentinel

```yaml
production:
  adapter: redis
  url: redis://mymaster
  sentinels:
    - host: sentinel1.example.com
      port: 26379
    - host: sentinel2.example.com
      port: 26379
  channel_prefix: myapp_production
```

### Channel Prefix

**Always set `channel_prefix` in production** to avoid collisions when multiple apps share a Redis instance.

---

## PostgreSQL Adapter

Uses PostgreSQL's `NOTIFY`/`LISTEN` mechanism. No additional dependencies needed.

```yaml
production:
  adapter: postgresql
```

**Limitations:**
- 8KB payload limit per notification (PostgreSQL constraint)
- Uses the application's database connection pool
- Not suitable for very high-throughput scenarios

**When to use:** You're already on PostgreSQL, message payloads are small, and you don't want to add Redis or set up Solid Cable.

---

## Testing Patterns

### Channel Tests

```ruby
require "test_helper"

class ChatChannelTest < ActionCable::Channel::TestCase
  setup do
    @user = users(:active_user)
    @room = rooms(:general)
    # Stub the connection's current_user
    stub_connection current_user: @user
  end

  # Subscription tests
  test "subscribes successfully with valid room" do
    subscribe room_id: @room.id
    assert subscription.confirmed?
  end

  test "rejects subscription without room" do
    subscribe room_id: nil
    assert subscription.rejected?
  end

  test "rejects subscription for inaccessible room" do
    private_room = rooms(:private_room)
    subscribe room_id: private_room.id
    assert subscription.rejected?
  end

  # Stream tests
  test "streams from room" do
    subscribe room_id: @room.id
    assert_has_stream_for @room
  end

  test "streams with string name" do
    subscribe room_id: @room.id
    assert_has_stream "chat_#{@room.id}"
  end

  # Action tests
  test "speak creates a message" do
    subscribe room_id: @room.id
    assert_difference "Message.count", 1 do
      perform :speak, body: "Hello, world!"
    end
  end

  test "speak broadcasts to room" do
    subscribe room_id: @room.id

    assert_broadcasts("chat_#{@room.id}", 1) do
      perform :speak, body: "Hello!"
    end
  end

  # Unsubscribe tests
  test "marks user offline on unsubscribe" do
    subscribe room_id: @room.id
    assert @user.reload.online?

    unsubscribe
    refute @user.reload.online?
  end
end
```

### Connection Tests

```ruby
require "test_helper"

class ApplicationCable::ConnectionTest < ActionCable::Connection::TestCase
  setup do
    @user = users(:active_user)
  end

  test "connects with authenticated user" do
    connect cookies: { user_id: @user.id }
    assert_equal @user, connection.current_user
  end

  test "rejects connection without authentication" do
    assert_reject_connection { connect }
  end

  test "rejects connection with invalid user" do
    assert_reject_connection do
      connect cookies: { user_id: "nonexistent" }
    end
  end

  # Token-based auth test
  test "connects with valid token" do
    token = @user.generate_auth_token
    connect params: { token: token }
    assert_equal @user, connection.current_user
  end
end
```

### Broadcast Assertions in Integration Tests

```ruby
class MessagesIntegrationTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:active_user)
    @room = rooms(:general)
    sign_in @user
  end

  test "creating a message broadcasts to the room channel" do
    assert_broadcasts("chat_#{@room.id}", 1) do
      post room_messages_path(@room),
        params: { message: { body: "Hello!" } }
    end
  end

  test "broadcast contains expected data" do
    assert_broadcast_on("chat_#{@room.id}",
      hash_including("type" => "new_message")
    ) do
      post room_messages_path(@room),
        params: { message: { body: "Hello!" } }
    end
  end
end
```

### System Tests with Action Cable

```ruby
class ChatSystemTest < ApplicationSystemTestCase
  setup do
    @user = users(:active_user)
    @room = rooms(:general)
    sign_in @user
  end

  test "receives messages in real-time" do
    visit room_path(@room)

    # Simulate another user sending a message
    Message.create!(room: @room, user: users(:other_user), body: "Hello from other!")

    # Assert it appears without page refresh
    assert_selector "[data-message]", text: "Hello from other!", wait: 5
  end
end
```

**Note:** System tests need the `async` or `test` adapter and may require `Capybara.server = :puma` for WebSocket support.

---

## Deployment

### Puma Configuration (Most Common)

```ruby
# config/puma.rb
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

# Action Cable needs at least as many DB connections as threads
# config/database.yml pool should be >= threads_count
```

### Nginx Reverse Proxy

```nginx
upstream app {
  server 127.0.0.1:3000;
}

server {
  listen 443 ssl;

  location /cable {
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # WebSocket timeout — increase from default 60s
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;
  }

  location / {
    proxy_pass http://app;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### Standalone Cable Server

For high-volume apps, run the cable server separately:

```ruby
# cable/config.ru
require_relative "../config/environment"
Rails.application.eager_load!
run ActionCable.server
```

```bash
bundle exec puma -p 28080 cable/config.ru
```

```ruby
# config/environments/production.rb
config.action_cable.mount_path = nil
config.action_cable.url = "wss://cable.yourdomain.com"
```

### Health Checks

```ruby
# config/routes.rb
get "/cable/health", to: proc { [200, {}, ["OK"]] }
```

---

## Turbo Streams vs Action Cable Decision Matrix

| Scenario | Use Turbo Streams | Use Action Cable |
|----------|:-:|:-:|
| New record appears on page | ✅ | |
| Record updated, reflect on page | ✅ | |
| Record deleted, remove from page | ✅ | |
| Chat messages (simple display) | ✅ | |
| Chat with typing indicators | | ✅ |
| Online/offline presence | | ✅ |
| Live dashboard with custom charts | | ✅ |
| Progress bar updates | | ✅ |
| Multiplayer game state | | ✅ |
| Notifications (HTML badge) | ✅ | |
| Notifications (browser Notification API) | | ✅ |
| Collaborative editing cursors | | ✅ |
| Live search/autocomplete | | ✅ |
| Model CRUD → page update | ✅ | |
| Client sends data to server | | ✅ |

### Turbo Streams Broadcasting Quick Reference

For comparison, here's how Turbo Streams handles the common case:

```ruby
# Model
class Message < ApplicationRecord
  broadcasts_to :room
end

# View — subscribes automatically
<%= turbo_stream_from @room %>

# That's it. No channel file, no JavaScript.
```

Action Cable is the transport layer underneath — Turbo Streams just automates the common pattern.

---

## Common Patterns

### Presence Tracking

```ruby
# app/channels/presence_channel.rb
class PresenceChannel < ApplicationCable::Channel
  def subscribed
    stream_from "presence_#{params[:room_id]}"
    current_user.update!(online: true, last_seen_at: Time.current)
    broadcast_presence
  end

  def unsubscribed
    current_user.update!(online: false)
    broadcast_presence
  end

  def heartbeat
    current_user.touch(:last_seen_at)
  end

  private
    def broadcast_presence
      online_users = User.where(online: true).pluck(:id, :display_name)
      ActionCable.server.broadcast("presence_#{params[:room_id]}", {
        type: "presence",
        users: online_users.map { |id, name| { id: id, name: name } }
      })
    end
end
```

### Typing Indicators

```ruby
# app/channels/typing_channel.rb
class TypingChannel < ApplicationCable::Channel
  def subscribed
    stream_from "typing_#{params[:room_id]}"
  end

  def typing(data)
    ActionCable.server.broadcast("typing_#{params[:room_id]}", {
      user_id: current_user.id,
      user_name: current_user.display_name,
      typing: data["typing"]
    })
  end
end
```

```js
// Client
let typingTimeout
inputEl.addEventListener("input", () => {
  channel.perform("typing", { typing: true })
  clearTimeout(typingTimeout)
  typingTimeout = setTimeout(() => {
    channel.perform("typing", { typing: false })
  }, 2000)
})
```

### Progress Tracking

```ruby
# app/channels/progress_channel.rb
class ProgressChannel < ApplicationCable::Channel
  def subscribed
    stream_for current_user
  end
end

# In a background job
class ImportJob < ApplicationJob
  def perform(user, file)
    total = count_rows(file)
    file.each_row.with_index do |row, i|
      process_row(row)

      if (i % 100).zero?
        ProgressChannel.broadcast_to(user, {
          job_id: job_id,
          progress: ((i.to_f / total) * 100).round,
          status: "processing"
        })
      end
    end

    ProgressChannel.broadcast_to(user, {
      job_id: job_id,
      progress: 100,
      status: "complete"
    })
  end
end
```

### Admin Dashboard / Live Metrics

```ruby
class MetricsChannel < ApplicationCable::Channel
  def subscribed
    reject unless current_user.admin?
    stream_from "admin_metrics"
  end
end

# Broadcast from a recurring job
class MetricsBroadcastJob < ApplicationJob
  def perform
    ActionCable.server.broadcast("admin_metrics", {
      active_users: User.where("last_seen_at > ?", 5.minutes.ago).count,
      requests_per_minute: RequestLog.where("created_at > ?", 1.minute.ago).count,
      error_rate: ErrorLog.where("created_at > ?", 5.minutes.ago).count
    })
  end
end
```

---

## Error Handling

### Connection-Level

```ruby
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    rescue_from StandardError, with: :report_error

    private
      def report_error(e)
        Rails.error.report(e, handled: true, context: {
          user_id: current_user&.id,
          action: "connection"
        })
      end
  end
end
```

### Channel-Level

```ruby
class ChatChannel < ApplicationCable::Channel
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :invalid_record

  def speak(data)
    room = Room.find(params[:room_id])
    room.messages.create!(user: current_user, body: data["body"])
  end

  private
    def record_not_found(error)
      # Transmit error to the specific client
      transmit(type: "error", message: "Resource not found")
    end

    def invalid_record(error)
      transmit(type: "error", message: error.record.errors.full_messages.join(", "))
    end
end
```

### Client-Side Error Display

```js
received(data) {
  if (data.type === "error") {
    this.showError(data.message)
    return
  }
  // Normal message handling
  this.appendMessage(data)
},

showError(message) {
  const el = document.getElementById("cable-errors")
  el.textContent = message
  el.classList.remove("hidden")
  setTimeout(() => el.classList.add("hidden"), 5000)
}
```

---

## Performance

### Keep Broadcasts Small

```ruby
# BAD — sending entire model with associations
ChatChannel.broadcast_to(room, message.as_json(include: [:user, :attachments]))

# GOOD — send only what's needed
ChatChannel.broadcast_to(room, {
  id: message.id,
  body: message.body,
  user_name: message.user.display_name,
  created_at: message.created_at.iso8601
})

# BETTER — pre-render HTML server-side
ChatChannel.broadcast_to(room, {
  html: ApplicationController.renderer.render(
    partial: "messages/message",
    locals: { message: message }
  )
})
```

### Use Background Jobs for Heavy Broadcasts

```ruby
# BAD — blocking the request
def create
  @message = @room.messages.create!(message_params)
  html = render_to_string(partial: "messages/message", locals: { message: @message })
  ActionCable.server.broadcast("chat_#{@room.id}", { html: html })
  head :created
end

# GOOD — non-blocking
def create
  @message = @room.messages.create!(message_params)
  MessageBroadcastJob.perform_later(@message)
  head :created
end
```

### Worker Pool Sizing

```ruby
# config/environments/production.rb
# Match to your thread count and DB pool
config.action_cable.worker_pool_size = ENV.fetch("CABLE_WORKER_POOL_SIZE", 4).to_i
```

Ensure your database pool is at least `worker_pool_size` + your web thread count.

### Throttling Client Sends

```js
// Debounce rapid client events (e.g., typing, cursor position)
function throttle(fn, delay) {
  let lastCall = 0
  return function(...args) {
    const now = Date.now()
    if (now - lastCall >= delay) {
      lastCall = now
      fn.apply(this, args)
    }
  }
}

const throttledTyping = throttle(() => {
  channel.perform("typing", { typing: true })
}, 500)
```

---

## Security

### Validate All Channel Params

```ruby
def subscribed
  # NEVER trust params directly
  room = Room.find_by(id: params[:room_id])

  # Authorize access
  if room && current_user.can_access?(room)
    stream_for room
  else
    reject
  end
end
```

### Sanitize Broadcast Data

```ruby
# Avoid broadcasting user-supplied HTML directly
ActionCable.server.broadcast("chat_#{room.id}", {
  body: ActionController::Base.helpers.sanitize(message.body),
  user_name: message.user.display_name
})
```

### Rate Limit Channel Actions

```ruby
class ChatChannel < ApplicationCable::Channel
  def speak(data)
    if rate_limited?
      transmit(type: "error", message: "Too many messages. Slow down.")
      return
    end

    record_action
    Message.create!(user: current_user, body: data["body"], room_id: params[:room_id])
  end

  private
    def rate_limited?
      key = "cable_rate:#{current_user.id}"
      count = Rails.cache.increment(key, 1, expires_in: 1.minute)
      count > 30  # 30 messages per minute
    end

    def record_action
      # Already incremented in rate_limited? check
    end
end
```

### Force Disconnect

```ruby
# Disconnect a specific user from all channels
ActionCable.server.remote_connections.where(current_user: user).disconnect

# Use case: user banned, account deleted, password changed
class User < ApplicationRecord
  after_update :disconnect_cable, if: :saved_change_to_password_digest?

  private
    def disconnect_cable
      ActionCable.server.remote_connections.where(current_user: self).disconnect
    end
end
```

### Allowed Request Origins

```ruby
# config/environments/production.rb
config.action_cable.allowed_request_origins = [
  "https://yourdomain.com",
  %r{https://.*\.yourdomain\.com}
]

# NEVER do this in production:
# config.action_cable.disable_request_forgery_protection = true
```
