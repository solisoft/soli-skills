---
name: soli-lang
description: Use when writing, reading, reviewing or debugging Soli code — the Ruby-like language and MVC web framework (Rust-implemented, `soli` binary). Triggers on .sl / .html.slv files, soli.toml, config/routes.sl, `soli serve` / `soli test` / `soli new` / `soli generate` / `soli db:migrate`, Soli controllers (`< Controller`), models (`< Model`), ERB views, middleware, migrations and `describe`/`test` specs. Covers syntax, idiomatic style, framework APIs and the traps where Ruby or JS habits silently break.
---

# Soli

Soli looks like Ruby but is not Ruby. Code that "looks right" often parses and
then misbehaves. Everything here was checked against `soli` 2.28.2 and
rechecked on 2.36.3 (anything marked "unreleased" is on `main` only); when in
doubt, **run it**: `soli -e 'print(...)'` or `soli file.sl`.

## Verification loop (do this before saying "done")

```bash
soli fmt <files>            # canonical layout; rewrites `== nil` guards, // comments, blank-after-guard
soli lint <files>           # also lints .html.slv (double-escape), naming, smells, undefined locals
soli check <files>          # static type check (same checker `soli script.sl` runs)
soli test tests/<spec>.sl   # narrow first; coverage is ON by default (threshold 80%)
soli test --no-coverage     # fast loop
soli serve . --dev          # then curl the route: confirm status + markup
```

`fmt` before `lint`: fmt fixes layout so lint's remaining output needs judgement.

## Syntax essentials

```soli
name = "Alice"                 # bare assignment declares; `let` only for a type
let age: Int = 30              #   annotation or to avoid mutating an outer var
const MAX_SIZE = 100

def add(a: Int, b: Int) -> Int
  a + b                        # last expression is the return value
end

def greet                      # no-param def may omit ()
  "Hello!"
end

def configure(host: String = "localhost", port: Int = 8080)
  "#{host}:#{port}"
end
configure(port: 3000, host: "example.com")   # named args, any order

double = fn(x) { x * 2 }       # lambda in a variable: ALWAYS braces
add_one = |x| { x + 1 }
5 |> double() |> add_one()     # 11

[1, 2, 3].map { |x| x * 2 }                 # [2, 4, 6]
[1, 2, 3, 4].filter { |x| x > 2 }           # [3, 4]
[1, 2, 3].reduce(fn(acc, x) acc + x, 0)     # 6 — reduce(0) { ... } raises
(1..51).map { |i| i }.length                # 50 — ranges are EXCLUSIVE
xs.each do |x|
  print(x)
end

print("adult") if age >= 18
result = risky() rescue "default"           # postfix rescue
port = try                                  # `try` (2.35.0) and `if … end` (2.36.0)
  int(env_port)                             #   are values when assigned
catch e
  8080
end
label = match value {
  [first, ...rest] => "first: #{first}",
  {name: n, age: a} => "#{n} is #{a}",
  n if n > 0 => "positive",                 # guarded binding matches ANYTHING: keep it after
  _ => "other"                              # array/hash patterns or it compares an array to 0
}
```

Classes: `@field` reads/writes `this.field`; `@method` calls a method on self.

```soli
class Employee < Person
  salary: Float

  new(name: String, age: Int, salary: Float)
    super(name, age)
    @salary = salary
  end

  def summary -> String
    "#{@greet} and I earn #{@salary}"
  end

  def describe_more
    "more: " + super.describe_more()   # parent method: super.name(), not super()
  end
end
Employee.new("Bob", 40, 1000.5)
```

Errors: `throw` (there is **no `raise`**), `try / catch e / finally / end`,
typed `catch NotFound e`, `DomainError` subclasses become a 422 when uncaught
in a request.

## Code style (authoritative — matches the repo's CLAUDE.md)

- **`@field`, not `this.field`**, in instance methods and constructors. `this`
  only where there is no instance: `static { this.layout = ... }`, `static def`,
  scope bodies `scope("x", fn() { this.where(...) })`, or passing the object.
