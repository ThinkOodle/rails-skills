# Rails Caching Reference

Detailed patterns, edge cases, and advanced configuration for Rails caching.

## Table of Contents

- [Fragment Caching Deep Dive](#fragment-caching-deep-dive)
- [Russian Doll Caching Patterns](#russian-doll-caching-patterns)
- [Low-Level Caching Patterns](#low-level-caching-patterns)
- [Conditional GET (HTTP Caching)](#conditional-get-http-caching)
- [Cache Store Comparison](#cache-store-comparison)
- [Solid Cache Configuration](#solid-cache-configuration)
- [Cache Key Design](#cache-key-design)
- [Managing Dependencies](#managing-dependencies)
- [Counter Caches](#counter-caches)
- [Testing Caching](#testing-caching)
- [Performance Patterns](#performance-patterns)
- [Debugging & Troubleshooting](#debugging--troubleshooting)

---

## Fragment Caching Deep Dive

### How Cache Keys Are Generated

When you write `<% cache product do %>`, Rails builds a key from:

1. **Prefix:** `views/`
2. **Template digest:** MD5 of the template content (and its dependencies)
3. **Record key:** `product.cache_key_with_version` → `products/1-20260101120000000000`

Full key example:
```
views/products/_product:bea67108094918eeba42cd4a6e786901/products/1-20260101120000000000
```

### Composite Cache Keys

Pass an array to scope the cache by multiple dimensions:

```erb
<%# Scoped by user, locale, and record %>
<% cache [current_user, I18n.locale, product] do %>
  ...
<% end %>

<%# Scoped by role (not individual user) %>
<% cache [current_user.role, product] do %>
  ...
<% end %>

<%# Scoped by a custom version string %>
<% cache [product, "v2"] do %>
  ...
<% end %>
```

### Conditional Fragment Caching

```erb
<%# Only cache for non-admin users %>
<% cache_if !current_user&.admin?, product do %>
  <%= render product %>
<% end %>

<%# Cache unless in preview mode %>
<% cache_unless @preview_mode, product do %>
  <%= render product %>
<% end %>
```

### Shared Partial Caching Across MIME Types

A partial used by both HTML and JS responses:

```ruby
# Works for both HTML and JS requests
render(partial: "hotels/hotel", collection: @hotels, cached: true)
# Loads hotels/hotel.erb

# Force HTML format in any context
render(partial: "hotels/hotel", collection: @hotels, formats: :html, cached: true)
# Loads hotels/hotel.html.erb even from JS templates
```

### Collection Caching with Custom Keys

```erb
<%# Default: uses each record's cache_key_with_version %>
<%= render partial: "products/product", collection: @products, cached: true %>

<%# Custom key with locale prefix %>
<%= render partial: "products/product",
           collection: @products,
           cached: ->(product) { [I18n.locale, product] } %>

<%# Custom key with current user role %>
<%= render partial: "products/product",
           collection: @products,
           cached: ->(product) { [current_user.role, product] } %>
```

**How multi-fetch works:** Rails calls `read_multi` on the cache store, fetching all entries in a single round trip. Only missed entries are rendered and written back. This is dramatically faster than individual `cache` calls in a loop.

---

## Russian Doll Caching Patterns

### Basic Two-Level Nesting

```erb
<%# app/views/categories/show.html.erb %>
<% cache @category do %>
  <h2><%= @category.name %></h2>
  <%= render partial: "products/product", collection: @category.products, cached: true %>
<% end %>
```

```ruby
class Product < ApplicationRecord
  belongs_to :category, touch: true
end
```

### Three-Level Nesting

```erb
<%# app/views/stores/show.html.erb %>
<% cache @store do %>
  <% @store.categories.each do |category| %>
    <% cache category do %>
      <h3><%= category.name %></h3>
      <% category.products.each do |product| %>
        <% cache product do %>
          <%= render product %>
        <% end %>
      <% end %>
    <% end %>
  <% end %>
<% end %>
```

```ruby
class Product < ApplicationRecord
  belongs_to :category, touch: true
end

class Category < ApplicationRecord
  belongs_to :store, touch: true
  has_many :products
end
```

### Touch Propagation Patterns

```ruby
# Simple touch
belongs_to :post, touch: true

# Touch with specific timestamp column
belongs_to :post, touch: :comments_updated_at

# Conditional touch
after_save :touch_post, if: :published?
def touch_post
  post.touch
end

# Manual touch (useful in service objects)
@post.touch  # Updates updated_at
@post.touch(:reviewed_at)  # Updates a specific column
```

### When Touch Gets Expensive

If updating a child record touches hundreds of parents, consider:

```ruby
# Instead of touch cascading through everything:
class Comment < ApplicationRecord
  belongs_to :post, touch: true  # This touches post → author → ...
end

# Use targeted cache expiration:
class Comment < ApplicationRecord
  belongs_to :post

  after_commit :expire_post_cache, on: [:create, :update, :destroy]

  private

  def expire_post_cache
    Rails.cache.delete("post_comments_count/#{post_id}")
    post.touch  # Only touch one level
  end
end
```

---

## Low-Level Caching Patterns

### Rails.cache.fetch Patterns

```ruby
# Basic fetch with expiration
Rails.cache.fetch("site_stats", expires_in: 1.hour) do
  {
    total_users: User.count,
    total_orders: Order.count,
    revenue_today: Order.today.sum(:total)
  }
end

# Fetch with record-based key (auto-invalidates)
Rails.cache.fetch("#{product.cache_key_with_version}/full_description") do
  product.description_html  # Expensive markdown rendering
end

# Fetch with race condition TTL (prevents thundering herd)
Rails.cache.fetch("popular_products", expires_in: 1.hour, race_condition_ttl: 10.seconds) do
  Product.popular.limit(10).to_a.map(&:id)
end

# Force cache refresh
Rails.cache.fetch("key", force: true) { compute_value }
```

### race_condition_ttl Explained

When a cached value expires, multiple processes may try to regenerate it simultaneously (thundering herd). `race_condition_ttl` extends the stale entry briefly so only one process regenerates:

```ruby
Rails.cache.fetch("expensive_report", expires_in: 6.hours, race_condition_ttl: 30.seconds) do
  Report.generate_daily  # Takes 10 seconds
end
```

### fetch_multi for Batch Operations

```ruby
# Fetch multiple keys at once
results = Rails.cache.fetch_multi("user_1_stats", "user_2_stats", "user_3_stats", expires_in: 1.hour) do |key|
  user_id = key.split("_")[1]
  User.find(user_id).compute_stats
end

# With dynamic keys
keys = products.map { |p| "product_#{p.id}_price" }
prices = Rails.cache.fetch_multi(*keys, expires_in: 12.hours) do |key|
  product_id = key.split("_")[1]
  Competitor::API.find_price(product_id)
end
```

### Cache Write Options

```ruby
Rails.cache.write("key", value,
  expires_in: 1.hour,          # TTL
  expires_at: 1.hour.from_now, # Absolute expiration time
  namespace: "v2",             # Key namespace
  compress: true,              # Compress values (default for > 1KB)
  compress_threshold: 256      # Compress if value > 256 bytes
)
```

### Increment/Decrement (Atomic Operations)

```ruby
# Atomic counter operations (supported by Memcached, Redis, Solid Cache)
Rails.cache.increment("page_views/#{page.id}")
Rails.cache.decrement("remaining_credits/#{user.id}")

# With initial value and expiration
Rails.cache.increment("api_calls/#{api_key}/#{Date.today}", 1,
  initial: 0,
  expires_in: 1.day
)
```

### What NOT to Cache

```ruby
# ❌ Never cache AR objects
Rails.cache.fetch("user") { User.find(1) }

# ❌ Never cache Proc or Lambda
Rails.cache.fetch("fn") { -> { compute } }

# ❌ Never cache objects with IO handles
Rails.cache.fetch("file") { File.open("data.csv") }

# ❌ Never cache request-scoped data
Rails.cache.fetch("current_request") { request.url }

# ✅ Cache primitives: strings, numbers, arrays, hashes
Rails.cache.fetch("ids") { User.pluck(:id) }
Rails.cache.fetch("count") { Order.count }
Rails.cache.fetch("stats") { { total: 100, average: 42.5 } }
```

---

## Conditional GET (HTTP Caching)

### stale? vs fresh_when

| Method | Use When |
|--------|----------|
| `stale?` | You have custom response logic (respond_to, special rendering) |
| `fresh_when` | Default template rendering is fine |

### stale? Patterns

```ruby
class ProductsController < ApplicationController
  def show
    @product = Product.find(params[:id])

    if stale?(@product)
      respond_to do |format|
        format.html
        format.json { render json: @product }
      end
    end
  end

  # With explicit options
  def show
    @product = Product.find(params[:id])

    if stale?(
      last_modified: @product.updated_at.utc,
      etag: @product.cache_key_with_version,
      public: true  # Allow CDN/proxy caching
    )
      respond_to do |format|
        format.html
        format.json { render json: @product }
      end
    end
  end
end
```

### fresh_when Patterns

```ruby
class ProductsController < ApplicationController
  # Simplest form — pass the record
  def show
    @product = Product.find(params[:id])
    fresh_when @product
  end

  # For collections — uses max updated_at
  def index
    @products = Product.all
    fresh_when @products
  end

  # With explicit options
  def show
    @product = Product.find(params[:id])
    fresh_when(
      last_modified: @product.published_at.utc,
      etag: @product,
      public: false  # Private (default) — only browser caches
    )
  end
end
```

### http_cache_forever

For truly static content:

```ruby
class PagesController < ApplicationController
  def terms
    http_cache_forever(public: true) do
      render
    end
  end
end
```

**Warning:** Browsers and proxies will cache indefinitely. The only way to invalidate is changing the URL or user clearing their cache.

### Strong vs Weak ETags

Rails generates **weak ETags** by default (prefixed with `W/`). Weak ETags mean "semantically equivalent" — the response may differ in whitespace or encoding but the content is the same.

```ruby
# Weak ETag (default) — good for most cases
fresh_when etag: @product

# Strong ETag — exact byte-for-byte match required
# Use for: Range requests, CDNs that require strong ETags (Akamai)
fresh_when strong_etag: @product

# Set directly on response
response.strong_etag = response.body
```

### Public vs Private Caching

```ruby
# Private (default) — only the user's browser caches
fresh_when @product  # Cache-Control: private

# Public — CDNs and proxies can cache too
fresh_when @product, public: true  # Cache-Control: public

# Set cache control headers directly
expires_in 1.hour, public: true
expires_in 30.minutes, public: true, stale_while_revalidate: 60
expires_now  # Cache-Control: no-cache
```

---

## Cache Store Comparison

| Feature | Solid Cache | Redis | Memcached | Memory | File |
|---------|------------|-------|-----------|--------|------|
| **Infrastructure** | None (DB) | Redis server | Memcached server | None | None |
| **Shared across processes** | ✅ | ✅ | ✅ | ❌ | ✅ (same host) |
| **Persistence** | ✅ (DB) | Configurable | ❌ | ❌ | ✅ |
| **Eviction** | FIFO | LRU/LFU | LRU | LRU | Manual |
| **Max size** | Disk (large) | RAM | RAM | Configured | Disk |
| **delete_matched** | ✅ | ✅ | ❌ | ✅ | ✅ |
| **increment/decrement** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Encryption** | ✅ | ❌ | ❌ | N/A | ❌ |
| **Sharding** | ✅ | Manual | ✅ | N/A | N/A |
| **Speed** | ~1ms (SSD) | ~0.1ms | ~0.1ms | ~0.01ms | ~1ms |
| **Rails default (8+)** | ✅ Prod | — | — | ✅ Dev | — |

### When to Use Which

- **Solid Cache:** Default choice. No extra infra, large capacity, good enough speed for most apps.
- **Redis:** When you already run Redis (Sidekiq) and need sub-millisecond reads, or LRU eviction.
- **Memcached:** Legacy apps, multi-server setups, when you need distributed caching without Redis.
- **Memory:** Development and test only. Single-process apps.
- **File:** Low-traffic sites, development, or when you need persistence without a DB.

---

## Solid Cache Configuration

### Basic Setup

```yaml
# config/database.yml
production:
  primary:
    <<: *default
    database: storage/production.sqlite3
  cache:
    <<: *default
    database: storage/production_cache.sqlite3
    migrations_paths: db/cache_migrate
```

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store
```

### Tuning cache.yml

```yaml
# config/cache.yml
default: &default
  store_options:
    max_age: <%= 60.days.to_i %>     # Max age of any cache entry
    max_size: <%= 256.megabytes %>    # Total cache size limit
    namespace: <%= Rails.env %>       # Key namespace

production:
  <<: *default
  store_options:
    max_age: <%= 90.days.to_i %>
    max_size: <%= 1.gigabyte %>
```

### Sharding for Scale

```yaml
# config/database.yml
production:
  cache_shard1:
    database: cache1_production
    host: cache1-db
  cache_shard2:
    database: cache2_production
    host: cache2-db

# config/cache.yml
production:
  databases: [cache_shard1, cache_shard2]
```

### Encryption

```yaml
# config/cache.yml
production:
  encrypt: true
```

Requires Active Record Encryption to be set up in the app.

### Expiry Behavior

Solid Cache tracks writes with a counter. At 50% of `expiry_batch_size`, a background thread purges old entries. If you prefer background jobs:

```yaml
# config/cache.yml
production:
  expiry_method: :job  # Default is :thread
```

---

## Cache Key Design

### Active Record Cache Keys

```ruby
product = Product.find(1)
product.cache_key                # "products/1"
product.cache_key_with_version   # "products/1-20260101120000000000"
product.cache_version            # "20260101120000000000"
```

### Custom Cache Keys

```ruby
class Dashboard
  def cache_key
    "dashboards/#{user_id}/#{Date.today}"
  end
end

# Or use arrays — Rails joins them with "/"
Rails.cache.fetch([user.id, "dashboard", Date.today]) { ... }
```

### Namespace Your Caches

```ruby
# Global namespace in config
config.cache_store = :solid_cache_store, { namespace: "myapp-v1" }

# Per-call namespace
Rails.cache.fetch("stats", namespace: "reports") { ... }
# Actual key: "reports/stats"
```

### Versioning Cache Keys for Schema Changes

When your cached data structure changes, bump the key:

```ruby
# Before: cached a simple string
Rails.cache.fetch("#{product.cache_key_with_version}/description") { ... }

# After: cached a hash — old cached strings will cause errors
# Bump the key version:
Rails.cache.fetch("#{product.cache_key_with_version}/description/v2") { ... }
```

---

## Managing Dependencies

### Implicit Dependencies

Rails auto-detects these render calls for cache digest computation:

```erb
<%= render partial: "comments/comment", collection: commentable.comments %>
<%= render "comments/comments" %>
<%= render @topic %>           <%# → "topics/topic" %>
<%= render topics %>           <%# → "topics/topic" %>
```

### Explicit Dependencies

When Rails can't detect the dependency (helpers, dynamic partials):

```erb
<%# Template Dependency: todolists/todolist %>
<%= render_sortable_todolists @project.todolists %>

<%# Wildcard dependency %>
<%# Template Dependency: events/* %>
<%= render_categorizable_events @person.events %>
```

### Collection Template Annotation

When a collection partial doesn't start with `cache`:

```erb
<%# Template Collection: notification %>
<% my_helper_that_calls_cache(some_arg, notification) do %>
  <%= notification.name %>
<% end %>
```

### External/Helper Dependencies

When changing a helper should bust the cache, add a version comment:

```erb
<%# Helper Dependency Updated: 2026-03-01 %>
<% cache product do %>
  <%= render_stars(product.rating) %>
<% end %>
```

Change the date whenever you update the helper.

---

## Counter Caches

Counter caches avoid `COUNT(*)` queries. Not strictly "caching" but a key performance pattern.

```ruby
# Migration
add_column :posts, :comments_count, :integer, default: 0, null: false

# Model
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: true
end

# Usage — no SQL COUNT query
@post.comments_count  # Reads the column directly

# Reset if data gets out of sync
Comment.counter_culture_fix_counts  # If using counter_culture gem
# Or manually:
Post.find_each do |post|
  Post.reset_counters(post.id, :comments)
end
```

### Custom Counter Cache Column

```ruby
class Comment < ApplicationRecord
  belongs_to :post, counter_cache: :total_comments
end
```

### Conditional Counter Cache (via counter_culture gem)

```ruby
# Gemfile
gem "counter_culture"

class Comment < ApplicationRecord
  belongs_to :post
  counter_culture :post,
    column_name: proc { |comment| comment.approved? ? "approved_comments_count" : nil }
end
```

---

## Testing Caching

### Test Cache Invalidation, Not Rails Internals

```ruby
class ProductCacheTest < ActiveSupport::TestCase
  test "updating product expires its cache" do
    product = products(:widget)

    # Write to cache
    Rails.cache.write("product_#{product.id}", product.name)

    # Verify cache exists
    assert_equal "Widget", Rails.cache.read("product_#{product.id}")
  end

  test "touch propagation works through association chain" do
    category = categories(:electronics)
    product = products(:widget)

    original = category.updated_at
    product.update!(name: "Updated Widget")

    assert_operator category.reload.updated_at, :>, original
  end
end
```

### Test Conditional GET

```ruby
class ProductsControllerTest < ActionDispatch::IntegrationTest
  test "returns 304 for unchanged resource" do
    product = products(:widget)

    # First request — full response
    get product_path(product)
    assert_response :success
    etag = response.headers["ETag"]
    last_modified = response.headers["Last-Modified"]

    # Second request with ETag — 304
    get product_path(product), headers: { "HTTP_IF_NONE_MATCH" => etag }
    assert_response :not_modified

    # Second request with Last-Modified — 304
    get product_path(product), headers: { "HTTP_IF_MODIFIED_SINCE" => last_modified }
    assert_response :not_modified
  end

  test "returns 200 after resource update" do
    product = products(:widget)

    get product_path(product)
    etag = response.headers["ETag"]

    product.update!(name: "New Name")

    get product_path(product), headers: { "HTTP_IF_NONE_MATCH" => etag }
    assert_response :success
  end
end
```

### Test Environment Configuration

```ruby
# config/environments/test.rb

# Option A: Disable caching entirely (simpler tests)
config.cache_store = :null_store

# Option B: Use memory store (test cache behavior)
config.cache_store = :memory_store

# Clear cache between tests
# test/test_helper.rb
class ActiveSupport::TestCase
  teardown do
    Rails.cache.clear
  end
end
```

---

## Performance Patterns

### Avoid N+1 Cache Calls

```ruby
# BAD: N+1 cache reads
@products.each do |product|
  Rails.cache.fetch("product_#{product.id}/price") { product.compute_price }
end

# GOOD: Batch cache read
keys = @products.map { |p| "product_#{p.id}/price" }
Rails.cache.fetch_multi(*keys, expires_in: 1.hour) do |key|
  id = key.split("_")[1].split("/").first
  Product.find(id).compute_price
end
```

### Cache Warming

Pre-populate caches after deploy or data migration:

```ruby
# lib/tasks/cache.rake
namespace :cache do
  desc "Warm product caches"
  task warm_products: :environment do
    Product.find_each do |product|
      Rails.cache.fetch("#{product.cache_key_with_version}/full", expires_in: 1.day) do
        ProductSerializer.new(product).as_json
      end
    end
  end
end
```

### Fragment Cache + eager_load

```ruby
# Controller
def show
  @post = Post.includes(:comments, :author).find(params[:id])
  fresh_when @post
end
```

Even with fragment caching, eager-load associations for cache misses.

### Cache Stampede Prevention

Multiple patterns to prevent thundering herd:

```ruby
# 1. race_condition_ttl (built-in)
Rails.cache.fetch("key", expires_in: 1.hour, race_condition_ttl: 10.seconds) { ... }

# 2. Stale-while-revalidate (HTTP level)
expires_in 1.hour, public: true, stale_while_revalidate: 60.seconds

# 3. Background refresh (via Solid Queue or Sidekiq)
class CacheRefreshJob < ApplicationJob
  def perform(cache_key)
    value = expensive_computation
    Rails.cache.write(cache_key, value, expires_in: 1.hour)
  end
end
```

---

## Debugging & Troubleshooting

### Check If Caching Is Enabled

```ruby
# In console
Rails.application.config.action_controller.perform_caching  # Fragment caching
Rails.cache.class  # What store is active

# Toggle in development
# bin/rails dev:cache
```

### View Cache Hits/Misses in Logs

```
# In development.log:
Read fragment views/products/1-20260101/abc123 (0.5ms)        # HIT
Write fragment views/products/1-20260101/abc123 (1.2ms)       # MISS → WRITE
```

### Inspect Cache Contents

```ruby
# Read a specific key
Rails.cache.read("key")

# Check existence
Rails.cache.exist?("key")

# For Solid Cache — query the database directly
SolidCache::Entry.count
SolidCache::Entry.where("key_hash LIKE ?", "%products%").count
```

### Clear All Caches

```ruby
Rails.cache.clear                          # Clear cache store
ActionView::Digestor.cache.clear           # Clear template digest cache
```

```bash
bin/rails tmp:cache:clear                  # Clear file-based cache
```

### Common Stale Cache Symptoms

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Updated record shows old data | Missing `touch: true` | Add `touch: true` to `belongs_to` |
| Template change not reflected | Template digest not updating | Check for undetected dependencies |
| Different users see same content | Missing user in cache key | Add `current_user` to cache key array |
| Cache never expires | No `expires_in` set | Always set expiration on `fetch`/`write` |
| Slow first page load after deploy | Cache cold start | Implement cache warming |
| All caches expire at once | Same `expires_in` for everything | Jitter expirations |

### Jittering Expirations

Prevent cache stampedes by adding randomness:

```ruby
# Instead of every cache expiring at exactly 1 hour:
expires_in: (55..65).to_a.sample.minutes

# Or in a helper:
def jittered_ttl(base, jitter: 0.1)
  variation = base * jitter
  base + rand(-variation..variation)
end

Rails.cache.fetch("key", expires_in: jittered_ttl(1.hour)) { ... }
```

---

## SQL Caching (Automatic)

Rails automatically caches SQL query results within a single request. No configuration needed.

```ruby
class ProductsController < ApplicationController
  def index
    @products = Product.all        # Hits DB
    @count = Product.all.count     # Uses cached result (same query)
  end
end
```

**Key facts:**
- Scoped to a single request (destroyed at end of action)
- Stores query string → result set mapping
- Each retrieval still instantiates new AR objects
- For persistent caching, use low-level caching instead

---

## Redis Cache Store Advanced Config

### Dedicated Cache Instance

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_CACHE_URL"],  # Separate from Sidekiq Redis!
  connect_timeout: 5,
  read_timeout: 0.2,
  write_timeout: 0.2,
  reconnect_attempts: 2,
  error_handler: ->(method:, returning:, exception:) {
    Rails.logger.warn("Redis cache error: #{exception.message}")
    Sentry.capture_exception(exception, level: "warning")
  }
}
```

### Redis Server Config for Caching

```
# redis.conf for cache-only instance
maxmemory 2gb
maxmemory-policy allkeys-lfu  # Redis 4+ (best for caching)
# maxmemory-policy allkeys-lru  # Redis 3
```

### Connection Pooling

```ruby
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  pool: { size: 16, timeout: 2 }  # Default: size=5, timeout=5
}

# Disable pooling
config.cache_store = :redis_cache_store, {
  url: ENV["REDIS_URL"],
  pool: false
}
```

---

## Memcached Configuration

```ruby
# Gemfile
gem "dalli"

# Config
config.cache_store = :mem_cache_store, "cache-1.example.com", "cache-2.example.com"

# With options
config.cache_store = :mem_cache_store, "cache-1.example.com", {
  pool: { size: 10, timeout: 2 },
  compress: true,
  serializer: :json
}
```

**Limitations vs Redis:**
- No `delete_matched` (no pattern scanning)
- 1MB value size limit (default)
- No persistence
- No pub/sub

---

## Cache Namespacing Strategies

### Per-Deploy Cache Busting

```ruby
# Bust all caches on deploy by including git SHA
config.cache_store = :solid_cache_store, {
  namespace: "myapp-#{ENV['GIT_SHA'] || 'dev'}"
}
```

### Per-Feature Namespacing

```ruby
# Isolate different features
Rails.cache.fetch("stats", namespace: "analytics", expires_in: 1.hour) { ... }
Rails.cache.fetch("feed", namespace: "social", expires_in: 5.minutes) { ... }

# Clear only one namespace
Rails.cache.delete_matched("analytics/*")
```
