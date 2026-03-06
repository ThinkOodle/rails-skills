# Active Record Associations — Reference

Detailed patterns, edge cases, and advanced examples for Rails 8.1 associations.

## Table of Contents

- [Association Types Deep Dive](#association-types-deep-dive)
- [Polymorphic Associations](#polymorphic-associations)
- [Self-Referential Associations](#self-referential-associations)
- [Single Table Inheritance (STI)](#single-table-inheritance-sti)
- [Delegated Types](#delegated-types)
- [Counter Caches](#counter-caches)
- [Scoped Associations](#scoped-associations)
- [Association Callbacks](#association-callbacks)
- [Association Extensions](#association-extensions)
- [Eager Loading Strategies](#eager-loading-strategies)
- [Bi-directional Associations & inverse_of](#bi-directional-associations--inverse_of)
- [Migration Patterns](#migration-patterns)
- [Testing Associations](#testing-associations)
- [Performance Patterns](#performance-patterns)
- [Troubleshooting](#troubleshooting)

---

## Association Types Deep Dive

### `belongs_to`

The child side of any relationship. The table with the FK column declares `belongs_to`.

```ruby
class Book < ApplicationRecord
  belongs_to :author
end
```

**Key behaviors:**
- Validates presence by default (Rails 5+). Use `optional: true` to allow NULL.
- Does NOT auto-save the parent. Saving the child saves the FK, not the parent object.
- `build_author` / `create_author` — note the `build_` prefix (not `author.build`).

**All methods added:**
```ruby
book.author                        # Get associated author (cached)
book.author = author               # Set association (updates FK)
book.build_author(attrs)           # Build unsaved author
book.create_author(attrs)          # Build + save author
book.create_author!(attrs)         # Build + save, raise on invalid
book.reload_author                 # Force reload from DB
book.reset_author                  # Clear cache, lazy reload next access
book.author_changed?               # FK will change on next save?
book.author_previously_changed?    # FK changed on last save?
```

**Options reference:**

| Option | Description |
|---|---|
| `class_name: "Patron"` | Non-standard class name |
| `foreign_key: "patron_id"` | Non-standard FK column |
| `primary_key: "guid"` | Parent's PK isn't `id` |
| `polymorphic: true` | Polymorphic — stores type + id |
| `optional: true` | Allow NULL FK (skip presence validation) |
| `touch: true` | Update parent's `updated_at` on save/destroy |
| `touch: :books_updated_at` | Touch a specific timestamp column |
| `counter_cache: true` | Maintain count on parent (column: `#{table}_count`) |
| `counter_cache: :num_books` | Custom counter cache column name |
| `inverse_of: :books` | Explicit inverse association |
| `default: -> { Current.user }` | Default value for new records |
| `strict_loading: true` | Raise on lazy load |

### `has_one`

The parent side of a one-to-one relationship. FK lives on the **other** table.

```ruby
class User < ApplicationRecord
  has_one :profile, dependent: :destroy
end

class Profile < ApplicationRecord
  belongs_to :user
end
```

**Key behaviors:**
- Assigning a new object auto-saves it AND nullifies/destroys the old one.
- Always add a **unique index** on the FK column to enforce one-to-one at the DB level.
- Uses `build_profile` / `create_profile` (not `profile.build`).

**When to use `has_one` vs `belongs_to`:**
- Ask: "Who owns whom?" The owner gets `has_one`. The owned gets `belongs_to`.
- The FK column is always on the `belongs_to` table.

### `has_many`

The parent side of a one-to-many relationship.

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end
```

**Key collection methods:**
```ruby
author.books                       # CollectionProxy (lazy)
author.books.load                  # Force load, cache result
author.books.reload                # Force reload from DB
author.books.size                  # Uses counter cache if available, else COUNT
author.books.count                 # Always COUNT query
author.books.length                # Loads all, counts in memory
author.books.empty?                # Uses counter cache or EXISTS
author.books.any?                  # Uses counter cache or EXISTS
author.books.exists?(id: 5)       # EXISTS query
author.books.find(id)             # Find within association
author.books.where(published: true) # Scoped query (lazy)
author.books.build(attrs)         # Build unsaved (note: .build, not build_)
author.books.create(attrs)        # Build + save
author.books.create!(attrs)       # Build + save, raise on invalid
author.books << book              # Add to collection, saves immediately
author.books.delete(book)         # Remove (nullify FK or delete)
author.books.destroy(book)        # Remove + destroy record
author.books.clear                # Remove all (strategy depends on dependent:)
author.book_ids                   # Array of IDs
author.book_ids = [1, 2, 3]      # Assign by IDs (adds/removes as needed)
```

**`size` vs `count` vs `length`:**
- `size` — smartest choice. Uses counter cache if available, loaded collection if already loaded, else COUNT query.
- `count` — always fires a COUNT query. Ignores cache.
- `length` — loads ALL records into memory, counts them. Wasteful for large collections.

### `has_many :through`

Many-to-many via a join model. **Always prefer this over HABTM.**

```ruby
class Doctor < ApplicationRecord
  has_many :appointments, dependent: :destroy
  has_many :patients, through: :appointments
end

class Appointment < ApplicationRecord
  belongs_to :doctor
  belongs_to :patient
  # Can have extra attributes: scheduled_at, notes, status, etc.
end

class Patient < ApplicationRecord
  has_many :appointments, dependent: :destroy
  has_many :doctors, through: :appointments
end
```

**Why always `:through`?**
1. You can add attributes to the join (timestamps, status, metadata)
2. You can add validations and callbacks to the join model
3. You can query the join model independently
4. You can use `dependent:` options
5. Every HABTM eventually needs to become a `:through` anyway

**Nested through (shortcut associations):**
```ruby
class Country < ApplicationRecord
  has_many :states, dependent: :destroy
  has_many :cities, through: :states  # Shortcut through nested has_many
end

class State < ApplicationRecord
  belongs_to :country
  has_many :cities, dependent: :destroy
end

class City < ApplicationRecord
  belongs_to :state
  has_one :country, through: :state  # has_one :through for the inverse
end
```

**`source:` option — when Rails can't infer the source:**
```ruby
class User < ApplicationRecord
  has_many :memberships, dependent: :destroy
  has_many :groups, through: :memberships, source: :team
  # source: :team tells Rails: Membership.belongs_to(:team) is the source
end

class Membership < ApplicationRecord
  belongs_to :user
  belongs_to :team  # Name doesn't match "groups" — need source:
end
```

### `has_and_belongs_to_many` (HABTM) — Don't Use

Exists for legacy compatibility. **Always use `has_many :through` instead.**

If you encounter HABTM in existing code:
```ruby
# Legacy HABTM
class Assembly < ApplicationRecord
  has_and_belongs_to_many :parts
end

# Convert to has_many :through
class Assembly < ApplicationRecord
  has_many :assembly_parts, dependent: :destroy
  has_many :parts, through: :assembly_parts
end

class AssemblyPart < ApplicationRecord
  belongs_to :assembly
  belongs_to :part
end
```

Migration to convert (add model to join table):
```ruby
class AddIdToAssembliesParts < ActiveRecord::Migration[8.1]
  def change
    # Add primary key and timestamps to existing join table
    add_column :assemblies_parts, :id, :primary_key
    add_timestamps :assemblies_parts
    # Rename to follow convention (plural of join model)
    rename_table :assemblies_parts, :assembly_parts
  end
end
```

---

## Polymorphic Associations

A model belongs to multiple other models via a single association.

```ruby
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end

class Article < ApplicationRecord
  has_many :comments, as: :commentable, dependent: :destroy
end
```

**Migration:**
```ruby
create_table :comments do |t|
  t.text :body
  t.references :commentable, polymorphic: true, null: false
  # Creates: commentable_id (bigint), commentable_type (string)
  # Creates: index on [commentable_type, commentable_id]
  t.timestamps
end
```

### When to Use Polymorphic

**Good use cases:**
- Comments on many different models
- Attachments/uploads shared across models
- Activity logs / audit trails
- Tags / taggings
- Notifications referencing various sources

**Bad use cases (don't use polymorphic):**
- Only 2-3 specific models — use separate FKs instead
- You need database-level foreign key constraints (can't add FK on polymorphic)
- You need efficient JOINs across the polymorphic boundary
- The "types" are actually related (consider STI or delegated types)

### `has_many :through` with Polymorphic Source

```ruby
class Tag < ApplicationRecord
  has_many :taggings, dependent: :destroy
  has_many :posts, through: :taggings, source: :taggable, source_type: "Post"
  has_many :articles, through: :taggings, source: :taggable, source_type: "Article"
end

class Tagging < ApplicationRecord
  belongs_to :tag
  belongs_to :taggable, polymorphic: true
end

class Post < ApplicationRecord
  has_many :taggings, as: :taggable, dependent: :destroy
  has_many :tags, through: :taggings
end
```

**Note:** The `through` association itself cannot be polymorphic, but the `source` can be.

---

## Self-Referential Associations

A model that references itself.

### Tree Structure (Parent-Child)

```ruby
class Category < ApplicationRecord
  belongs_to :parent, class_name: "Category", optional: true
  has_many :children, class_name: "Category", foreign_key: "parent_id",
           dependent: :destroy, inverse_of: :parent
end
```

Migration:
```ruby
create_table :categories do |t|
  t.string :name
  t.references :parent, foreign_key: { to_table: :categories }
  t.timestamps
end
```

**Note:** `foreign_key: { to_table: :categories }` — critical for self-joins. Without `to_table`, Rails looks for a `parents` table.

### Manager-Employee

```ruby
class Employee < ApplicationRecord
  belongs_to :manager, class_name: "Employee", optional: true
  has_many :subordinates, class_name: "Employee", foreign_key: "manager_id",
           dependent: :nullify, inverse_of: :manager
end
```

### Social Follow/Friend

```ruby
class User < ApplicationRecord
  # Users I follow
  has_many :active_follows, class_name: "Follow", foreign_key: "follower_id",
           dependent: :destroy
  has_many :following, through: :active_follows, source: :followed

  # Users following me
  has_many :passive_follows, class_name: "Follow", foreign_key: "followed_id",
           dependent: :destroy
  has_many :followers, through: :passive_follows, source: :follower
end

class Follow < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"

  validates :follower_id, uniqueness: { scope: :followed_id }
end
```

Migration:
```ruby
create_table :follows do |t|
  t.references :follower, null: false, foreign_key: { to_table: :users }
  t.references :followed, null: false, foreign_key: { to_table: :users }
  t.timestamps
end
add_index :follows, [:follower_id, :followed_id], unique: true
```

---

## Single Table Inheritance (STI)

Multiple models in one table, differentiated by a `type` column.

```ruby
class Vehicle < ApplicationRecord
  # type column stores "Car", "Truck", "Motorcycle"
end

class Car < Vehicle
  has_many :doors, dependent: :destroy  # STI models can have their own associations
end

class Truck < Vehicle
  has_many :cargo_holds, dependent: :destroy
end
```

**When to use STI:**
- Subclasses share most attributes
- You query across types frequently (`Vehicle.all`)
- Differences are mostly behavioral (methods), not structural (columns)

**When NOT to use STI:**
- Subclasses have very different attributes (table bloat with NULLs)
- You rarely query across types
- Consider delegated types instead

**STI + Associations:**
```ruby
# Associating with an STI model
class Fleet < ApplicationRecord
  has_many :vehicles, dependent: :destroy
  has_many :cars    # Automatically scoped: WHERE type = 'Car'
  has_many :trucks  # Automatically scoped: WHERE type = 'Truck'
end
```

**Gotcha — custom inheritance column:**
```ruby
class Vehicle < ApplicationRecord
  self.inheritance_column = "kind"  # Use "kind" instead of "type"
end
```

**Gotcha — disabling STI:**
```ruby
class Vehicle < ApplicationRecord
  self.inheritance_column = nil  # Disable STI entirely
end
```

---

## Delegated Types

Alternative to STI that avoids table bloat. Shared attributes in superclass table, type-specific attributes in separate tables.

```ruby
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[Message Comment], dependent: :destroy
end

module Entryable
  extend ActiveSupport::Concern
  included do
    has_one :entry, as: :entryable, touch: true
  end
end

class Message < ApplicationRecord
  include Entryable
  # Has its own table with message-specific columns
end

class Comment < ApplicationRecord
  include Entryable
  # Has its own table with comment-specific columns
end
```

**When to use delegated types vs STI:**
- STI: Models share 80%+ of attributes
- Delegated types: Models share some attributes but have many unique ones

---

## Counter Caches

Avoid `COUNT(*)` queries by caching the count in a column.

### Setup

```ruby
# 1. Declare on belongs_to side
class Book < ApplicationRecord
  belongs_to :author, counter_cache: true
end

class Author < ApplicationRecord
  has_many :books, dependent: :destroy
end

# 2. Add column to parent table
class AddBooksCountToAuthors < ActiveRecord::Migration[8.1]
  def change
    add_column :authors, :books_count, :integer, default: 0, null: false
  end
end

# 3. Backfill existing data
class BackfillBooksCount < ActiveRecord::Migration[8.1]
  def up
    Author.find_each do |author|
      Author.reset_counters(author.id, :books)
    end
  end
end
```

### Custom Column Name

```ruby
belongs_to :author, counter_cache: :num_books
# Column must be: authors.num_books
```

### Safe Backfill Pattern

For large tables, use `counter_cache: { active: false }` during backfill:

```ruby
# During backfill — always hits DB for count
belongs_to :author, counter_cache: { active: false }

# After backfill complete — uses cached value
belongs_to :author, counter_cache: true
```

### Resetting Stale Counters

```ruby
# Reset for one record
Author.reset_counters(author.id, :books)

# Reset all
Author.find_each { |a| Author.reset_counters(a.id, :books) }
```

---

## Scoped Associations

Add conditions to associations.

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy
  has_many :published_books, -> { where(published: true) },
           class_name: "Book", dependent: :destroy
  has_many :recent_books, -> { where("created_at > ?", 30.days.ago) },
           class_name: "Book", dependent: :destroy
  has_many :books_by_title, -> { order(:title) },
           class_name: "Book", dependent: :destroy
end
```

**With owner context (prevents preloading):**
```ruby
class Supplier < ApplicationRecord
  has_one :account, ->(supplier) { where(active: supplier.active?) }
  # WARNING: Can't preload this because scope depends on owner
end
```

**Hash conditions auto-apply on create:**
```ruby
has_many :published_books, -> { where(published: true) }, class_name: "Book"

author.published_books.create(title: "New Book")
# Automatically sets published: true
```

**Scoped `has_many :through` with `distinct`:**
```ruby
class Person < ApplicationRecord
  has_many :readings, dependent: :destroy
  has_many :articles, -> { distinct }, through: :readings
end
```

---

## Association Callbacks

Triggered when objects are added to or removed from a collection.

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy,
           before_add: :check_limit,
           after_add: :notify_subscribers,
           before_remove: :log_removal,
           after_remove: :update_stats

  private

  def check_limit(book)
    throw(:abort) if books.count >= 100
  end

  def notify_subscribers(book)
    AuthorSubscriptionMailer.new_book(self, book).deliver_later
  end

  def log_removal(book)
    Rails.logger.info "Removing #{book.title} from #{name}"
  end

  def update_stats(book)
    recalculate_stats!
  end
end
```

**Available callbacks:**
- `before_add` — before adding to collection. `throw(:abort)` to prevent.
- `after_add` — after adding to collection.
- `before_remove` — before removing from collection.
- `after_remove` — after removing from collection.

**Multiple callbacks:**
```ruby
has_many :books, before_add: [:check_limit, :validate_genre]
```

---

## Association Extensions

Add custom methods to association proxies.

### Inline Extension

```ruby
class Author < ApplicationRecord
  has_many :books, dependent: :destroy do
    def published
      where(published: true)
    end

    def by_year(year)
      where("EXTRACT(YEAR FROM published_at) = ?", year)
    end

    def find_by_isbn(isbn)
      find_by(isbn: isbn)
    end
  end
end

# Usage
author.books.published
author.books.by_year(2024)
author.books.find_by_isbn("978-0-123456-78-9")
```

### Shared Extension Module

```ruby
module FindRecentExtension
  def recent(days = 30)
    where("created_at > ?", days.days.ago)
  end
end

class Author < ApplicationRecord
  has_many :books, -> { extending FindRecentExtension }, dependent: :destroy
end

class Publisher < ApplicationRecord
  has_many :editions, -> { extending FindRecentExtension }, dependent: :destroy
end
```

### Accessing Proxy Internals

```ruby
module AdvancedExtension
  def search_and_log(query)
    results = where(title: query)
    owner = proxy_association.owner
    assoc_name = proxy_association.reflection.name
    Rails.logger.info "#{owner.class}##{assoc_name}: searched '#{query}', found #{results.size}"
    results
  end
end
```

---

## Eager Loading Strategies

### `includes` (Smart Default)

```ruby
# Separate queries (preload strategy)
Author.includes(:books)
# SELECT * FROM authors
# SELECT * FROM books WHERE author_id IN (1, 2, 3, ...)

# Switches to LEFT JOIN when filtering by association
Author.includes(:books).where(books: { published: true })
# SELECT authors.*, books.* FROM authors LEFT JOIN books ON ...
```

### `preload` (Always Separate Queries)

```ruby
Author.preload(:books)
# Always: SELECT * FROM authors
#         SELECT * FROM books WHERE author_id IN (...)

# Can't filter by association — raises error:
Author.preload(:books).where(books: { published: true })
# => ActiveRecord::StatementInvalid
```

### `eager_load` (Always LEFT JOIN)

```ruby
Author.eager_load(:books)
# Always: SELECT authors.*, books.*
#         FROM authors LEFT OUTER JOIN books ON ...

# Can filter by association:
Author.eager_load(:books).where(books: { published: true })
```

### Nested Eager Loading

```ruby
# Multiple associations
Author.includes(:books, :profile)

# Nested associations
Author.includes(books: :reviews)

# Deep nesting
Author.includes(books: [reviews: :reviewer, publisher: :address])

# Mixed
Author.includes(:profile, books: [:reviews, :tags])
```

### `strict_loading` (Prevent N+1)

```ruby
# On a query
authors = Author.strict_loading.all
authors.first.books  # => ActiveRecord::StrictLoadingViolationError

# On a model (all queries)
class Author < ApplicationRecord
  self.strict_loading_by_default = true
end

# On a specific association
class Author < ApplicationRecord
  has_many :books, strict_loading: true, dependent: :destroy
end

# On a record
author = Author.first
author.strict_loading!
author.books  # => raises
```

### Preload in Batches

```ruby
# For very large collections
Author.find_each do |author|
  # Still N+1 — find_each doesn't support includes
end

# Use in_batches instead
Author.includes(:books).in_batches(of: 100) do |batch|
  batch.each { |author| process(author) }
end
```

---

## Bi-directional Associations & inverse_of

### When Rails Auto-Detects

```ruby
# Auto-detected — no inverse_of needed
class Author < ApplicationRecord
  has_many :books
end
class Book < ApplicationRecord
  belongs_to :author
end

author = Author.first
book = author.books.first
book.author.equal?(author)  # => true (same object in memory)
```

### When You Must Specify `inverse_of`

```ruby
# Custom foreign_key — NOT auto-detected
class Author < ApplicationRecord
  has_many :books, foreign_key: "writer_id", inverse_of: :writer
end
class Book < ApplicationRecord
  belongs_to :writer, class_name: "Author", foreign_key: "writer_id", inverse_of: :books
end

# Custom class_name — NOT auto-detected
class Author < ApplicationRecord
  has_many :publications, class_name: "Book", inverse_of: :author
end

# Scoped association — NOT auto-detected (unless automatic_scope_inversing is true)
class Author < ApplicationRecord
  has_many :published_books, -> { where(published: true) }, class_name: "Book", inverse_of: :author
end
```

### What Breaks Without Correct inverse_of

1. **Extra queries:** `book.author` fires a new query even when author is loaded
2. **Inconsistent data:** Modifying `author.name` isn't reflected in `book.author.name`
3. **Failed autosave:** Building `author.books.new` then saving book doesn't save author
4. **Failed presence validation:** `book.valid?` says "Author must exist" even when built from `author.books.new`

### Enable Automatic Scope Inversing

```ruby
# config/application.rb
config.active_record.automatic_scope_inversing = true
# Now scoped associations auto-detect inverse_of
```

---

## Migration Patterns

### Standard Foreign Key

```ruby
# In create_table
t.references :author, null: false, foreign_key: true

# Adding to existing table
add_reference :books, :author, null: false, foreign_key: true

# These are equivalent:
t.references :author
t.belongs_to :author
```

### Polymorphic Foreign Key

```ruby
t.references :commentable, polymorphic: true, null: false
# Creates: commentable_id, commentable_type, composite index

# Manual equivalent:
t.bigint :commentable_id, null: false
t.string :commentable_type, null: false
add_index :comments, [:commentable_type, :commentable_id]
```

### Self-Referential Foreign Key

```ruby
t.references :parent, foreign_key: { to_table: :categories }
# to_table is REQUIRED for self-joins
```

### Custom Foreign Key Name

```ruby
t.bigint :creator_id, null: false
add_foreign_key :posts, :users, column: :creator_id
add_index :posts, :creator_id
```

### Join Table for `has_many :through`

```ruby
create_table :taggings do |t|
  t.references :tag, null: false, foreign_key: true
  t.references :article, null: false, foreign_key: true
  t.timestamps
end
add_index :taggings, [:tag_id, :article_id], unique: true
```

### Unique Index for `has_one`

```ruby
t.references :user, null: false, foreign_key: true, index: { unique: true }
```

---

## Testing Associations

### Model Test (Minitest)

```ruby
class AuthorTest < ActiveSupport::TestCase
  test "has many books" do
    author = authors(:prolific_author)
    assert_respond_to author, :books
    assert_kind_of ActiveRecord::Associations::CollectionProxy, author.books
  end

  test "destroying author destroys books" do
    author = authors(:prolific_author)
    book_count = author.books.count
    assert_operator book_count, :>, 0

    assert_difference "Book.count", -book_count do
      author.destroy
    end
  end

  test "book requires author" do
    book = Book.new(title: "Orphan")
    refute book.valid?
    assert_includes book.errors[:author], "must exist"
  end
end
```

### Testing Eager Loading

```ruby
test "index eager loads authors" do
  assert_queries_count(2) do  # 1 for posts, 1 for authors
    Post.includes(:author).to_a
  end
end
```

### Testing Counter Cache

```ruby
test "books_count reflects actual count" do
  author = authors(:prolific_author)
  assert_equal author.books.count, author.books_count

  assert_difference -> { author.reload.books_count }, 1 do
    author.books.create!(title: "New Book")
  end
end
```

---

## Performance Patterns

### Batch Preloading

```ruby
# Preload after the fact (useful in service objects)
posts = Post.limit(100).to_a
ActiveRecord::Associations::Preloader.new(
  records: posts,
  associations: [:author, :comments]
).call
```

### Avoiding `.count` on Large Collections

```ruby
# SLOW — COUNT query every time
author.books.count

# FAST — uses counter cache
author.books.size  # When counter_cache is configured

# FAST — just check existence
author.books.any?     # EXISTS query
author.books.empty?   # EXISTS query (negated)
author.books.exists?  # EXISTS query
```

### Pluck Instead of Load

```ruby
# SLOW — loads all Book objects into memory
author.books.map(&:id)

# FAST — only fetches IDs
author.book_ids

# FAST — select specific columns
author.books.pluck(:id, :title)
```

### Conditional Eager Loading

```ruby
# Only eager load when you'll use it
scope = Post.all
scope = scope.includes(:author) if params[:include_author]
scope = scope.includes(:comments) if params[:include_comments]
```

---

## Troubleshooting

### "uninitialized constant" Error

You used plural when you needed singular (or vice versa):
```ruby
belongs_to :authors    # WRONG — should be :author
has_many :book         # WRONG — should be :books
```

### "Cannot find inverse" Warning

Add explicit `inverse_of:`:
```ruby
has_many :books, foreign_key: "writer_id", inverse_of: :writer
```

### Foreign Key Violation on Delete

You're missing `dependent:` or using the wrong option:
```ruby
# If FK constraint is NOT NULL
has_many :books, dependent: :destroy  # or :delete_all

# If FK constraint allows NULL
has_many :books, dependent: :nullify
```

### Counter Cache Out of Sync

```ruby
Author.reset_counters(author_id, :books)
# Or reset all:
Author.find_each { |a| Author.reset_counters(a.id, :books) }
```

### Polymorphic Association Renamed Class

When you rename a class used in polymorphic associations:
```ruby
class RenameProductToItem < ActiveRecord::Migration[8.1]
  def up
    # Must update the type column!
    Picture.where(imageable_type: "Product").update_all(imageable_type: "Item")
  end
end
```

### N+1 in Serializers / Jbuilder

```ruby
# Controller must eager load what the view/serializer needs
def index
  @posts = Post.includes(author: :profile, comments: :user)
end
```

### `touch: true` Cascading Slowly

If `touch: true` cascades through many levels, consider:
```ruby
# Use touch: false and handle manually
belongs_to :author, touch: false

# Or batch touch in a callback
after_commit :touch_author
def touch_author
  author.touch
end
```

### Association Scope Not Applied on Create

Only hash-style `where` scopes auto-apply on create:
```ruby
# Auto-applies published: true on create
has_many :published_books, -> { where(published: true) }, class_name: "Book"

# Does NOT auto-apply (string condition)
has_many :recent_books, -> { where("created_at > ?", 1.week.ago) }, class_name: "Book"
```