- **Ruby blocks for iteration**: `xs.map { |x| x * 2 }`, `xs.each do |x| … end`.
  Keep `fn` for stored lambdas (with braces), `reduce(fn(acc, x) …, 0)`,
  `grouped(fn() { … })`, `respond_to` hashes, pipelines.
- **Implicit returns**; `return` only for guard clauses — and before a last line
  that would start with `fn(` (read as a named function declaration).
- **No `()` on zero-arg method calls** (`Post.all`, `@post.save`,
  `name.trim.downcase`, `DateTime.utc.to_unix`) — **except**: bare functions
  (`current_user()`, `session_destroy()`, `posts_path()`) and any method that
  prints as `<method ...>` when called bare (e.g. `xs.tally()`). `Post.new`
  and `Post.new()` both build an instance. A static method whose parameters
  all have defaults (`static def featured(limit = 6)`) prints `<fn featured>`
  bare on the VM (production) through 2.38.1: write `Catalog.featured()` there
  (fixed after 2.38.1; instance methods were already called).
  `clock` is the one builtin that works bare.
- **A blank line after every guard clause** (`return … if …`, `next if …`)
  unless the next line is `end`/`else`/another guard. `soli fmt` inserts it.
- **`nil`, not `null`** (`nil == null` is true). `null` only inside SDBQL/JSON.
- **No `let` by default, no semicolons.**
- **Controllers**: every public action has a route comment right above it
  (`# GET /posts/:id`); helpers are `_name`, under `private`, called `@_name(...)`.
  View data goes through `@fields`, not a hash passed to `render`.
- **Intelligible names** — no `p`, `r`, `qb`. Short names only for `i`, `j`,
  obvious block params.
- Idioms: `.blank?`/`.present?`, `.nil?` (not `== nil`), `x.to_s` (not `x ?? ""`),
  `["a","b"].includes?(s)` (not `||` chains), `@n ||= 0`, `user&.name`.

## Top traps (full list: references/traps.md)

1. `10 / 3` is `3` (integer division). Use `10.to_f / 3` or `10 / 3.0`.
2. Ranges `a..b` exclude `b`; there is no `...` range.
3. A real zero-arg method that is not auto-called prints `<method array.tally>`:
   add `()`. A made-up one raises `Cannot access property 'upper' on String`
   (since 2.34.0; earlier it was a silent `<method …>` too).
