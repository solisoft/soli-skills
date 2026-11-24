# Models and the ORM (verified on soli 2.28.2, SQLite adapter; rechecked on 2.36.3)

## Database setup without SoliDB (SQLite)

The default app talks to SoliDB (`SOLIDB_HOST` in `.env`). For local work and
tests, SQLite needs nothing installed:

```bash
# .env
SOLI_DB_ADAPTER=sqlite
DATABASE_URL=sqlite://db/app.sqlite3

# .env.test   (soli test refuses to run without this file)
SOLI_DB_ADAPTER=sqlite
DATABASE_URL=sqlite://db/test.sqlite3
```

Then `soli db:migrate up`. The model surface is the same on SoliDB, SQLite,
Postgres and MySQL (documents stored as `_key` + JSON). The test runner
creates, migrates and drops its database itself (it still prints a SoliDB
host in its banner; ignore that).

## Defining a model

```soli
# app/models/post.sl  (any filename works; auto-loaded, no import anywhere)
class Post < Model
  has_many("comments")

  validates("title", {"presence": true, "min_length": 3})
  validates("body", {"presence": true})
  validates("status", {"inclusion": ["draft", "published"], "allow_nil": true})

  before_save("normalize_fields")

  scope("published", fn() { this.where({"status": "published"}) })

  def normalize_fields
    @title = @title.trim unless @title.blank?
    @status ||= "draft"
    true                       # see callbacks: never end on a false value
  end

  def shout
    @title.upcase
  end
end
```

Fields are not declared; whatever you assign is persisted. Collection name is
the plural snake_case (`Post` → `posts`).

## CRUD

```soli
post_record = Post.create({"title": "Hello", "body": "World"})  # always returns an instance
                                 # literal field defaults (`status: String = "draft"`) are stored and
                                 # set on it — also by create_many and find_or_create_by; a
                                 # value_field given as a hash comes back as its value object
post_record._key                 # string key (UUID); `id` is nil on a plain create
                                 # attributes never set the key: `_key` / `id` / `_from` / STI `type`
                                 # in the hash are dropped by create, create_many,
                                 # find_or_create_by, upsert, update(hash) and update_all —
                                 # pick a key with Post.create(attrs, {"key": "…"})
post_record._errors              # nil on success
Post.find(key)                   # RAISES "Post with id 'x' not found" → 404 in a request
Post.find_by("title", "Hello")   # nil on miss
Post.first_by("title", "Hello")  # nil on miss
Post.find_or_create_by("title", "Foc", {"body": "x"})
Post.new({"title": "Draft", "body": "b"}).save   # → true; Post.new() needs ()
post_record.title = "New"
post_record.save                 # true / false; on success _errors == []
post_record.update({"title": "x"})   # SKIPS callbacks (before_save didn't run)
post_record.delete
Post.count                       # Int
Post.all                         # array of instances
```

- **Failed create**: `_errors` is an **array of hashes**:
  `[{field => title, message => can't be blank}]`, and `_key` is nil.
  Read `post_record._errors[0]["field"]`.
