# Active Record Querying — Reference

Detailed patterns, edge cases, and advanced examples. Companion to `SKILL.md`.

---

## Table of Contents

- [Finder Methods In Depth](#finder-methods-in-depth)
- [Where Conditions — Full Reference](#where-conditions--full-reference)
- [Ordering](#ordering)
- [Select and Pluck](#select-and-pluck)
- [Grouping and Aggregations](#grouping-and-aggregations)
- [Joins — Complete Guide](#joins--complete-guide)
- [Eager Loading — Deep Dive](#eager-loading--deep-dive)
- [Scopes — Patterns and Pitfalls](#scopes--patterns-and-pitfalls)
- [Batching — Full API](#batching--full-api)
- [Enums — Query Patterns](#enums--query-patterns)
- [Raw SQL — Safe Patterns](#raw-sql--safe-patterns)
- [Existence and Calculations](#existence-and-calculations)
- [Locking](#locking)
- [Overriding and Resetting Conditions](#overriding-and-resetting-conditions)
- [Method Chaining Patterns](#method-chaining-patterns)
- [Performance Patterns](#performance-patterns)
- [Rails 8.1 Features](#rails-81-features)

---

## Finder Methods In Depth

### find — By Primary Key

```ruby
# Single record — raises ActiveRecord::RecordNotFound if missing
user = User.find(42)

# Multiple records — raises if ANY id is missing
users = User.find([1, 2, 3])
users = User.find(1, 2, 3)  # Same thing

# Composite primary keys
record = Model.find([store_id, id])
records = Model.find([[1, 8], [7, 15]])
```

**When to use:** You have an ID and the record MUST exist. Controller show/edit/update/destroy actions.

### find_by — By Attributes

```ruby
user = User.find_by(email: "alice@example.com")        # nil if not found
user = User.find_by!(email: "alice@example.com")       # raises if not found

# Multiple conditions
user = User.find_by(email: "alice@example.com", active: true)
```

**When to use:** Looking up by non-PK attributes. Use bang version when missing record is an error.

**Gotcha:** `find_by` does NOT guarantee order. If multiple records match, you get an arbitrary one. Add `.order(...)` before if you need deterministic results:

```ruby
# Deterministic
User.order(:created_at).find_by(role: :admin)
```

### first / last / take

```ruby
User.first              # ORDER BY id ASC LIMIT 1
User.last               # ORDER BY id DESC LIMIT 1
User.take               # LIMIT 1 (no ORDER — database decides)

User.first(5)           # First 5 records
User.last(3)            # Last 3 records

# With scopes
User.where(active: true).first   # First active user by PK
User.order(:name).first           # First alphabetically

# Bang versions — raise if no records
User.first!
User.last!
User.take!
```

### find_or_create_by / find_or_initialize_by

```ruby
# Creates if not found (runs validations — may return unsaved record)
user = User.find_or_create_by(email: "new@example.com")

# Bang version — raises on validation failure
user = User.find_or_create_by!(email: "new@example.com")

# With additional attributes for creation only
user = User.find_or_create_by(email: "new@example.com") do |u|
  u.name = "New User"
  u.role = :member
end

# Or with create_with
user = User.create_with(name: "New User", role: :member)
            .find_or_create_by(email: "new@example.com")

# Initialize without saving (like .new)
user = User.find_or_initialize_by(email: "maybe@example.com")
user.persisted?   # false if it was just built
user.new_record?  # true if it was just built
user.save         # Save when ready
```

**Race condition warning:** `find_or_create_by` is NOT atomic. Two concurrent requests can both fail the `find`, then both `create`. Add a unique index + rescue:

```ruby
User.find_or_create_by(email: "x@y.com")
rescue ActiveRecord::RecordNotUnique
  retry
end
```

---

## Where Conditions — Full Reference

### Hash Conditions

```ruby
# Equality
Book.where(title: "Dune")

# IN (array)
Book.where(id: [1, 2, 3])

# Range (BETWEEN)
Book.where(price: 10..50)

# Beginless range (<=)
Book.where(price: ..50)

# Endless range (>=)
Book.where(price: 10..)

# NULL
Book.where(deleted_at: nil)

# Boolean
Book.where(published: true)

# Association object
Book.where(author: author_instance)

# Nested hash (on joined/included tables)
Book.joins(:author).where(authors: { name: "Frank Herbert" })
```

### NOT Conditions

```ruby
Book.where.not(out_of_print: true)
Book.where.not(id: [1, 2, 3])    # NOT IN
Book.where.not(deleted_at: nil)   # IS NOT NULL
```

**Gotcha with nullable columns:**
```ruby
# If nullable_country is NULL for some records:
Customer.where.not(nullable_country: "UK")
# Does NOT return records where nullable_country IS NULL!
# NULL != "UK" evaluates to NULL (unknown), not true

# To include NULLs:
Customer.where.not(nullable_country: "UK").or(Customer.where(nullable_country: nil))
```

### OR Conditions

```ruby
User.where(role: :admin).or(User.where(role: :moderator))
# WHERE role = 'admin' OR role = 'moderator'

# Often cleaner with IN:
User.where(role: [:admin, :moderator])
```

### AND Conditions

```ruby
# Implicit AND — just chain where
User.where(active: true).where(verified: true)

# Explicit AND (for intersecting separate relations)
admins = User.where(role: :admin)
verified = User.where(verified: true)
admins.and(verified)
```

### String Conditions

```ruby
# Positional parameters
Book.where("price > ? AND price < ?", 10, 50)

# Named parameters
Book.where("price > :min AND price < :max", min: 10, max: 50)

# LIKE with sanitization
Book.where("title LIKE ?", "#{Book.sanitize_sql_like(query)}%")

# ILIKE (PostgreSQL case-insensitive)
Book.where("title ILIKE ?", "%#{Book.sanitize_sql_like(query)}%")
```

### Tuple Conditions (Composite Keys)

```ruby
Book.where([:author_id, :id] => [[15, 1], [15, 2]])
# WHERE (author_id = 15 AND id = 1) OR (author_id = 15 AND id = 2)
```

---

## Ordering

```ruby
# Symbol (ascending by default)
Book.order(:title)

# Hash (explicit direction)
Book.order(title: :asc)
Book.order(created_at: :desc)

# Multiple columns
Book.order(title: :asc, created_at: :desc)

# String (for complex expressions)
Book.order("LOWER(title) ASC")
Book.order(Arel.sql("FIELD(status, 'active', 'pending', 'archived')"))

# From joined tables
Book.includes(:author).order("authors.name ASC")
Book.includes(:author).order(authors: { name: :asc })

# Override previous order
Book.order(:title).reorder(:created_at)   # Only created_at

# Reverse
Book.order(:title).reverse_order          # title DESC

# Append
Book.order(:title).order(:created_at)     # title ASC, created_at ASC
```

**Important:** `order` with string args and `Arel.sql` can be SQL injection vectors if you interpolate user input. Always use hash syntax or whitelisting.

---

## Select and Pluck

### select — Returns AR Objects with Limited Attributes

```ruby
# Only load specific columns
Book.select(:id, :title)
Book.select("id, title")

# Computed columns
Book.select("title, LENGTH(title) AS title_length")
book.title_length  # Accessible as an attribute

# Distinct
Customer.select(:last_name).distinct
```

**Gotcha:** Accessing unselected columns raises `ActiveModel::MissingAttributeError`:
```ruby
book = Book.select(:title).first
book.isbn  # MissingAttributeError!
```

### reselect — Override Previous Select

```ruby
Book.select(:title, :isbn).reselect(:created_at)
# SELECT books.created_at FROM books
```

### pluck — Returns Raw Arrays

```ruby
Book.pluck(:title)                    # ["Dune", "1984", ...]
Book.pluck(:id, :title)              # [[1, "Dune"], [2, "1984"], ...]
Book.where(published: true).pluck(:id)

# From joined tables
Order.joins(:customer).pluck("orders.id, customers.email")
```

**Key differences from select:**
- `pluck` returns plain Ruby arrays, NOT AR objects
- `pluck` triggers the query immediately (can't chain further)
- `pluck` is significantly faster for extracting data
- `pluck` skips model instantiation and callbacks

```ruby
# DO: pluck then chain Ruby methods
ids = User.where(active: true).pluck(:id)

# DON'T: pluck then try to chain AR methods
User.pluck(:id).where(active: true)  # NoMethodError! pluck returns Array
```

### pick — Single Value

```ruby
User.where(id: 1).pick(:email)            # "alice@example.com"
User.where(id: 1).pick(:first_name, :last_name)  # ["Alice", "Smith"]
# Equivalent to: .limit(1).pluck(:email).first
```

### ids — Shorthand for pluck(:id)

```ruby
User.where(active: true).ids  # [1, 4, 7, 12]
```

---

## Grouping and Aggregations

### group

```ruby
# Count per status
Order.group(:status).count
# => {"pending" => 5, "shipped" => 12, "delivered" => 8}

# Sum per category
Product.group(:category).sum(:price)

# Group by date
Order.group("DATE(created_at)").count

# Multiple groups
Order.group(:status, :payment_method).count
# => {["pending", "card"] => 3, ["shipped", "card"] => 7, ...}
```

### having

```ruby
# Filter groups
Author.joins(:books).group("authors.id")
  .having("COUNT(books.id) > ?", 5)
  .select("authors.*, COUNT(books.id) AS book_count")

# With aggregation
Order.group(:customer_id)
  .having("SUM(total) > ?", 1000)
  .sum(:total)
```

### regroup — Override Previous Group

```ruby
Book.group(:author).regroup(:id)
# GROUP BY id (not author)
```

---

## Joins — Complete Guide

### INNER JOIN (joins)

```ruby
# Single association
Book.joins(:author)

# Multiple associations
Book.joins(:author, :reviews)

# Nested associations
Book.joins(reviews: :customer)

# Deep nesting
Author.joins(books: [{ reviews: { customer: :orders } }, :supplier])

# String SQL (for custom joins)
Author.joins("INNER JOIN books ON books.author_id = authors.id AND books.out_of_print = FALSE")
```

### LEFT OUTER JOIN

```ruby
Customer.left_outer_joins(:orders)

# With aggregation
Customer.left_outer_joins(:orders)
  .select("customers.*, COUNT(orders.id) AS order_count")
  .group("customers.id")
```

### where.associated / where.missing

```ruby
# Records WITH the association (cleaner than joins + where not null)
Customer.where.associated(:reviews)

# Records WITHOUT the association (cleaner than left join + where null)
Customer.where.missing(:reviews)
```

### Conditions on Joined Tables

```ruby
# Hash conditions (preferred)
Book.joins(:author).where(authors: { name: "Frank Herbert" })

# Using merge with scopes (very clean)
class Review < ApplicationRecord
  scope :recent, -> { where(created_at: 1.week.ago..) }
end

Book.joins(:reviews).merge(Review.recent).distinct
```

### joins vs includes — Reminder

```ruby
# joins: INNER JOIN for filtering. Does NOT preload association.
posts = Post.joins(:comments).where(comments: { approved: true }).distinct
posts.first.comments  # Triggers ANOTHER query!

# includes: Eager load for access. Separate queries by default.
posts = Post.includes(:comments)
posts.first.comments  # No additional query

# If you need BOTH filtering AND eager loading:
posts = Post.eager_load(:comments).where(comments: { approved: true })
# Single LEFT OUTER JOIN, and comments are preloaded
```

---

## Eager Loading — Deep Dive

### includes (Auto Strategy)

```ruby
# Single association
Post.includes(:author)

# Multiple
Post.includes(:author, :comments)

# Nested
Post.includes(comments: :author)

# Deep nesting
Post.includes(comments: { author: :profile })

# Mixed
Post.includes(:tags, comments: :author)
```

Rails decides the strategy:
- **Default:** Separate queries (`SELECT * FROM posts`, then `SELECT * FROM authors WHERE id IN (...)`)
- **With where on association:** Switches to LEFT OUTER JOIN automatically

```ruby
# This forces a JOIN strategy because of the where on comments
Post.includes(:comments).where(comments: { approved: true })

# For string conditions, you must add references
Post.includes(:comments).where("comments.approved = ?", true).references(:comments)
```

### preload (Force Separate Queries)

```ruby
Post.preload(:comments)
# Always: SELECT * FROM posts; SELECT * FROM comments WHERE post_id IN (...)
```

**When to use over includes:**
- When `includes` incorrectly switches to JOIN strategy
- When you want predictable query plans
- When the association table is very large (JOINs can be slower than IN queries)

**Cannot** filter on the preloaded association (no `where(comments: {...})` — use `eager_load` for that).

### eager_load (Force Single JOIN)

```ruby
Post.eager_load(:comments)
# Always: SELECT posts.*, comments.* FROM posts LEFT OUTER JOIN comments ON ...
```

**When to use:**
- When you need to filter/order by the association AND preload it
- When a single query is faster than two (rare — test with EXPLAIN)

### strict_loading — Catch Lazy Loading

```ruby
# On a relation
users = User.strict_loading.all
users.first.posts  # Raises ActiveRecord::StrictLoadingViolationError!

# On a record
user = User.first
user.strict_loading!
user.posts  # Raises!

# N+1 only mode — allows belongs_to, catches has_many lazy loads
user.strict_loading!(mode: :n_plus_one_only)
user.profile      # OK (belongs_to, single record)
user.posts         # Raises! (has_many would cause N+1)
user.posts.first.likes  # Raises!

# On an association declaration
class Author < ApplicationRecord
  has_many :books, strict_loading: true
end

# App-wide config
config.active_record.strict_loading_by_default = true
config.active_record.action_on_strict_loading_violation = :log  # or :raise (default)
```

**Recommendation:** Enable `strict_loading` in development/test to catch N+1s early. Use `:log` mode in production if you're not confident in full coverage.

---

## Scopes — Patterns and Pitfalls

### Basic Patterns

```ruby
class Article < ApplicationRecord
  # Simple conditions
  scope :published, -> { where(published: true) }
  scope :draft, -> { where(published: false) }
  scope :recent, -> { order(created_at: :desc) }

  # With arguments
  scope :by_author, ->(author) { where(author: author) }
  scope :created_after, ->(date) { where(created_at: date..) }
  scope :tagged, ->(tag) { joins(:tags).where(tags: { name: tag }) }

  # Composing scopes
  scope :featured, -> { published.where(featured: true).recent }

  # With safe conditional
  scope :search, ->(query) {
    query.present? ? where("title ILIKE ?", "%#{sanitize_sql_like(query)}%") : all
  }
end

# Chain freely
Article.published.recent.by_author(user).limit(10)
Article.published.tagged("ruby").search(params[:q])
```

### Scope vs Class Method

```ruby
class Article < ApplicationRecord
  # Scope — always returns Relation (nil becomes .all)
  scope :status, ->(s) { where(status: s) if s.present? }

  # Class method — can return nil (breaks chaining!)
  def self.status(s)
    where(status: s) if s.present?  # Returns nil when s is blank!
  end

  # Fix: always return a Relation from class methods
  def self.status(s)
    return all if s.blank?
    where(status: s)
  end
end
```

**Rule:** Use scopes for simple query fragments. Use class methods for complex logic — but always return `all` as fallback, never `nil`.

### Scope on Associations

```ruby
author = Author.first
author.books.published.recent  # Scopes work on associations!
author.books.costs_more_than(50)
```

### default_scope — Avoid It

```ruby
# DON'T — this infects every query on the model
class Article < ApplicationRecord
  default_scope { where(published: true) }  # Seems helpful...
end

# Problems:
Article.new.published  # true — affects new records!
Article.count          # Only counts published
Article.joins(:comments)  # Only joins published
Article.unscoped       # You'll need this everywhere

# DO — use explicit scopes instead
class Article < ApplicationRecord
  scope :published, -> { where(published: true) }
  scope :visible, -> { published.where(hidden: false) }
end
```

### Merging Scopes

```ruby
# AND — chaining scopes ANDs them
Article.published.where(featured: true)
# WHERE published = true AND featured = true

# merge — last wins for same column
Article.published.merge(Article.draft)
# WHERE published = false (merge overrides)

# merge with association scopes (very powerful)
Customer.joins(:orders).merge(Order.created_before(1.week.ago))
```

### Removing Scopes

```ruby
# Remove all scopes (including default_scope)
Article.unscoped

# Remove specific scope type
Article.where(published: true).limit(20).unscope(:limit)

# Remove specific where condition
Article.where(id: 10, published: true).unscope(where: :id)
# WHERE published = true
```

---

## Batching — Full API

### find_each

```ruby
User.find_each { |user| process(user) }

# Options
User.find_each(batch_size: 500)       # Default: 1000
User.find_each(start: 2000)           # Start from ID 2000
User.find_each(finish: 10000)         # Stop at ID 10000
User.find_each(order: :desc)          # Reverse order (default :asc)
User.find_each(start: 2000, finish: 10000, batch_size: 500)

# With scope
User.where(active: true).find_each { |u| process(u) }

# Returns Enumerator when no block given
User.find_each.map { |u| u.email.downcase }
```

### find_in_batches

```ruby
User.find_in_batches(batch_size: 1000) do |batch|
  # batch is an Array of up to 1000 User objects
  SomeAPI.bulk_create(batch.map(&:email))
end
```

Same options as `find_each`: `batch_size`, `start`, `finish`, `order`.

### in_batches

```ruby
# Yields ActiveRecord::Relation — for bulk SQL operations
User.where(active: false).in_batches(of: 1000) do |batch_relation|
  batch_relation.update_all(archived: true)   # Single UPDATE per batch
  batch_relation.delete_all                    # Single DELETE per batch
end

# load option — also load records
User.in_batches(of: 500, load: true) do |batch_relation|
  batch_relation.records.each { |u| process(u) }
end
```

**Key constraint:** All batch methods order by primary key internally. **Custom `order()` is ignored** (or raises, depending on config).

---

## Enums — Query Patterns

### Declaration

```ruby
class Order < ApplicationRecord
  # Hash syntax (recommended — explicit integer mapping)
  enum :status, { pending: 0, processing: 1, shipped: 2, delivered: 3, cancelled: 4 }

  # Array syntax (integers assigned by position — fragile!)
  enum :status, [:pending, :processing, :shipped, :delivered, :cancelled]

  # With prefix/suffix to avoid method name collisions
  enum :status, { active: 0, archived: 1 }, prefix: true
  # Methods: status_active?, status_archived?, status_active!, etc.

  enum :role, { admin: 0, user: 1 }, suffix: :type
  # Methods: admin_type?, user_type?
end
```

### Querying

```ruby
Order.pending                    # Scope: WHERE status = 0
Order.not_pending                # Scope: WHERE status != 0
Order.where(status: :shipped)    # Explicit
Order.where(status: [:shipped, :delivered])  # Multiple values

order.shipped?                   # Check
order.shipped!                   # Set + save!
```

### Gotcha: Don't Reorder Enum Arrays

```ruby
# If you had:
enum :status, [:pending, :shipped, :delivered]

# And later insert :processing:
enum :status, [:pending, :processing, :shipped, :delivered]
# shipped is now 2 (was 1), delivered is now 3 (was 2)
# ALL EXISTING DATA IS WRONG

# Always use hash syntax to avoid this:
enum :status, { pending: 0, shipped: 1, delivered: 2 }
# Safe to add: processing: 3
```

---

## Raw SQL — Safe Patterns

### sanitize_sql_like

```ruby
# User input in LIKE queries — escape wildcards
query = params[:search]  # Could contain % or _
safe = ActiveRecord::Base.sanitize_sql_like(query)
User.where("name LIKE ?", "#{safe}%")
```

### sanitize_sql_array / sanitize_sql_for_conditions

```ruby
# Build a parameterized SQL string
sql = ActiveRecord::Base.sanitize_sql_array(
  ["name = ? AND age > ?", "Alice", 18]
)
# => "name = 'Alice' AND age > 18"
```

### find_by_sql

```ruby
users = User.find_by_sql([
  "SELECT users.*, COUNT(posts.id) AS post_count
   FROM users
   LEFT JOIN posts ON posts.user_id = users.id
   WHERE users.active = ?
   GROUP BY users.id
   HAVING COUNT(posts.id) > ?
   ORDER BY post_count DESC
   LIMIT ?",
  true, 5, 20
])

users.first.post_count  # Accessible as virtual attribute
```

### select_all (Raw Hashes)

```ruby
result = ActiveRecord::Base.lease_connection.select_all(
  ActiveRecord::Base.sanitize_sql_array([
    "SELECT DATE(created_at) as day, COUNT(*) as total
     FROM orders
     WHERE created_at > ?
     GROUP BY day
     ORDER BY day",
    30.days.ago
  ])
)

result.to_a
# => [{"day" => "2025-01-01", "total" => 42}, ...]
```

### execute (Last Resort)

```ruby
# For DDL or statements that don't return records
ActiveRecord::Base.lease_connection.execute("VACUUM ANALYZE users")
```

---

## Existence and Calculations

### Existence

| Method | SQL | When to use |
|--------|-----|-------------|
| `exists?` | `SELECT 1 ... LIMIT 1` | Always prefer for existence checks |
| `exists?(id)` | `SELECT 1 WHERE id = ? LIMIT 1` | Check specific record by PK |
| `any?` | `SELECT 1 ... LIMIT 1` | Same as exists? on relations |
| `none?` | `SELECT 1 ... LIMIT 1` | Inverse of any? |
| `many?` | `SELECT COUNT(*) FROM (... LIMIT 2)` | More than one? |
| `present?` | Loads relation! | **Avoid** — use exists? |
| `empty?` | `SELECT 1 ... LIMIT 1` | OK — similar to !any? |

```ruby
# All valid
User.exists?(42)
User.where(active: true).exists?
User.any?
User.where(role: :admin).many?
order.line_items.any?
```

### Calculations

```ruby
Order.count                           # COUNT(*)
Order.count(:discount)                # COUNT(discount) — excludes NULL
Order.where(status: :shipped).count

Order.sum(:total)
Order.average(:total)
Order.minimum(:total)
Order.maximum(:total)

# With group
Order.group(:status).count            # => {"shipped" => 12, "pending" => 5}
Order.group(:status).sum(:total)      # => {"shipped" => 5000, "pending" => 1200}
Order.group(:status).average(:total)

# Distinct count
Order.distinct.count(:customer_id)    # Number of unique customers who ordered
```

---

## Locking

### Optimistic Locking

```ruby
# Requires lock_version integer column
c1 = Customer.find(1)
c2 = Customer.find(1)

c1.name = "Sandra"
c1.save!

c2.name = "Michael"
c2.save!  # Raises ActiveRecord::StaleObjectError
```

### Pessimistic Locking

```ruby
# Row-level lock (FOR UPDATE)
Book.transaction do
  book = Book.lock.find(42)    # SELECT ... FOR UPDATE
  book.stock -= 1
  book.save!
end

# Lock existing instance
book = Book.first
book.with_lock do
  book.increment!(:views)
end

# Share lock (PostgreSQL/MySQL)
Book.transaction do
  book = Book.lock("FOR SHARE").find(42)
  # Other transactions can read but not modify
end
```

---

## Overriding and Resetting Conditions

```ruby
# unscope — remove specific clause types
Book.where(published: true).order(:title).unscope(:order)
Book.where(id: 10, published: true).unscope(where: :id)

# only — keep only specific clauses
Book.where(published: true).order(:title).limit(10).only(:where, :order)

# reselect — replace select
Book.select(:title).reselect(:isbn)

# reorder — replace order
Book.order(:title).reorder(:created_at)

# rewhere — replace where condition
Book.where(published: true).rewhere(published: false)

# regroup — replace group
Book.group(:author).regroup(:status)

# none — empty relation (useful as fallback)
Book.none  # Chainable, returns empty relation, fires no queries

# unscoped — remove ALL scopes (including default_scope)
Book.unscoped
Book.unscoped { Book.published }  # Block form: removes default, applies published
```

---

## Method Chaining Patterns

### Building Queries Incrementally

```ruby
def index
  @posts = Post.published

  if params[:author_id].present?
    @posts = @posts.where(author_id: params[:author_id])
  end

  if params[:tag].present?
    @posts = @posts.joins(:tags).where(tags: { name: params[:tag] })
  end

  if params[:search].present?
    @posts = @posts.where("title ILIKE ?", "%#{Post.sanitize_sql_like(params[:search])}%")
  end

  @posts = @posts.order(created_at: :desc).page(params[:page])
end
```

### Query Objects (For Complex Queries)

```ruby
class PostSearch
  def initialize(params)
    @params = params
    @scope = Post.published
  end

  def results
    filter_by_author
    filter_by_tag
    filter_by_search
    apply_ordering
    @scope
  end

  private

  def filter_by_author
    return unless @params[:author_id].present?
    @scope = @scope.where(author_id: @params[:author_id])
  end

  def filter_by_tag
    return unless @params[:tag].present?
    @scope = @scope.joins(:tags).where(tags: { name: @params[:tag] }).distinct
  end

  def filter_by_search
    return unless @params[:search].present?
    @scope = @scope.where(
      "title ILIKE ?",
      "%#{Post.sanitize_sql_like(@params[:search])}%"
    )
  end

  def apply_ordering
    @scope = @scope.order(created_at: :desc)
  end
end

# Usage
posts = PostSearch.new(params).results.page(params[:page])
```

---

## Performance Patterns

### Avoiding SELECT *

```ruby
# When you only need specific columns (e.g., for a dropdown)
User.select(:id, :name).order(:name)

# Even better if you don't need AR objects
User.order(:name).pluck(:id, :name)
# => [[1, "Alice"], [2, "Bob"]]
```

### Subqueries

```ruby
# Use subqueries to avoid loading intermediate results
active_user_ids = User.where(active: true).select(:id)
Post.where(user_id: active_user_ids)
# SELECT posts.* FROM posts WHERE user_id IN (SELECT id FROM users WHERE active = true)
# Note: Rails automatically makes this a subquery — no .pluck needed!
```

### update_all / delete_all (Bulk Operations)

```ruby
# Skip callbacks and validations — direct SQL
User.where(active: false).update_all(archived: true)
User.where("last_login < ?", 2.years.ago).delete_all

# With increment/decrement
Post.where(id: post_ids).update_all("view_count = view_count + 1")
```

### Caching with size

```ruby
users = User.where(active: true)

# .count always runs SELECT COUNT(*)
users.count  # Query
users.count  # Query again

# .size is smart
users.size   # SELECT COUNT(*) (not loaded yet)
users.to_a   # Loads records
users.size   # No query (uses loaded array length)

# .length always loads records
users.length # Loads ALL records, counts in Ruby
```

### Explain Your Queries

```ruby
# Basic explain
User.where(email: "x@y.com").explain

# With analysis (PostgreSQL)
User.where(email: "x@y.com").explain(:analyze, :verbose)

# MySQL/MariaDB
User.where(email: "x@y.com").explain(:analyze)
```

---

## Rails 8.1 Features

### normalizes (Attribute Normalization)

While not strictly a query feature, `normalizes` affects how data is stored and therefore queried:

```ruby
class User < ApplicationRecord
  normalizes :email, with: ->(email) { email.strip.downcase }
end

# Queries automatically apply normalization
User.where(email: "  Alice@Example.COM  ")
# Matches records with email "alice@example.com"
User.find_by(email: "  Alice@Example.COM  ")
# Correctly finds the user
```

### Composite Primary Keys

```ruby
class TravelRoute < ApplicationRecord
  self.primary_key = [:origin, :destination]
end

route = TravelRoute.find(["Chicago", "NYC"])
routes = TravelRoute.find([["Chicago", "NYC"], ["NYC", "LA"]])
```

### where.associated / where.missing (Added Rails 7, Stable in 8.1)

```ruby
# Cleaner than LEFT JOIN + WHERE NULL checks
Customer.where.associated(:orders)    # Has at least one order
Customer.where.missing(:orders)       # Has no orders
```

### in_batches Improvements

```ruby
# use_ranges option for better performance with large datasets
User.in_batches(of: 1000, use_ranges: true) do |batch|
  batch.update_all(processed: true)
end
# Uses WHERE id >= X AND id < Y instead of WHERE id IN (...)
```

### assert_queries_count (Testing)

```ruby
# Assert exact query count
assert_queries_count(2) do
  User.includes(:posts).first.posts.to_a
end

# Assert no queries (e.g., cached data)
assert_no_queries do
  @cached_user.name
end
```

---

## Index of All Query Methods

**Finders:** `find`, `find_by`, `find_by!`, `first`, `first!`, `last`, `last!`, `take`, `take!`, `find_or_create_by`, `find_or_create_by!`, `find_or_initialize_by`, `find_by_sql`

**Conditions:** `where`, `where.not`, `where.associated`, `where.missing`, `or`, `and`, `rewhere`

**Ordering:** `order`, `reorder`, `reverse_order`

**Limiting:** `limit`, `offset`

**Selecting:** `select`, `reselect`, `distinct`

**Grouping:** `group`, `regroup`, `having`

**Joins:** `joins`, `left_outer_joins`, `includes`, `preload`, `eager_load`

**Eager Loading:** `includes`, `preload`, `eager_load`, `strict_loading`, `extract_associated`

**Scoping:** `scope`, `default_scope`, `unscoped`, `unscope`, `only`, `merge`

**Batching:** `find_each`, `find_in_batches`, `in_batches`

**Calculations:** `count`, `sum`, `average`, `minimum`, `maximum`, `pluck`, `pick`, `ids`

**Existence:** `exists?`, `any?`, `many?`, `none?`

**Mutation:** `update_all`, `delete_all`, `destroy_all`, `touch_all`

**Locking:** `lock`, `with_lock`

**Other:** `none`, `readonly`, `extending`, `annotate`, `optimizer_hints`, `create_with`, `from`, `explain`