4. `soli script.sl` / `soli check` run a static type checker. It accepts any
   value as a condition and widens an unannotated local reassigned with another
   type; it refuses a value of another type only for a binding whose type you
   wrote (`let x: Int`, a typed parameter), or for a bare `name = …` that
   would rebind a `def` of the file, or a builtin at the top level (inside a
   function a builtin's name is just a local). `T?` takes `nil`.
5. `def name` (no parens) whose first body line starts with `(` → parse error.
   Write `def name()`.
6. Brace-less `fn(x) x * 2` swallows the following statements.
7. `for k, v in hash` gives key + **index**, silently. Use `h.each do |k, v|`
   (`for k in hash` walks the keys).
8. `h["x"]["y"] ?? d` raises when `h["x"]` is nil — use `h.dig("x", "y") ?? d`
   or `h["x"]&.y`.
9. Truthiness: `false`, `nil`, `0`, `""`, `[]`, `{}` are falsy; `0.0` is truthy.
    `x || default` replaces `0` and `""`; use `x ?? default` for nil-only.
10. `<%= h(x) %>` and `<%= attr(x) %>` double-escape — `<%=` already escapes.
11. `Model.find` raises (→ 404 in a request); `find_by`/`first_by` return nil.
    Never nil-check after `find`.
12. Migrations: `soli db:migrate generate <name>` — `soli generate migration`
    does not exist.
13. In a block, `next` skips one element (`map` keeps a nil for it) and `break`
    stops the whole iterator, which answers nil (since 2.36.0). On 2.36.0–2.36.2
    under `soli serve` a `next` in a block ended the loop — run ≥ 2.36.3.

## Framework at a glance

```soli
# config/routes.sl
get("/", "home#index")
resources("posts")          # index/new/create/show/edit/update/destroy + posts_path, post_path(p), ...
get("/about", "pages#about", name: "about")   # about_path()
middleware("authenticate", -> {
  get("/admin", "admin#index")
})
```

```soli
# app/controllers/posts_controller.sl — models auto-load, no import
class PostsController < Controller
  static {
    this.layout = "application"
    this.before_action(:show, :edit, :update, :destroy) = fn(req) {
      @post = Post.find(params["id"])
      req
    }
  }

  # GET /posts
  def index
    @posts = Post.order("title").all     # implicit render of posts/index
  end

  # POST /posts
  def create
    @post = Post.create(@_permit_params(params))
    return render("posts/new", {}, {"status": 422}) if @post._errors

    redirect(post_path(@post))
  end

  private

  def _permit_params(params)
    permit(params, {"title": true, "body": true})
  end
end
```

```soli
# app/models/post.sl
class Post < Model
  has_many("comments")
  validates("title", {"presence": true, "min_length": 3})
  before_save("normalize")
  scope("published", fn() { this.where({"status": "published"}) })

  def normalize
    @title = @title.trim unless @title.blank?
    true                     # a before_* callback returning false ABORTS the save
  end
end
```

```erb
<%# app/views/posts/index.html.slv %>
<% for post in @posts %>
  <a href="<%= post_path(post) %>"><%= post.title %></a>
<% end %>
<%- form_with(@post) do |f| -%>
  <%- f.text_field("title") %><%- f.errors_for("title") %><%- f.submit("Save") %>
<%- end -%>
```

```soli
# tests/controllers/posts_controller_spec.sl
describe("POST /posts") do
  before_each() do
    as_guest()
  end

  test("refuses a post without a title, and writes nothing") do
    count_before = Post.count
    response = post("/posts", {"body": "x"})
    assert_eq(res_status(response), 422)
    assert_eq(Post.count, count_before)
  end
end
```

## Reference index

Read the file for the task at hand. **Before hand-writing a string, array or
hash helper (padding, casing, slicing, grouping…), check
`references/stdlib-builtins.md`**: it is probably built in (`"42".lpad(5,
"0")`, `rpad`, `center`). An unknown method's error names the Soli equivalent
of a JS/Python name (`pad_start` → `did you mean 'lpad'?`) — read the hint.

| File | When |
|---|---|
| `references/language.md` | syntax, types, collections, blocks/lambdas, classes, modules/imports, pattern matching, errors, strings, truthiness |
| `references/stdlib-builtins.md` | String/Array/Hash/number methods, JSON, XML (parse, find/xpath, build, stream), DateTime/Duration, Regex, Crypto (TOTP, `qr_svg`), passwords, JWT, SAML, passkeys (`WebAuthn`), Base64, files, env, HTTP, app metrics (`Metrics`), V validation, realtime (WebSockets, `broadcast`, `SOLI_REALTIME_BUS`) |
| `references/controllers-routing.md` | routes, resources/helpers, actions, request access, response shapes, before_action, halt, JSON APIs, uploads |
| `references/models-orm.md` | Model API, validations, callbacks, associations, I18n (validation messages, names, translated fields), query builder, aggregates, grouped, transactions, migrations' `db` API, SQLite setup |
| `references/views-forms.md` | ERB `.html.slv`, escaping, loops, partials, layouts, form_with, button_to, csrf, components |
| `references/testing.md` | spec DSL, hooks, assertions, E2E client, assigns/view_path, coverage, .env.test |
| `references/middleware-sessions-security.md` | middleware directives, sessions, cookies, flash, CSRF, permit, SSRF |
| `references/cli.md` | `soli new/serve/test/fmt/lint/check/routes/db:*`, generators (`auth --passkeys/--two-factor`, `saml`, `audit`, `oauth`, scaffold), profiler, `soli dap`/`lsp`, code graph |
| `references/traps.md` | the footgun cheatsheet — skim before writing code |

Every Soli app also ships its own docs: `docs/*.md` in the app (refresh with
`soli update docs`) and per-directory `CLAUDE.md` files. `grep -ril <topic> docs/`
before writing infrastructure by hand (2FA, uploads, PDFs, jobs, rate limits…).
Treat those docs as helpful but not infallible — see "Doc drift" in traps.md.