- `_errors` is nil after a good `create`, `[]` after a good `save`/`update`.
  Both are falsy, so `if record._errors` works for either (checked on
  2.36.3; the app CLAUDE.md's "`[]` is truthy" is wrong) — `_errors.length > 0`
  is the explicit form.
- Never add a nil-check after `Model.find` (lint: `idiom/manual-find-guard`).
  Only `.find` on a class name raises; `Array#find`, `XmlNode#find` and a
  query chain's `.find { … }` return nil, and the lint leaves those alone.
- `Post.delete_all` empties the collection, with or without parentheses
  (before 2.41 the bare form evaluated to the function and deleted nothing).

## Validations

Options verified: `presence`, `min_length`, `format` (regex string),
`numericality` (form strings like `"12"` fail it), `min`, `inclusion`
("is not included in the list"), `allow_nil`, `custom` (method name).

```soli
class Account < Model
  validates("email", {"presence": true, "format": "^[^@]+@[^@]+$"})
  validates("name", {"custom": "validate_name"})
  validates("age", {"numericality": true, "min": 0, "allow_nil": true})

  def validate_name
    return "is too short" if @name.to_s.length < 2   # a String = the message

    nil                                              # nil / true = pass
  end
end
```

On an update (`record.update(attrs)`, `save` on a loaded record,
`Model.update(id, attrs)`) a custom validator sees the merged record with its
identity: `@_key` / `record["_key"]` is the key being updated and `@id` the
row's `id` (or that key). "Unique except me" is therefore
`taken.nil? || taken._key == @_key || "is taken"` (verified on SQLite, all
three paths; before the #674 fix the instance paths had no `@_key`).
`uniqueness: true` excludes the record itself on update.

A misspelled option key (`"presense"`) raises when the class loads — the app
(and the test server) refuses to start. Others documented but not verified
here: `uniqueness` (global, best-effort), `max_length`, `max`, `type`,
`on: "create"`, `if`/`unless` closures, `custom` closures.

## Callbacks

`before_create`, `after_create`, `before_update`, `after_update`,
`before_save`, `after_save`, `before_delete`, `after_delete` — each takes a
**method name string**.

**A `before_*` callback whose value is `false` aborts persistence** (verified:
`_key` stays nil, `_errors` = `before_create / before_save callback returned
false; persistence aborted`). A method ending in `@flagged ||= false` returns
`false`. End callbacks with `true`.

`update(attrs)` and `save(attrs)` skip callbacks: assign fields, then `save`.

## Querying

```soli
Post.where({"status": "published"}).count             # hash form: values are bound
Post.where({"views": {"gte": 6}}).count               # operator hash
Post.where("doc.views >= @min", {"min": 5}).count     # string form + binds; never concatenate input
Post.order("title", "desc").first                     # nil when none
Post.order("title").limit(2).offset(1).all
Post.order("status").order("title", "desc").all       # chained .order ADDS: status, then title
Post.order({"status": "asc", "title": "desc"}).all    # the same, one call
Post.published.reorder("views", "desc").all           # .reorder drops the sort a scope set
Post.order("title").first(2)                          # array of 2
Post.published.count                                  # scope; Post.published() also works
Post.where({...}).to_query                            # the generated SDBQL + binds
```

Terminators and what they actually return:

| Call | Returns |
|---|---|
| `.all` | array of instances |
| `.first` | instance or nil |
| `.count` | Int |
| `.exists` | **a QueryBuilder (always truthy!)** — use `.exists.first` → true/false |
| `.pluck("title")` | **a QueryBuilder** — use `.pluck("title").all` → `["Aaa", "Bbb"]` |
| `.sum("views")` | a QueryBuilder — `.sum("views").first` → 12 |
| `.aggregate({"total": ["sum", "views"], "n": ["count"]}).first` | `{n => 2, total => 12}` |
| `.where(...).update_all({...})` | bulk update, skips validations/callbacks |
| `.where(...).delete_all` | bulk delete of the matching rows |
| `Model.paginate({"page": 1, "per": 20})` | `{records => [...], pagination => {page, per, total, total_pages}}` |

A database error raises from every terminal — `.first`, `.exists.first`, the
aggregates, `.all`, `.count`, `update_all`, `delete_all`, `record.delete`:
none answers nil,
false or an `"Error: …"` string.

Prefer `.limit(n).all` over `.all.slice(0, n)`; `.all.map { |p| p.title }` or
`.all.map(&:title)` for a plain array.

## Associations

```soli
class Post < Model
  has_many("comments")
end

class Comment < Model
  belongs_to("post")      # FK post_id on comments
end

Comment.create({"text": "hi", "post_id": post_record._key})
post_record.comments.count                         # has_many → QueryBuilder (chainable)
comment.post.title                                 # belongs_to → instance
Post.where({...}).includes("comments").all         # eager-load: loaded[0].comments
Post.includes("comments", {"approved": true}).all   # hash filter = on the related record (rel.…),
                                                    # also {"where": {...}}; "fields" takes
                                                    # validated field names only (anything else raises)
Post.join("comments", {"approved": true}).all       # same: tests the comment, not the post.length → 2
```

Relation names in `s`: `has_many("order_remises")` reaches `OrderRemise` (the
class that exists, when `OrderRemis` does not), and `has_many("buses")` reads
`Bus`'s own collection, `bus` — a class name ending in `s` is its collection
name as is (`Address` → `address`). A `dependent: "delete"` cascade inside
`Model.transaction` needs SoliDB ≥ 2.10.2 to see children the transaction
itself created.

Also documented (unverified): `has_one`, `has_and_belongs_to_many`,
`{"class_name": "User", "foreign_key": "author_id"}`, `includes_count`.

## grouped — one round-trip for several reads

```soli
grouped(fn() {
  @posts = Post.all
  @total = Post.count
})
# after the block they are plain values
```

Batches `all`, `first`, `count`, `exists`, aggregates, `find`, `find_by`,
`first_by`. Writes run immediately. Reading a result inside the block forces
an early flush. Under `--dev` the reads are not coalesced (readable query log).

## Transactions

```soli
Post.transaction do
  Post.create({"title": "Tx1", "body": "x"})
end
# a throw inside the block rolls everything back (verified), then propagates
```

Reads inside the block (observed 2026-11-16 on a SoliDB build after 2.11.0;
the server does this since 2.10.2): `Post.count`, `.where(...).all`,
`find_by` and `find` see what the block wrote before them, and the second of
two records that break a `uniqueness` validation in one block is refused
(`_key` null). Same inside the test helper `with_transaction(fn() { ... })`.
On a SoliDB older than 2.10.2 those reads return committed data: the count
leaves the block's inserts out and both records commit.

## Read replicas (Postgres / MySQL, since 2.36.1)

```toml
# config/database.toml
[connections.primary]
adapter = "postgres"
url = "${DATABASE_URL}"
replicas = ["${REPLICA_URL:-}"]   # empty entry = no replica; refused on sqlite/d1/solidb
replica_delay = 2                 # seconds a client that wrote keeps reading the primary
```

- `GET`/`HEAD` requests run under the `reading` role: model reads go to a
  replica. Other verbs, jobs, sockets, scripts and migrations run under
  `writing` (primary only). Writes always go to the primary — never refused.
- A request's first write pins its later reads to the primary, and sets a
  `soli_db_primary` cookie keeping that client's GETs on the primary for
  `replica_delay` s. Reads inside a transaction always use the primary.
- `with_role("writing") do … end` forces the primary (a GET that must see
  fresh data); `with_role("reading") do … end` reads a replica even in a
  POST/job or after a write. `current_role()` → `"reading"` / `"writing"`.
  Both verified on the VM and the tree-walker; `with_role` without a block
  raises.
- Once a request has read a replica, `save`/`update` write only fields changed
  since load + the keys given to `update(…)`/`save(…)` (never the stale rest).
  In such a request, reassign a nested Hash/Array field after mutating it in
  place, or the change is not written.
- `--dev`: the dev bar's DB panel header shows `ROLE READING|WRITING` and
  tags each query `replica`/`primary`; `dev_queries()` entries gain
  `replica: true/false` (key absent when no connection has replicas).

## Encryption at rest (fields and files)

- `encrypts(:ssn)` — string fields, AES-256-GCM with `SOLI_ENCRYPTION_KEY`;
  cannot be filtered on by value.
- Files: `"encrypted": true` on `uploader` / `has_*_attached` (since 2.41.0; verified on a SoliDB `uploader` and on disk `store_attachment`). The
  store holds ciphertext (`SBE\x01…`); `attach_<field>`, `<field>_url`,
  `read_upload` read back the plain file. Needs `SOLI_ENCRYPTION_KEY` (store
  raises without it); old keys go in `SOLI_ENCRYPTION_OLD_KEYS`
  (comma-separated). On SoliDB an encrypted file streams with `Range`
  (since 2.41.1, verified: `206 bytes 65000-140000/208890`, `416`,
  `HEAD`); in 2.41.0 it was decrypted whole and a Range got the whole file.
  Disk/S3/R2 decrypt it whole. `direct_upload_start` refuses it — use
  resumable uploads.

## Serialization

`json_stringify(record)` gives the fields plus `_key`/`id`; `record.to_h` a
hash of fields. `record.to_json` returned nil in testing — don't use it on
models. `render_json` / `assigns()` serialize like `json_stringify`, so
`_errors` is not included. Instance/record JSON keys are sorted by name, and
fields named `password*`, `*_token`, `*_digest`, `*_secret`, `*_hash` are
omitted (`JSON.stringify` of an instance with `password_digest` and
`api_token` leaves both out). Build a Hash when you need a specific key order
or one of those fields.

## Domain modelling (DDD)

```soli
class Amount < ValueObject                # frozen, equal by value
  cents: Int
  invariant("cents must not be negative") { @cents >= 0 }   # InvariantViolation
end

class OrderPaid < DomainEvent
  number: String
end
subscribe(OrderPaid) do |event| … end     # app/subscribers/; runs after commit
subscribe(OrderPaid, ReceiptJob)          # job = transactional outbox

class Order < Model
  def pay
    @status = "paid"
    @record_event(OrderPaid.new(@number))  # published only if the save succeeds
  end
end

class Account < EventSourced              # emit / apply / Account.load(id)
  static snapshot_every: Int = 100        # unreleased on main (after 2.40.0)
  balance: Int = 0
  def deposit(cents: Int)
    @emit(Deposited.new(cents))
  end
  def apply(event)
    @balance = @balance + event.cents
  end
end
```

- Aggregates: `part_of :invoice` on a child → writes only inside
  `Invoice.transaction do … end`. `optimistic_lock` → `StaleRecord` (409).
- Bounded contexts: `module Billing … end` (tables `billing_*`, reopenable
  across files); `private class Ledger` inside it is unreachable from outside.
- Snapshots (after 2.40.0): `snapshot_every` writes one every N events in the
  events' transaction; `load` replays only what follows. The default keeps
  plain fields only (a `DateTime` field fails the save, naming it) — override
  `snapshot_state -> Hash` / `restore_snapshot(state)`; bump `static
  snapshot_version` when the state changes shape; `take_snapshot` backfills.
- Context map (after 2.40.0): `[contexts]` in `soli.toml`, e.g.
  `Shipping = ["Billing:events"]`; `soli lint` reports `Billing::Invoice` (or a
  `"Billing::Invoice"` class_name string) used from Shipping as
  `smell/context-boundary`. No table, no check.

## Raw SDBQL (SoliDB)

```soli
min_age = 18
users = @sdbql{
  FOR u IN users
  FILTER u.age >= #{min_age}
  RETURN u
}
```

`#{}` inside `@sdbql{}` is bound as a parameter, not interpolated. Returns raw
documents. (Not verified here — needs SoliDB; see the `solidb` skill.)

## Migrations

Generate, never hand-name: `soli db:migrate generate create_comments`
(`soli generate migration` does not exist). Migrations are top-level
functions:

```soli
# db/migrations/20261023110038_create_comments.sl
def up(db)
  db.create_collection("comments")
  db.create_index("comments", "idx_post_id", ["post_id"], {})
end

def down(db)
  db.drop_index("comments", "idx_post_id")
  db.drop_collection("comments")
end
```

**Counter fields** (unreleased — not in 2.36.3; observed on a build of `main`
against a SoliDB build after 2.11.0, 2026-11-16). SoliDB numbers a field on
insert, never twice the same and with no gap:

```soli
def up(db)
  db.create_collection("invoices")
  db.create_counter("invoices", "number", {
    "start": 41, "reset": "month", "timezone": "Europe/Paris",
    "prefix": "INV-{YYYY}-{MM}-", "padding": 4
  })                                   # options optional: db.create_counter("tickets", "seq") → 1, 2, 3
  db.set_counter_next("invoices", "number", 1288)   # an existing numbering
end

def down(db)
  db.drop_counter("invoices", "number")
  db.drop_collection("invoices")
end
```

- The model declares nothing. `Invoice.create({"total": 10}).number` →
  `"INV-2026-11-0041"`; `Invoice.new({...})` then `.save` sets it too (both
  take from the insert's answer the fields the record does not hold).
- Never set the field: `Invoice.create({"number": "X"})` saves nothing
  (`_key` is null), and an update that changes it is refused.
- **In `Model.transaction do … end` the record has no number**: it is given
  at commit. Keep the `_key` and `Invoice.find(key)` after the block.
- `db.list_counters("invoices")` → `[{"field", …, "state": {"next": 1289}}]`.
- SoliDB only; on a SQL connection the helpers raise `create_counter is
  SoliDB-only.`. On the `Solidb` client the same four methods exist
  (`db.create_counter(coll, field, options?)` returns the record).

**Field rules** (Soli 2.38.0 or later, against SoliDB 2.14.0 or later;
observed on a build of `main` before the release, 2026-11-17). SoliDB holds every write
of a collection to them, the model or not:

```soli
def up(db)
  db.set_field_rules("products", [
    {"field": "name", "required": true, "type": "string"},
    {"field": "price", "type": "number", "min": 0}
  ])                                        # replaces the rules it had
end

def down(db)
  db.set_field_rules("products", [])        # removes them
end
```

- Keys of a rule: `field`, `required`, `type` (`string` `number` `integer`
  `bool` `object` `array`), `min`, `max`, `in`. `db.list_field_rules(coll)`
  reads them. Stored documents are not checked.
- A refused write raises on the raw client (`'name' is required`); keep
  `validates` on the model for `_errors`.
- SoliDB only; on a SQL connection `set_field_rules is SoliDB-only.`.

**Foreign keys** (Soli 2.38.0 or later, against SoliDB 2.10.0 or later;
observed on a build of `main` before the release, 2026-11-17). SoliDB refuses a child
whose parent is not there:

```soli
def up(db)
  db.add_foreign_key("orders", "customer_id", "customers")              # restrict
  db.add_foreign_key("notes", "customer_id", "customers", {"on_delete": "cascade", "name": "fk_notes_customer"})
end

def down(db)
  db.remove_foreign_key("notes", "fk_notes_customer")
end
```

- `Order.create({"customer_id": "nobody"})` saves nothing (`_key` nil) and
  `order._errors == [{"field": "customer_id", "message": "must reference an
  existing record"}]`; `order.save` → `false`, same error. A nil field passes.
- `customer.delete` → `false` when a restricting key protects it, with
  `_errors == [{"field": "_base", "message": "is still referenced by other
  records"}]`. With `"cascade"` the delete goes through and SoliDB deletes
  the children itself: **their model callbacks do not run**.
- `on_delete`: `"restrict"` (default), `"cascade"`, `"set_null"`. Without a
  `name`, read it from `db.list_foreign_keys("orders")["foreign_keys"]`.
- SoliDB only; on a SQL connection `add_foreign_key is SoliDB-only.` (use
  `"references"` in `create_table`). On the `Solidb` client:
  `create_foreign_key`, `drop_foreign_key`, `list_foreign_keys`.
- Before this build a violation came back as "has already been taken".

Verified on SQLite (since 2.36.3): `create_collection`, `create_index`,
`drop_index`, `drop_collection`, `list_indexes` (`[{"name": …}]`). On a SQL
document table `create_index(t, name, ["email"], {"unique": true})` indexes the
field `doc.email` (`CREATE UNIQUE INDEX … ((doc ->> '$.email'))`); up to 2.36.2
it indexed a constant on SQLite — a unique index then refused the second row —
and failed on Postgres/MySQL, and `list_indexes` did not exist on SQL. The
generator's stub also lists SQL-only helpers (`db.create_table("users",
{...columns})`, `db.add_index`, `db.add_column`, `db.rename_column`,
`db.drop_column`, `db.execute`) and SoliDB `db.query("SDBQL")` — unverified
here. Two migration files with the same version stop `db:migrate` with an
error naming both (since 2.36.3; before, one was skipped silently); the
generators pick a free version.

Concerns: `app/models/concerns/x.sl` (`module X … end`, `included do … end`,
`class_methods do … end`) + `include X` in a model. Up to 2.36.2 a concern in
that folder loaded after the models and the app failed to boot ("Undefined
variable 'X'"); after, module files load first.

Commands: `soli db:migrate up | down | status | redo | reset`,
`soli db:seed generate <name>`, `soli db:seed`.
