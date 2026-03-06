# Turbo Reference — Patterns, Examples, Edge Cases

Detailed implementation patterns for the turbo skill. Read SKILL.md first.

---

## Table of Contents

1. [Inline Editing with Frames](#inline-editing-with-frames)
2. [Tab Navigation with Frames](#tab-navigation-with-frames)
3. [Lazy-Loaded Content](#lazy-loaded-content)
4. [Flash Messages with Streams](#flash-messages-with-streams)
5. [Live Search / Filtering](#live-search--filtering)
6. [Modal Dialogs](#modal-dialogs)
7. [Infinite Scroll](#infinite-scroll)
8. [Nested Form Items with Streams](#nested-form-items-with-streams)
9. [Counter / Badge Updates](#counter--badge-updates)
10. [Toast Notifications](#toast-notifications)
11. [Turbo Stream Templates (.turbo_stream.erb)](#turbo-stream-templates)
12. [Broadcasting Patterns](#broadcasting-patterns)
13. [Morph Refresh Patterns](#morph-refresh-patterns)
14. [Testing Turbo](#testing-turbo)
15. [Turbo + Stimulus Integration](#turbo--stimulus-integration)
16. [Edge Cases and Gotchas](#edge-cases-and-gotchas)

---

## Inline Editing with Frames

The classic Turbo Frame use case — click "Edit" to swap content in place.

### Show view (read mode)

```erb
<%# app/views/posts/_post.html.erb %>
<%= turbo_frame_tag dom_id(post) do %>
  <div class="post-card">
    <h3><%= post.title %></h3>
    <p><%= post.body %></p>
    <%= link_to "Edit", edit_post_path(post) %>
  </div>
<% end %>
```

### Edit view

```erb
<%# app/views/posts/edit.html.erb %>
<%= turbo_frame_tag dom_id(@post) do %>
  <%= render "form", post: @post %>
<% end %>

<%# Content outside the frame is ignored when loaded in a frame context %>
<p>This text only appears when visiting edit page directly.</p>
```

### Form partial

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with model: post do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body %>
  <div>
    <%= f.submit %>
    <%= link_to "Cancel", post_path(post) %>
  </div>
<% end %>
```

### Controller

```ruby
def update
  @post = Post.find(params[:id])
  if @post.update(post_params)
    redirect_to @post  # Turbo Frame follows redirect, fetches show, extracts matching frame
  else
    render :edit, status: :unprocessable_entity
  end
end
```

**How it works:** The redirect loads the show page, Turbo finds the matching `turbo_frame_tag dom_id(@post)`, and swaps just that frame. The URL doesn't change (it's a frame navigation).

---

## Tab Navigation with Frames

Use a frame with `target` to build tabbed interfaces without JavaScript.

```erb
<%# Tabs navigation — targets the content frame %>
<%= turbo_frame_tag "tabs_nav", target: "tab_content" do %>
  <nav>
    <%= link_to "Details", project_details_path(@project),
        class: ("active" if current_page?(project_details_path(@project))) %>
    <%= link_to "Members", project_members_path(@project),
        class: ("active" if current_page?(project_members_path(@project))) %>
    <%= link_to "Settings", project_settings_path(@project),
        class: ("active" if current_page?(project_settings_path(@project))) %>
  </nav>
<% end %>

<%# Content area — updated by tab clicks %>
<%= turbo_frame_tag "tab_content" do %>
  <%= yield %>
<% end %>
```

Each tab endpoint renders its content inside a matching frame:

```erb
<%# app/views/projects/details.html.erb %>
<%= turbo_frame_tag "tab_content" do %>
  <h2>Project Details</h2>
  <%# ... details content ... %>
<% end %>
```

**Tip:** Add `data-turbo-action="advance"` to tab links to update the URL:
```erb
<%= link_to "Details", project_details_path(@project),
    data: { turbo_action: "advance" } %>
```

---

## Lazy-Loaded Content

Load expensive content after the page renders.

```erb
<%# Main page loads fast %>
<h1><%= @post.title %></h1>
<p><%= @post.body %></p>

<%# Comments load lazily (deferred until visible) %>
<%= turbo_frame_tag "comments",
    src: post_comments_path(@post),
    loading: :lazy do %>
  <div class="loading-spinner">
    Loading comments...
  </div>
<% end %>

<%# Sidebar stats load immediately after page (no :lazy) %>
<%= turbo_frame_tag "stats",
    src: post_stats_path(@post) do %>
  <p>Loading stats...</p>
<% end %>
```

### Controller for lazy endpoint

```ruby
class CommentsController < ApplicationController
  def index
    @post = Post.find(params[:post_id])
    @comments = @post.comments.includes(:author).order(created_at: :desc)
    # Renders index.html.erb — Turbo extracts the matching frame
  end
end
```

```erb
<%# app/views/comments/index.html.erb %>
<%= turbo_frame_tag "comments" do %>
  <% @comments.each do |comment| %>
    <%= render comment %>
  <% end %>
  <p><%= pluralize(@comments.count, "comment") %></p>
<% end %>
```

**Key point:** The controller doesn't know or care that it's being loaded in a frame. It renders a full page. Turbo extracts only the matching frame.

---

## Flash Messages with Streams

Update flash messages after stream-based form submissions.

```erb
<%# app/views/layouts/application.html.erb %>
<div id="flash_messages">
  <% flash.each do |type, message| %>
    <div class="flash flash-<%= type %>"><%= message %></div>
  <% end %>
</div>
```

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts", partial: "posts/post", locals: { post: @post } %>

<%= turbo_stream.update "flash_messages" do %>
  <div class="flash flash-notice">Post created successfully!</div>
<% end %>
```

**Alternative — use a shared partial:**

```erb
<%# app/views/shared/_flash.html.erb %>
<% flash.each do |type, message| %>
  <div class="flash flash-<%= type %>" data-controller="auto-dismiss"><%= message %></div>
<% end %>

<%# In the stream response %>
<%= turbo_stream.update "flash_messages", partial: "shared/flash" %>
```

---

## Live Search / Filtering

Turbo Frames make live search trivial — no JavaScript needed.

```erb
<%# app/views/posts/index.html.erb %>
<%= turbo_frame_tag "search_results" do %>
  <%= form_with url: posts_path, method: :get, data: { turbo_frame: "search_results" } do |f| %>
    <%= f.search_field :q, value: params[:q],
        oninput: "this.form.requestSubmit()",
        placeholder: "Search posts..." %>
  <% end %>

  <div id="posts_list">
    <%= render @posts %>
  </div>
<% end %>
```

**How it works:**
1. User types in search field
2. `oninput` submits the form (GET request)
3. The form targets the `search_results` frame (via `data-turbo-frame`)
4. Server returns filtered results
5. Turbo swaps the frame

**Debounce with Stimulus (optional):**
```erb
<%= f.search_field :q, value: params[:q],
    data: { controller: "debounce", action: "input->debounce#search" } %>
```

**Tip:** Add `data-turbo-action="replace"` to the form to update the URL with search params without creating history entries.

---

## Modal Dialogs

Use a dedicated frame for modals. The content loads from a different page.

```erb
<%# app/views/layouts/application.html.erb %>
<%= yield %>

<%# Empty frame for modals at the bottom of the layout %>
<%= turbo_frame_tag "modal" %>
```

```erb
<%# Any link can target the modal frame %>
<%= link_to "New Post", new_post_path, data: { turbo_frame: "modal" } %>
```

```erb
<%# app/views/posts/new.html.erb %>
<%= turbo_frame_tag "modal" do %>
  <div class="modal-overlay" data-controller="modal">
    <div class="modal-content">
      <h2>New Post</h2>
      <%= render "form", post: @post %>
    </div>
  </div>
<% end %>

<%# Fallback when visiting directly (no frame context) %>
<% content_for :main do %>
  <h1>New Post</h1>
  <%= render "form", post: @post %>
<% end %>
```

**Close the modal after successful save:**
```ruby
def create
  @post = Post.new(post_params)
  if @post.save
    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: [
          turbo_stream.prepend("posts", partial: "posts/post", locals: { post: @post }),
          turbo_stream.update("modal", "")  # Clear the modal
        ]
      end
      format.html { redirect_to @post }
    end
  else
    render :new, status: :unprocessable_entity
  end
end
```

---

## Infinite Scroll

Load more content when the user scrolls to the bottom.

```erb
<%# app/views/posts/index.html.erb %>
<div id="posts">
  <%= render @posts %>
</div>

<%= turbo_frame_tag "pagination", src: posts_path(page: @next_page), loading: :lazy do %>
  <p>Loading more...</p>
<% end if @next_page %>
```

```erb
<%# The paginated response APPENDS posts and includes the next pagination frame %>
<%# app/views/posts/index.turbo_frame.erb — or just use the same template %>
<%= turbo_frame_tag "pagination" do %>
  <%# This frame's content replaces the old "pagination" frame %>
  
  <%# Append each post using turbo streams instead, OR: %>
  <% @posts.each do |post| %>
    <%= render post %>
  <% end %>
  
  <%# Next page lazy loader %>
  <% if @next_page %>
    <%= turbo_frame_tag "pagination", src: posts_path(page: @next_page), loading: :lazy do %>
      <p>Loading more...</p>
    <% end %>
  <% end %>
<% end %>
```

**Better approach using Turbo Streams:**
```ruby
# Controller
def index
  @posts = Post.page(params[:page]).per(20)
  @next_page = @posts.next_page

  respond_to do |format|
    format.html
    format.turbo_stream
  end
end
```

```erb
<%# app/views/posts/index.turbo_stream.erb %>
<% @posts.each do |post| %>
  <%= turbo_stream.append "posts", partial: "posts/post", locals: { post: post } %>
<% end %>

<% if @next_page %>
  <%= turbo_stream.replace "pagination" do %>
    <%= turbo_frame_tag "pagination", src: posts_path(page: @next_page, format: :turbo_stream), loading: :lazy do %>
      <p>Loading more...</p>
    <% end %>
  <% end %>
<% else %>
  <%= turbo_stream.remove "pagination" %>
<% end %>
```

---

## Nested Form Items with Streams

Add/remove nested form fields dynamically.

```erb
<%# Parent form %>
<%= form_with model: @invoice do |f| %>
  <%= f.text_field :number %>
  
  <div id="line_items">
    <%= f.fields_for :line_items do |li| %>
      <%= render "line_item_fields", f: li %>
    <% end %>
  </div>
  
  <%= link_to "Add Line Item", new_line_item_invoice_path(@invoice),
      data: { turbo_method: :post } %>
  
  <%= f.submit %>
<% end %>
```

```ruby
# Controller action that returns a stream with the new fields
def add_line_item
  respond_to do |format|
    format.turbo_stream do
      render turbo_stream: turbo_stream.append("line_items",
        partial: "line_item_fields",
        locals: { f: nil, index: Time.current.to_i })
    end
  end
end
```

---

## Counter / Badge Updates

Update counts in the navbar or sidebar after actions.

```erb
<%# Layout with a notification badge %>
<nav>
  <span id="unread_count"><%= current_user.unread_notifications_count %></span>
</nav>
```

```erb
<%# After marking a notification as read %>
<%= turbo_stream.replace dom_id(@notification), partial: "notifications/notification",
    locals: { notification: @notification } %>
<%= turbo_stream.update "unread_count", html: current_user.unread_notifications_count.to_s %>
```

**With broadcasts (real-time badge updates):**
```ruby
class Notification < ApplicationRecord
  after_create_commit do
    broadcast_update_to(
      user,  # Stream name scoped to user
      target: "unread_count",
      html: user.unread_notifications_count.to_s
    )
  end
end
```

```erb
<%# Subscribe to user-specific stream %>
<%= turbo_stream_from current_user %>
```

---

## Toast Notifications

Push ephemeral notifications to any user.

```erb
<%# Layout %>
<div id="toasts" class="toast-container"></div>
```

```ruby
# From anywhere — controller, job, model
Turbo::StreamsChannel.broadcast_append_to(
  user,
  target: "toasts",
  partial: "shared/toast",
  locals: { message: "New message from #{sender.name}", type: "info" }
)
```

```erb
<%# app/views/shared/_toast.html.erb %>
<div class="toast toast-<%= type %>" data-controller="auto-dismiss" data-auto-dismiss-delay-value="5000">
  <%= message %>
</div>
```

---

## Turbo Stream Templates

### File naming convention

```
app/views/posts/create.turbo_stream.erb
app/views/posts/update.turbo_stream.erb
app/views/posts/destroy.turbo_stream.erb
```

Rails automatically renders `{action}.turbo_stream.erb` when the controller responds with `format.turbo_stream`.

### Multi-action template example

```erb
<%# app/views/comments/create.turbo_stream.erb %>

<%# 1. Add the new comment to the list %>
<%= turbo_stream.append "comments_for_#{@comment.post_id}",
    partial: "comments/comment",
    locals: { comment: @comment } %>

<%# 2. Update the comment count %>
<%= turbo_stream.update "comment_count_#{@comment.post_id}",
    html: pluralize(@comment.post.comments.count, "comment") %>

<%# 3. Clear the form %>
<%= turbo_stream.replace "new_comment_form" do %>
  <%= render "comments/form", comment: Comment.new(post: @comment.post) %>
<% end %>
```

### Stream with block syntax

```erb
<%= turbo_stream.update "sidebar" do %>
  <h3>Updated Content</h3>
  <p>This can contain arbitrary HTML.</p>
  <%= render partial: "shared/widget" %>
<% end %>
```

---

## Broadcasting Patterns

### Scoped broadcasts (multi-tenant safe)

```ruby
class Message < ApplicationRecord
  belongs_to :room

  # Broadcast to room-specific stream
  broadcasts_to :room

  # Equivalent to:
  # after_create_commit  { broadcast_append_to(room) }
  # after_update_commit  { broadcast_replace_to(room) }
  # after_destroy_commit { broadcast_remove_to(room) }
end
```

```erb
<%# Subscribe to this room's messages %>
<%= turbo_stream_from @room %>
```

### Custom stream names

```ruby
after_create_commit do
  broadcast_append_to(
    [project, "tasks"],       # Stream name: "projects:123:tasks"
    target: "task_list",
    partial: "tasks/task_row"
  )
end
```

### Broadcast to specific users

```ruby
# Notify just one user
after_create_commit do
  broadcast_append_to(
    assignee,                   # User model — stream is "users:456"
    target: "notifications",
    partial: "notifications/notification"
  )
end
```

### Background job broadcasts

```ruby
class ImportJob < ApplicationJob
  def perform(import)
    import.rows.each_with_index do |row, i|
      process(row)

      # Update progress bar
      Turbo::StreamsChannel.broadcast_replace_to(
        import,
        target: "import_progress",
        partial: "imports/progress",
        locals: { current: i + 1, total: import.rows.count }
      )
    end

    Turbo::StreamsChannel.broadcast_replace_to(
      import,
      target: "import_status",
      html: "<p class='success'>Import complete!</p>"
    )
  end
end
```

### Suppressing broadcasts for the current user

Sometimes you don't want the acting user to receive the broadcast (they already see the change via the HTTP response).

```ruby
# In controller
after_action :suppress_turbo_broadcast, only: [:create, :update]

# Or handle in the partial with a conditional
<%# _message.html.erb %>
<div id="<%= dom_id(message) %>" class="message <%= 'own' if message.user == current_user %>">
  <%= message.body %>
</div>
```

---

## Morph Refresh Patterns

### Basic morph setup (Rails 8+)

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
</head>
```

### Model-driven refresh

```ruby
class Post < ApplicationRecord
  # Broadcast a page refresh (instead of granular stream actions)
  broadcasts_refreshes

  # Or manually:
  after_update_commit { broadcast_refresh_to "posts" }
end
```

### Preserving elements across morphs

```erb
<%# Video player, audio, maps — anything with client-side state %>
<div id="video_player" data-turbo-permanent>
  <video src="<%= @video.url %>" autoplay></video>
</div>

<%# Form inputs are automatically preserved during morph if they have focus %>
```

### When morph is better than streams

| Scenario | Use Morph | Use Streams |
|----------|-----------|-------------|
| Dashboard with many counters | ✅ Re-render whole page | ❌ Too many stream targets |
| Chat messages | ❌ Full page refresh wasteful | ✅ Append single message |
| Kanban board reorder | ✅ Morph preserves drag state | ❌ Complex multi-target streams |
| Single field update | ❌ Overkill | ✅ Targeted update |

---

## Testing Turbo

### Testing Turbo Stream responses

```ruby
class PostsControllerTest < ActionDispatch::IntegrationTest
  test "create responds with turbo stream" do
    sign_in users(:active_user)

    post posts_path, params: { post: { title: "Test", body: "Content" } },
         headers: { "Accept" => "text/vnd.turbo-stream.html" }

    assert_response :success
    assert_match "turbo-stream", response.content_type
    assert_match "prepend", response.body
    assert_match "posts", response.body  # target
  end

  test "create with invalid data returns unprocessable entity" do
    sign_in users(:active_user)

    post posts_path, params: { post: { title: "" } }

    assert_response :unprocessable_entity
  end
end
```

### Testing Turbo Frame responses

```ruby
test "edit loads in turbo frame" do
  sign_in users(:active_user)
  post = posts(:published)

  get edit_post_path(post),
      headers: { "Turbo-Frame" => dom_id(post) }

  assert_response :success
  assert_select "turbo-frame##{dom_id(post)}"
end
```

### System tests with Turbo

```ruby
class PostsSystemTest < ApplicationSystemTestCase
  test "inline edit with turbo frame" do
    sign_in users(:active_user)
    post = posts(:published)

    visit posts_path

    within "##{dom_id(post)}" do
      click_on "Edit"
      fill_in "Title", with: "Updated Title"
      click_on "Update"
    end

    # Frame updates in place — no full page navigation
    within "##{dom_id(post)}" do
      assert_text "Updated Title"
    end
  end

  test "turbo stream appends new post" do
    sign_in users(:active_user)
    visit posts_path

    fill_in "Title", with: "New Post"
    click_on "Create"

    # New post appears without page reload
    assert_selector "#posts .post", count: Post.count
    assert_text "New Post"
  end
end
```

### Testing broadcasts

```ruby
class PostBroadcastTest < ActiveSupport::TestCase
  test "broadcasts append on create" do
    assert_broadcasts_on("posts", count: 1) do
      Post.create!(title: "Test", body: "Content")
    end
  end

  # Or test the broadcast content
  test "broadcasts to project stream" do
    project = projects(:active)

    assert_broadcast_on(project, count: 1) do
      project.posts.create!(title: "Test", body: "Content")
    end
  end
end
```

---

## Turbo + Stimulus Integration

Turbo and Stimulus complement each other. Turbo handles HTML delivery; Stimulus handles client-side behavior.

### Auto-dismiss flash messages

```erb
<div data-controller="auto-dismiss" data-auto-dismiss-delay-value="3000">
  <div class="flash">Post created!</div>
</div>
```

```javascript
// app/javascript/controllers/auto_dismiss_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 3000 } }

  connect() {
    this.timeout = setTimeout(() => {
      this.element.remove()
    }, this.delayValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

### Form submission with loading state

```erb
<%= form_with model: @post, data: { controller: "submit-button" } do |f| %>
  <%= f.text_field :title %>
  <%= f.submit "Save", data: { submit_button_target: "button", action: "submit->submit-button#disable" } %>
<% end %>
```

### Turbo events in Stimulus

```javascript
// Listen for Turbo events in a Stimulus controller
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    // Turbo Frame loaded
    this.element.addEventListener("turbo:frame-load", this.onFrameLoad.bind(this))

    // Before Turbo Stream action
    document.addEventListener("turbo:before-stream-render", this.onBeforeStream.bind(this))
  }

  onFrameLoad(event) {
    // Frame content was replaced
    console.log("Frame loaded:", event.target.id)
  }

  onBeforeStream(event) {
    // Customize stream behavior
    const { action, target } = event.detail.newStream
    console.log(`Stream: ${action} → ${target}`)
  }
}
```

### Key Turbo events

| Event | When | Use For |
|-------|------|---------|
| `turbo:load` | Page load (Drive) | Initialize page-level JS |
| `turbo:frame-load` | Frame content loaded | Post-load setup for frame content |
| `turbo:before-stream-render` | Before stream action | Custom stream behavior |
| `turbo:submit-start` | Form submit begins | Show loading state |
| `turbo:submit-end` | Form submit completes | Hide loading state |
| `turbo:before-fetch-request` | Before any Turbo fetch | Add headers, modify request |
| `turbo:before-fetch-response` | Before processing response | Inspect/modify response |

---

## Edge Cases and Gotchas

### 1. Frame requests include a `Turbo-Frame` header

The server can detect frame requests:
```ruby
def show
  if turbo_frame_request?
    # Only render the minimal content needed for the frame
    render partial: "post_detail", locals: { post: @post }
  else
    # Full page render
  end
end
```

But **you usually don't need this.** Just include matching `turbo_frame_tag` in your normal views.

### 2. Turbo caching and preview

Turbo Drive caches pages for back/forward navigation. This can show stale content.

```erb
<%# Disable caching for this page %>
<meta name="turbo-cache-control" content="no-cache">

<%# Or no-preview (don't show cached version while loading) %>
<meta name="turbo-cache-control" content="no-preview">
```

### 3. File uploads

Standard file uploads break with Turbo. Options:

```erb
<%# Option A: Disable Turbo for the form %>
<%= form_with model: @document, data: { turbo: false } do |f| %>
  <%= f.file_field :attachment %>
<% end %>

<%# Option B: Use Active Storage direct upload (works with Turbo) %>
<%= form_with model: @document do |f| %>
  <%= f.file_field :attachment, direct_upload: true %>
<% end %>
```

### 4. Redirect after DELETE

```ruby
def destroy
  @post.destroy

  respond_to do |format|
    format.turbo_stream { render turbo_stream: turbo_stream.remove(dom_id(@post)) }
    format.html { redirect_to posts_path, status: :see_other }  # ← 303, not 302!
  end
end
```

**`status: :see_other` (303) is required for DELETE redirects with Turbo.** A 302 redirect after DELETE will try to DELETE the redirect target. 303 forces a GET.

### 5. Custom Turbo Stream actions

You can define custom actions beyond the 7 built-in ones:

```javascript
// app/javascript/application.js
import { StreamActions } from "@hotwired/turbo"

// Custom "log" action
StreamActions.log = function() {
  console.log(this.getAttribute("message"))
}

// Custom "redirect" action
StreamActions.redirect = function() {
  Turbo.visit(this.getAttribute("url"))
}
```

```erb
<%# Use in a template %>
<turbo-stream action="redirect" url="<%= posts_path %>"></turbo-stream>

<%# Or from Ruby %>
<%= turbo_stream.action(:redirect, url: posts_path) %>
```

### 6. Turbo Streams over SSE (Server-Sent Events)

Alternative to WebSockets for broadcasting:

```ruby
# config/cable.yml
# Use async adapter for development, Redis for production
development:
  adapter: async

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") %>
```

### 7. Signed stream names (security)

`turbo_stream_from` uses signed stream names to prevent unauthorized subscriptions:

```erb
<%# This is safe — stream name is signed server-side %>
<%= turbo_stream_from current_user %>

<%# For custom verification: %>
<%= turbo_stream_from current_user, :notifications %>
```

### 8. Handling race conditions in broadcasts

Broadcasts from `after_create_commit` can fire before the transaction is fully visible to other database connections (especially with replicas):

```ruby
# If you see missing records in broadcast partials, add a small delay:
after_create_commit do
  broadcast_append_later_to("posts",
    target: "posts_list",
    partial: "posts/post")
end
```

`broadcast_*_later_to` methods use Active Job, which adds a natural delay.

### 9. CSRF tokens and Turbo

Turbo automatically includes CSRF tokens in form submissions. But if you're making custom fetch requests alongside Turbo:

```javascript
// Get the CSRF token
const token = document.head.querySelector("meta[name=csrf-token]")?.content

fetch("/posts", {
  method: "POST",
  headers: {
    "X-CSRF-Token": token,
    "Content-Type": "application/json",
    "Accept": "text/vnd.turbo-stream.html"  // Request Turbo Stream response
  },
  body: JSON.stringify({ post: { title: "Test" } })
})
```

### 10. Frame loading states

Style frames during loading:

```css
turbo-frame {
  /* While loading (has src, hasn't loaded yet) */
  &[busy] {
    opacity: 0.5;
    pointer-events: none;
  }
}

/* Or target the loading placeholder */
turbo-frame:not([complete]) .loading-indicator {
  display: block;
}
```

### 11. Preventing double submissions

Turbo automatically disables submit buttons during submission. But for custom buttons:

```erb
<%= f.submit "Save", data: { turbo_submits_with: "Saving..." } %>
```

This changes the button text to "Saving..." and disables it during submission.

### 12. Turbo Native (mobile apps)

If your Rails app serves a Turbo Native iOS/Android app, some patterns change:

```ruby
# Detect Turbo Native requests
if turbo_native_app?
  # Return minimal chrome, no nav bar, etc.
  render layout: "turbo_native"
end
```

### 13. Content Security Policy (CSP)

Turbo Streams use inline `<turbo-stream>` elements. If your CSP blocks inline scripts, you may need:

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  # Turbo doesn't use inline scripts, but some custom actions might
  policy.script_src :self
end
```

### 14. Request format precedence

When both `format.turbo_stream` and `format.html` are defined:
- **Turbo form submission** → picks `turbo_stream` (Turbo sends `Accept: text/vnd.turbo-stream.html`)
- **Direct browser navigation** → picks `html`
- **JavaScript `fetch`** → depends on your `Accept` header

Always define both for graceful degradation:
```ruby
respond_to do |format|
  format.turbo_stream { ... }  # Turbo-enhanced
  format.html { ... }          # Fallback
end
```
