# Action Text Reference

Detailed patterns, examples, and edge cases for Action Text in Rails.

## Table of Contents

- [Installation Details](#installation-details)
- [Lexxy Editor Configuration](#lexxy-editor-configuration)
- [Trix Editor Configuration](#trix-editor-configuration)
- [Model Patterns](#model-patterns)
- [Form Patterns](#form-patterns)
- [Rendering Patterns](#rendering-patterns)
- [Attachment Patterns](#attachment-patterns)
- [Custom Attachables](#custom-attachables)
- [Styling Reference](#styling-reference)
- [N+1 Query Prevention](#n1-query-prevention)
- [Content Security](#content-security)
- [API / Headless Usage](#api--headless-usage)
- [Testing Patterns](#testing-patterns)
- [Migration & UUID Support](#migration--uuid-support)
- [Troubleshooting](#troubleshooting)

---

## Installation Details

### What `action_text:install` Does

1. **Creates migrations** for:
   - `action_text_rich_texts` (polymorphic join table)
   - `active_storage_blobs` (if Active Storage not installed)
   - `active_storage_attachments` (if Active Storage not installed)
   - `active_storage_variant_records` (if Active Storage not installed)

2. **Adds JavaScript** imports to `app/javascript/application.js`:
   ```js
   import "trix"
   import "@rails/actiontext"
   ```

3. **Creates view partials**:
   - `app/views/layouts/action_text/contents/_content.html.erb`
   - `app/views/active_storage/blobs/_blob.html.erb`

4. **Creates stylesheet**: `app/assets/stylesheets/actiontext.css`

5. **Adds `image_processing` gem** to Gemfile

### Schema: `action_text_rich_texts`

```ruby
create_table :action_text_rich_texts do |t|
  t.string     :name, null: false        # The attribute name (e.g., "content")
  t.text       :body                     # The HTML content
  t.references :record, null: false, polymorphic: true, index: false
  t.timestamps

  t.index [:record_type, :record_id, :name], name: "index_action_text_rich_texts_uniqueness", unique: true
end
```

### Active Storage Dependencies

Action Text uses Active Storage for file uploads. Ensure these are available:

| Dependency | Purpose | Install |
|-----------|---------|---------|
| `libvips` | Image analysis & transforms | `apt install libvips` / `brew install vips` |
| `image_processing` gem | Ruby bindings for vips/imagemagick | In Gemfile |
| `ffmpeg` | Video previews (optional) | `apt install ffmpeg` |
| `poppler` | PDF previews (optional) | `apt install poppler-utils` |

---

## Lexxy Editor Configuration

Lexxy is the modern replacement for Trix, shipping with Rails 8.1+.

### Setup

```ruby
# Gemfile
gem "lexxy"
```

```erb
<%# Layout: load lexxy stylesheet BEFORE app styles %>
<%= stylesheet_link_tag "lexxy", "data-turbo-track": "reload" %>
<%= stylesheet_link_tag :app, "data-turbo-track": "reload" %>
```

### Content Wrapper

Update the Action Text content partial to use Lexxy's class:

```erb
<%# app/views/layouts/action_text/contents/_content.html.erb %>
<div class="lexxy-content">
  <%= yield %>
</div>
```

### Full CSS Variables Reference

Override these in your app CSS (loaded after `lexxy.css`):

```css
:root {
  /* Ink (text) colors */
  --lexxy-color-ink: /* Primary text */;
  --lexxy-color-ink-medium: /* Secondary text */;
  --lexxy-color-ink-light: /* Tertiary/subtle text */;
  --lexxy-color-ink-lighter: /* Borders, dividers */;
  --lexxy-color-ink-lightest: /* Subtle backgrounds */;
  --lexxy-color-ink-inverted: /* Inverted text (on dark bg) */;

  /* Canvas and text */
  --lexxy-color-canvas: /* Editor background */;
  --lexxy-color-text: /* Editor text */;
  --lexxy-color-text-subtle: /* Placeholder text */;
  --lexxy-color-link: /* Link color */;
  --lexxy-color-code-bg: /* Code block background */;

  /* Selection */
  --lexxy-color-selected: /* Selection background */;
  --lexxy-color-selected-hover: /* Selection hover */;

  /* Accents */
  --lexxy-color-accent-dark: /* Primary accent */;
  --lexxy-color-accent-medium: /* Secondary accent */;
  --lexxy-color-accent-light: /* Light accent */;
  --lexxy-color-accent-lightest: /* Lightest accent */;

  /* Focus */
  --lexxy-focus-ring-color: /* Focus outline color */;

  /* Typography */
  --lexxy-font-base: /* Base font family */;
  --lexxy-font-mono: /* Monospace font family */;
}
```

### Dark Mode Pattern

```css
:root {
  color-scheme: light dark;

  --lexxy-color-canvas: light-dark(#ffffff, #1a1a1a);
  --lexxy-color-text: light-dark(#1a1a1a, #e5e5e5);
  --lexxy-color-link: light-dark(#2563eb, #60a5fa);
  --lexxy-color-code-bg: light-dark(#f5f5f5, #2a2a2a);
  --lexxy-color-selected: light-dark(oklch(0.92 0.026 254), oklch(0.3 0.05 254));
}
```

### Editor Sizing

```css
lexxy-editor,
.lexxy-editor {
  min-height: 200px;  /* Minimum editor height */
  max-height: 600px;  /* Optional: cap height */
  overflow-y: auto;   /* Scroll when exceeding max */
}
```

---

## Trix Editor Configuration

Trix is the original Action Text editor. Still the default in Rails 6–8.0.

### Toolbar Customization

```js
// Disable file attachments
document.addEventListener("trix-file-accept", (event) => {
  event.preventDefault()
  alert("File attachments are not supported")
})

// Custom toolbar actions
document.addEventListener("trix-initialize", (event) => {
  const toolbar = event.target.toolbarElement
  // Modify toolbar HTML as needed
})
```

### Disable Specific Toolbar Buttons

```css
/* Hide the file attachment button */
trix-toolbar .trix-button--icon-attach { display: none; }

/* Hide code formatting */
trix-toolbar .trix-button--icon-code { display: none; }

/* Hide heading button */
trix-toolbar .trix-button--icon-heading-1 { display: none; }
```

---

## Model Patterns

### Single Rich Text Field

```ruby
class Article < ApplicationRecord
  has_rich_text :content
end
```

### Multiple Rich Text Fields

```ruby
class Product < ApplicationRecord
  has_rich_text :description
  has_rich_text :specifications
  has_rich_text :care_instructions
end
```

### Rich Text with Validations

Action Text doesn't ship with built-in validations. Add custom ones:

```ruby
class Article < ApplicationRecord
  has_rich_text :content

  validate :content_not_blank
  validate :content_length

  private

  def content_not_blank
    errors.add(:content, "can't be blank") if content.blank?
  end

  def content_length
    if content.to_plain_text.length > 10_000
      errors.add(:content, "is too long (maximum 10,000 characters)")
    end
  end
end
```

### Rich Text with Callbacks

```ruby
class Article < ApplicationRecord
  has_rich_text :content

  before_save :extract_plain_text_excerpt

  private

  def extract_plain_text_excerpt
    self.excerpt = content.to_plain_text.truncate(300) if content.present?
  end
end
```

### Checking for Attachments

```ruby
article = Article.find(1)

# Check if content has any attachments
article.content.embeds.any?

# Count attachments
article.content.embeds.count

# Access attachment blobs
article.content.embeds.map(&:blob)
```

---

## Form Patterns

### Basic Form

```erb
<%= form_with model: @article do |form| %>
  <%= form.rich_text_area :content %>
<% end %>
```

### With Placeholder

```erb
<%= form.rich_text_area :content, placeholder: "Write your article..." %>
```

### With Stimulus Controller

```erb
<%= form.rich_text_area :content,
      data: { controller: "autosave", autosave_url_value: autosave_article_path(@article) } %>
```

### Multiple Rich Text Fields in One Form

```erb
<%= form_with model: @product do |form| %>
  <div class="field">
    <%= form.label :description, "Product Description" %>
    <%= form.rich_text_area :description %>
  </div>

  <div class="field">
    <%= form.label :specifications, "Technical Specs" %>
    <%= form.rich_text_area :specifications %>
  </div>
<% end %>
```

### Controller Params

```ruby
class ProductsController < ApplicationController
  private

  def product_params
    params.expect(product: [:name, :price, :description, :specifications])
  end
end
```

---

## Rendering Patterns

### Basic Rendering

```erb
<%# Renders sanitized HTML — safe to embed directly %>
<%= @article.content %>
```

### With Wrapper Class

```erb
<div class="article-body lexxy-content">
  <%= @article.content %>
</div>
```

### Plain Text Extraction

```ruby
# Full plain text (strips all HTML)
@article.content.to_plain_text
# => "Hello world. This is bold text."

# Truncated for previews
@article.content.to_plain_text.truncate(150)

# For meta description
content_tag :meta, nil, name: "description",
  content: @article.content.to_plain_text.truncate(160)
```

### Conditional Rendering

```erb
<% if @article.content.present? %>
  <div class="article-content">
    <%= @article.content %>
  </div>
<% else %>
  <p class="text-muted">No content yet.</p>
<% end %>
```

### Rendering in JSON (API)

```ruby
class ArticleSerializer
  def as_json
    {
      id: article.id,
      title: article.title,
      content_html: article.content.to_s,           # Sanitized HTML
      content_plain: article.content.to_plain_text,  # Plain text
      has_attachments: article.content.embeds.any?
    }
  end
end
```

---

## Attachment Patterns

### Image Attachments

Images dragged into the editor are automatically uploaded via Active Storage Direct Upload.

**Customize the blob partial:**

```erb
<%# app/views/active_storage/blobs/_blob.html.erb %>
<figure class="attachment attachment--<%= blob.representable? ? "preview" : "file" %> attachment--<%= blob.filename.extension %>">
  <% if blob.representable? %>
    <%= image_tag blob.representation(resize_to_limit: local_assigns[:in_gallery] ? [800, 600] : [1024, 768]),
          loading: "lazy",
          class: "attachment__image" %>
  <% end %>

  <figcaption class="attachment__caption">
    <% if caption = blob.try(:caption) %>
      <%= caption %>
    <% else %>
      <span class="attachment__name"><%= blob.filename %></span>
      <span class="attachment__size"><%= number_to_human_size blob.byte_size %></span>
    <% end %>
  </figcaption>
</figure>
```

### Disable File Attachments

If you don't want file uploads in the editor:

```js
// Trix
document.addEventListener("trix-file-accept", (event) => {
  event.preventDefault()
})
```

### Restrict File Types

```js
// Trix: Only allow images
document.addEventListener("trix-file-accept", (event) => {
  const acceptedTypes = ["image/jpeg", "image/png", "image/gif", "image/webp"]
  if (!acceptedTypes.includes(event.file.type)) {
    event.preventDefault()
    alert("Only image files are allowed")
  }
})
```

### Restrict File Size

```js
// Trix: Max 10MB
document.addEventListener("trix-file-accept", (event) => {
  const maxSize = 10 * 1024 * 1024 // 10MB
  if (event.file.size > maxSize) {
    event.preventDefault()
    alert("File is too large (max 10MB)")
  }
})
```

### Direct Upload Progress

```js
addEventListener("direct-upload:progress", (event) => {
  const { id, progress } = event.detail
  const progressBar = document.getElementById(`direct-upload-${id}`)
  if (progressBar) progressBar.style.width = `${progress}%`
})
```

### Purging Unattached Uploads

Files uploaded but never saved in rich text content become orphaned. Clean them up:

```ruby
# In a recurring job
ActiveStorage::Blob.unattached.where("created_at < ?", 2.days.ago).find_each(&:purge_later)
```

---

## Custom Attachables

Embed any model inside rich text content using Signed GlobalIDs.

### Basic Custom Attachable

```ruby
# app/models/user.rb
class User < ApplicationRecord
  include ActionText::Attachable

  # Custom partial for rendering inside rich text
  def to_attachable_partial_path
    "users/mention"
  end
end
```

```erb
<%# app/views/users/_mention.html.erb %>
<span class="user-mention" data-user-id="<%= user.id %>">
  <%= image_tag user.avatar_url, class: "user-mention__avatar", size: "20x20" if user.avatar_url %>
  @<%= user.name %>
</span>
```

### Inserting Attachables Programmatically

```ruby
user = User.find(1)
sgid = user.attachable_sgid

# Build the attachment HTML
attachment_html = %(<action-text-attachment sgid="#{sgid}"></action-text-attachment>)

# Use in content
article.update!(content: "Hello #{attachment_html}, welcome!")
```

### Product Embed Example

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  include ActionText::Attachable

  def to_attachable_partial_path
    "products/embed"
  end

  def self.to_missing_attachable_partial_path
    "products/missing_embed"
  end
end
```

```erb
<%# app/views/products/_embed.html.erb %>
<div class="product-embed">
  <% if product.image.attached? %>
    <%= image_tag product.image.variant(resize_to_fill: [80, 80]), class: "product-embed__image" %>
  <% end %>
  <div class="product-embed__info">
    <strong><%= product.name %></strong>
    <span class="product-embed__price"><%= number_to_currency product.price %></span>
  </div>
</div>
```

```erb
<%# app/views/products/missing_embed.html.erb %>
<div class="product-embed product-embed--missing">
  <span>Product no longer available</span>
</div>
```

### Custom SGIDs with Expiration

```ruby
user.attachable_sgid                              # Default: no expiration
user.to_sgid(expires_in: 1.hour).to_s             # Expires in 1 hour
user.to_sgid(for: "action_text_attachable").to_s   # Scoped purpose
```

---

## Styling Reference

### Comprehensive Lexxy Content Styles

```css
.lexxy-content {
  line-height: 1.6;
  overflow-wrap: break-word;
  word-break: break-word;
}

/* Paragraphs */
.lexxy-content p {
  margin: 0 0 1rem;
}

/* Headings */
.lexxy-content h1 {
  font-size: 1.75rem;
  font-weight: 700;
  margin: 2rem 0 0.75rem;
}
.lexxy-content h2 {
  font-size: 1.375rem;
  font-weight: 600;
  margin: 1.75rem 0 0.75rem;
}
.lexxy-content h3 {
  font-size: 1.125rem;
  font-weight: 600;
  margin: 1.5rem 0 0.75rem;
}

/* First child: no top margin */
.lexxy-content > :first-child { margin-top: 0; }

/* Lists */
.lexxy-content ul,
.lexxy-content ol {
  margin: 0 0 1rem;
  padding-left: 1.5rem;
}
.lexxy-content li { margin-bottom: 0.25rem; }
.lexxy-content li > ul,
.lexxy-content li > ol { margin-top: 0.25rem; margin-bottom: 0; }

/* Links */
.lexxy-content a {
  color: var(--lexxy-color-link, #2563eb);
  text-decoration: underline;
  text-underline-offset: 2px;
}
.lexxy-content a:hover { text-decoration-thickness: 2px; }

/* Blockquotes */
.lexxy-content blockquote {
  border-left: 3px solid var(--color-border, #d1d5db);
  margin: 1rem 0;
  padding: 0.5rem 0 0.5rem 1rem;
  color: var(--color-ink-muted, #6b7280);
}

/* Code */
.lexxy-content code {
  font-family: var(--lexxy-font-mono, ui-monospace, monospace);
  font-size: 0.875em;
  background: var(--lexxy-color-code-bg, #f5f5f5);
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
}
.lexxy-content pre {
  background: var(--lexxy-color-code-bg, #f5f5f5);
  border-radius: 8px;
  padding: 1rem;
  margin: 1rem 0;
  overflow-x: auto;
}
.lexxy-content pre code {
  background: none;
  padding: 0;
  border-radius: 0;
}

/* Horizontal rule */
.lexxy-content hr {
  border: none;
  border-top: 1px solid var(--color-border, #e5e7eb);
  margin: 2rem 0;
}

/* Images / attachments */
.lexxy-content .attachment {
  margin: 1rem 0;
  max-width: 100%;
}
.lexxy-content .attachment img {
  max-width: 100%;
  height: auto;
  border-radius: 6px;
}
.lexxy-content .attachment__caption {
  font-size: 0.875rem;
  color: var(--color-ink-muted, #6b7280);
  margin-top: 0.5rem;
  text-align: center;
}
```

### Trix Content Styles

Same patterns apply, but use `.trix-content` instead of `.lexxy-content`:

```css
.trix-content { /* ... same structure ... */ }
```

---

## N+1 Query Prevention

### The Problem

Each `has_rich_text` declaration creates a polymorphic association. Accessing it triggers a query:

```ruby
# Generates N+1 queries
@articles = Article.all
@articles.each do |article|
  article.content.to_s  # Query per article!
end
```

### The Solution

```ruby
# In controller
@articles = Article.with_rich_text_content

# With attachments too
@articles = Article.with_rich_text_content_and_embeds

# Multiple rich text fields
@products = Product
  .with_rich_text_description_and_embeds
  .with_rich_text_specifications
```

### In Scopes

```ruby
class Article < ApplicationRecord
  has_rich_text :content

  scope :for_index, -> {
    with_rich_text_content
    .order(created_at: :desc)
    .limit(20)
  }

  scope :for_feed, -> {
    with_rich_text_content_and_embeds
    .includes(:author)
    .order(published_at: :desc)
  }
end
```

### In Associations

```ruby
class Author < ApplicationRecord
  has_many :articles

  def recent_articles
    articles.with_rich_text_content.order(created_at: :desc).limit(5)
  end
end
```

---

## Content Security

### Default Sanitization

Action Text uses Rails' built-in HTML sanitizer. By default, it allows:
- Text formatting: `<strong>`, `<em>`, `<del>`, `<u>`
- Structure: `<p>`, `<br>`, `<div>`, `<h1>`-`<h6>`
- Lists: `<ul>`, `<ol>`, `<li>`
- Links: `<a href="...">`
- Images: `<img src="...">`
- Blockquotes: `<blockquote>`
- Code: `<pre>`, `<code>`
- Tables: `<table>`, `<tr>`, `<td>`, `<th>`, etc.
- Figures: `<figure>`, `<figcaption>`
- Action Text attachments: `<action-text-attachment>`

### Stripped Tags (Not Allowed)

These are **always removed**:
- `<script>`, `<style>`, `<iframe>`, `<object>`, `<embed>`
- `<form>`, `<input>`, `<button>`, `<select>`, `<textarea>`
- Event handler attributes: `onclick`, `onload`, `onerror`, etc.

### Extending Allowed Tags

```ruby
# config/application.rb
config.after_initialize do
  # Add iframe support (for video embeds) — USE WITH CAUTION
  ActionText::ContentHelper.allowed_tags.add("iframe")
  ActionText::ContentHelper.allowed_attributes.add("src", "frameborder", "allowfullscreen", "width", "height")
end
```

⚠️ **Security Warning:** Expanding the allow-list can introduce XSS vulnerabilities. Only add tags you fully trust. Prefer custom attachables for embedding third-party content.

---

## API / Headless Usage

### Creating Rich Text via API

```ruby
# In an API controller
class Api::ArticlesController < ApiController
  def create
    article = Article.create!(
      title: params[:title],
      content: params[:content]  # Accepts HTML string
    )
    render json: { id: article.id, content_html: article.content.to_s }
  end
end
```

### Attachment Upload Endpoint

```ruby
# For frontend apps that need to upload files separately
class Api::AttachmentsController < ApiController
  def create
    blob = ActiveStorage::Blob.create_and_upload!(
      io: params[:file],
      filename: params[:file].original_filename,
      content_type: params[:file].content_type
    )
    render json: {
      attachable_sgid: blob.attachable_sgid,
      url: url_for(blob)
    }
  end
end
```

### Embedding Uploaded Attachments

```html
<!-- Frontend inserts this into the rich text HTML -->
<action-text-attachment sgid="BAh7CEkiCG..."></action-text-attachment>
```

---

## Testing Patterns

### Model Tests

```ruby
require "test_helper"

class ArticleTest < ActiveSupport::TestCase
  test "stores and retrieves rich text content" do
    article = Article.create!(title: "Test", content: "<h1>Hello</h1><p>World</p>")
    article.reload

    assert_equal "Hello\nWorld", article.content.to_plain_text.strip
    assert_includes article.content.to_s, "<h1>Hello</h1>"
  end

  test "content can be blank" do
    article = Article.create!(title: "Test")
    assert article.content.blank?
  end

  test "content can contain attachments" do
    article = articles(:with_content)
    # Use with_rich_text_content_and_embeds to test attachment loading
    loaded = Article.with_rich_text_content_and_embeds.find(article.id)
    assert loaded.content.present?
  end

  test "custom validation on rich text length" do
    article = Article.new(title: "Test")
    article.content = "x" * 20_000  # Exceeds max
    refute article.valid?
    assert_includes article.errors[:content], "is too long (maximum 10,000 characters)"
  end
end
```

### Request Tests

```ruby
require "test_helper"

class ArticlesRequestTest < ActionDispatch::IntegrationTest
  setup do
    @user = users(:active_user)
    sign_in @user
  end

  test "POST /articles with rich text content" do
    assert_difference "Article.count", 1 do
      post articles_path, params: {
        article: {
          title: "Test Article",
          content: "<h1>Hello</h1><p>This is <strong>rich</strong> text.</p>"
        }
      }
    end

    article = Article.last
    assert_equal "Test Article", article.title
    assert_includes article.content.to_plain_text, "rich"
    assert_includes article.content.to_s, "<strong>rich</strong>"
  end

  test "PATCH /articles/:id updates rich text" do
    article = articles(:published)

    patch article_path(article), params: {
      article: { content: "<p>Updated content</p>" }
    }

    assert_equal "Updated content", article.reload.content.to_plain_text.strip
  end
end
```

### System Tests (Editor Interaction)

```ruby
require "application_system_test_case"

class RichTextEditorTest < ApplicationSystemTestCase
  setup do
    sign_in users(:active_user)
  end

  # Trix editor
  test "writes content in Trix editor" do
    visit new_article_path

    fill_in "Title", with: "My Article"

    find("trix-editor").click
    find("trix-editor").set("Hello from the editor")

    click_on "Create Article"
    assert_text "Hello from the editor"
  end

  # Lexxy editor
  test "writes content in Lexxy editor" do
    visit new_article_path

    fill_in "Title", with: "My Article"

    find("lexxy-editor").click
    find("lexxy-editor").set("Hello from Lexxy")

    click_on "Create Article"
    assert_text "Hello from Lexxy"
  end

  test "attaches image via drag and drop" do
    visit new_article_path

    # Attach file to the hidden file input
    find("trix-editor").click
    attach_file file_fixture("test_image.png"), make_visible: true

    click_on "Create Article"
    assert_selector "img.attachment__image"
  end
end
```

### Fixture with Rich Text

```yaml
# test/fixtures/articles.yml
published:
  title: "Published Article"
  author: active_user

with_content:
  title: "Article with Content"
  author: active_user

# test/fixtures/action_text/rich_texts.yml
published_content:
  record: published (Article)
  name: content
  body: "<p>This is published content with <strong>formatting</strong>.</p>"

with_content_body:
  record: with_content (Article)
  name: content
  body: "<h1>Heading</h1><p>Paragraph with <a href='https://example.com'>link</a>.</p>"
```

### Testing Attachables

```ruby
require "test_helper"

class UserMentionTest < ActiveSupport::TestCase
  test "user can be embedded as attachable" do
    user = users(:active_user)
    sgid = user.attachable_sgid

    html = %(<action-text-attachment sgid="#{sgid}"></action-text-attachment>)
    content = ActionText::Content.new(html)

    assert_equal [user], content.attachables
  end

  test "missing user renders fallback partial" do
    sgid = users(:active_user).attachable_sgid
    users(:active_user).destroy!

    html = %(<action-text-attachment sgid="#{sgid}"></action-text-attachment>)
    content = ActionText::Content.new(html)

    # Verify it doesn't raise when the record is missing
    assert_nothing_raised { content.to_s }
  end
end
```

---

## Migration & UUID Support

### Standard Migration

Generated by `action_text:install`:

```ruby
class CreateActionTextTables < ActiveRecord::Migration[7.1]
  def change
    create_table :action_text_rich_texts do |t|
      t.string     :name, null: false
      t.text       :body
      t.references :record, null: false, polymorphic: true, index: false
      t.timestamps
      t.index [:record_type, :record_id, :name], name: "index_action_text_rich_texts_uniqueness", unique: true
    end
  end
end
```

### UUID Migration

If your app uses UUID primary keys, modify the generated migration:

```ruby
class CreateActionTextTables < ActiveRecord::Migration[7.1]
  def change
    create_table :action_text_rich_texts, id: :uuid do |t|
      t.string     :name, null: false
      t.text       :body
      t.references :record, null: false, polymorphic: true, index: false, type: :uuid
      t.timestamps
      t.index [:record_type, :record_id, :name], name: "index_action_text_rich_texts_uniqueness", unique: true
    end
  end
end
```

**Important:** ALL models using `has_rich_text` must use UUID primary keys if the `action_text_rich_texts` table uses UUID for `record_id`.

### Renaming a Model

If you rename a model that has `has_rich_text`, update the polymorphic type:

```ruby
class RenamePostToArticle < ActiveRecord::Migration[7.1]
  def up
    ActionText::RichText.where(record_type: "Post").update_all(record_type: "Article")
  end

  def down
    ActionText::RichText.where(record_type: "Article").update_all(record_type: "Post")
  end
end
```

---

## Troubleshooting

### Images Not Rendering in Editor

**Cause:** Missing `libvips` dependency.

```bash
# Debian/Ubuntu
sudo apt install libvips

# macOS
brew install vips
```

### Editor Not Appearing

**Cause:** Missing JavaScript imports.

Check `app/javascript/application.js`:
```js
import "trix"
import "@rails/actiontext"
```

Or for importmaps, check `config/importmap.rb`:
```ruby
pin "trix"
pin "@rails/actiontext", to: "actiontext.esm.js"
```

### Content Renders as Plain Text (Not HTML)

**Cause:** Using `@article.content.body` instead of `@article.content`.

```erb
<%# WRONG: renders raw HTML as text %>
<%= @article.content.body %>

<%# CORRECT: renders sanitized HTML %>
<%= @article.content %>
```

### Content Looks Different in Editor vs Rendered View

**Cause:** Editor and rendered content use different stylesheets.

The editor has its own internal styles. The rendered content uses your `.trix-content` or `.lexxy-content` styles. Keep both in sync.

### Styles Not Applying to Lexxy Editor

**Cause:** Wrong stylesheet load order.

```erb
<%# Lexxy CSS must load FIRST, then your overrides %>
<%= stylesheet_link_tag "lexxy" %>
<%= stylesheet_link_tag :app %>
```

### N+1 Queries in Views

**Cause:** Not preloading rich text associations.

```ruby
# Controller
@articles = Article.with_rich_text_content_and_embeds
```

### ActionText::RichText Record Not Found

**Cause:** `action_text:install` was never run, or migrations weren't executed.

```bash
bin/rails action_text:install
bin/rails db:migrate
```

### Turbo/Stimulus Conflicts with Editor

**Cause:** Turbo replacing the DOM removes the editor instance.

```erb
<%# Wrap editor in a Turbo permanent frame %>
<div id="editor-container" data-turbo-permanent>
  <%= form.rich_text_area :content %>
</div>
```

### File Uploads Fail Silently

**Cause:** Active Storage not configured or missing service.

Check `config/storage.yml` and `config/environments/*.rb`:
```ruby
config.active_storage.service = :local  # or :amazon, :google, etc.
```

### Blank Content After Save

**Cause:** Strong parameters not permitting the rich text attribute.

```ruby
# Ensure the attribute is permitted
params.expect(article: [:title, :content])
```
