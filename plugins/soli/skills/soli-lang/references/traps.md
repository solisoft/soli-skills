# Footgun cheatsheet (rows verified on soli 2.28.2; rechecked on 2.36.3)

## Ruby / JS habits that fail

| You'd write | Soli | What happens otherwise |
|---|---|---|
| `raise "x"` / `raise("x")` | `throw "x"` | `Undefined variable 'raise'` |
| `(1..50).map` for 1–50 | `(1..51).map` | ranges exclude the end; `1...3` is a parse error |
| `10 / 3 # => 3.33` | `10.to_f / 3` | Int / Int is integer division (`3`) |
| `xs.reduce(0) { \|a, x\| a + x }` | `xs.reduce(fn(acc, x) acc + x, 0)` | "reduce expects a function argument" |
| `xs.reduce(fn(acc) …, 0)` expecting `acc` | `fn(acc, x)` | a one-parameter reduce callback gets the `[acc, x]` pair |
| `rows.group_by { \|r\| r.k }` | `rows.group_by("k")` | group_by/sum_by/max_by/min_by take a field name string |
| `for k, v in hash` | `hash.each do \|k, v\| … end` | `v` is an index (0, 1, …), silently |
| `hash.map { \|k, v\| v * 2 }` | `hash.values.map { \|v\| v * 2 }` | Hash#map wants `[k, v]` pairs and returns a hash; otherwise `{}` |
| `h["a"]["b"] ?? d` | `h.dig("a", "b") ?? d` or `h["a"]&.b` | raises `Cannot access property 'get' on null` |
| `h.has_key?("a")` / `key?` | `h.has_key("a")` | no such method |
| `s.upper` / `s.lower` | `s.upcase` / `s.downcase` | static type error, or `Cannot access property 'upper' on String` at runtime |
| `s[1..3]` / `s.slice(0, 3)` | `s.substring(0, 3)` | type error |
| `"%.2f" % x`, `format(...)` | `x.round(2)` / `I18n.format_number` | not supported |
| `case x when …` | `match x { … }` | parse error |
| `loop do … end` | `while true … end` | parse error |
| `"\(name)"`, `"${name}"` | `"#{name}"` | lexer error / literal text |
| `@"raw"`, `[[raw]]` | `"""raw"""`, `r"raw"` | parse error / nested array |
| `def *args` | pass an array | parse error |
| `super()` in a normal method | `super.method_name()` | `super()` does not call the parent method |
| `interface X … end` (2.39.5 and earlier) | `interface X { def m() -> T; }` | parse error — accepted after 2.39.5 |
| `(-3).abs` written `-3.abs` | `(-3).abs` | `-3.abs` is `-(3.abs)` = -3 |

## Soli-specific traps

| Trap | Do this |
|---|---|
| In an `if`/`unless`/`elsif`/`while`/`for`/`match` head a `{` with no `\|` after it is the statement's brace body, so `if xs.map { 5 }.length > 0` is a parse error (a `{ \|x\| … }` block works there since card #657) | give the block a parameter: `{ \|x\| … }` |
| A zero-arg method that is not auto-called prints `<method array.tally>` | add `()`: `xs.tally()` |
| Brace-less `fn(x) x * 2` / `\|x\| x * 2` on its own line swallows the next statements | always `fn(x) { x * 2 }` |
| Last line of a method starting with `fn(` is read as a named declaration (parse error) | `return fn(x) { … }` |
| `def name` without parens whose first body line starts with `(` | `def name()` |
| Pattern arm `n if n > 0` before `[a, ...rest]` raises on array input | structural arms first |
| `"#{"a #{x}"}"` (interpolation inside a string inside interpolation) | build the inner string first |
| Assigning an outer variable's name inside a block mutates the outer one | rename, or `let` |
| A local named after a builtin (`options = …`, `stream = …` in a method, lambda or block) is a plain local on both engines — the builtin is fine outside the body — but calling `options(…)` in that same scope reaches the local; `soli lint` flags it as `smell/shadowed-builtin`. At the file's top level the assignment rebinds the builtin for the whole program, and `soli check` refuses it ("assigns a String to the builtin page_url()") | rename the local (`route_options`, `items`), or `let options = …` to shadow on purpose |
| A local named after a `def` of the same file (`helper = 5` inside another def or a method): both engines rebind the global function, so a later `helper()` is "Cannot call non-function value"; `soli check` refuses it ("assigns a Int to the function helper() this file defines") | rename the local, or `let helper = …` to shadow |
| Truthiness: `0`, `""`, `[]`, `{}` are falsy, `0.0` is truthy; `x \|\| d` replaces 0/"" | `x ?? d` for nil-only defaults |
| `HTTP.get` / `get_json` raise on non-2xx | `HTTP.request(...)["status"]` to branch |
| `params.keys` / `body.length` on a hash you did not build (params, JSON) | `params.keys()` / `body.length()`, or brackets — a bare read answers a key of that name first |
| `super.m()` inside a block of a method (`xs.map { … super.m(x) … }`) raises "super used outside of a subclass method" on the VM (scripts, `soli serve`); `--tree` works | call super before the block (`doubled = super.m(x)`) and use the local in the block |
| An enum rebuilt from storage (`enum_field` read, `Status.parse`) is a new instance: `==`/`!=` compare it by value, but `[Status.Paid].includes?(stored)` is false (`includes?`, `index_of`, `uniq`, array/hash `==` compare enum elements by identity) | `xs.any? { \|s\| s == stored }` |
| Closures made in an `each`/`map` block over its parameter, called later: one per element on the VM (scripts, production), all the last element under `--tree`/`--dev`/`soli test` | build them through a function, or keep the block's result instead |

## Framework traps

| Trap | Do this |
|---|---|
| `soli generate migration x` doesn't exist | `soli db:migrate generate x` |
| `Model.find(id)` raises → 404 | never nil-check after it; `find_by`/`first_by` for nil |
| `before_*` callback whose value is `false` aborts the save (`@flag \|\|= false` as last line!) | end callbacks with `true` |
| `record.update(attrs)` skips callbacks | assign fields, then `record.save` |
| `_errors` is `nil` after a good `create` but `[]` after a good `save`/`update` (both falsy: `if record._errors` works; the app CLAUDE.md wrongly calls `[]` truthy) | `_errors.length > 0` is the explicit form |
| `.exists`, `.pluck`, `.sum(...)` return a QueryBuilder (always truthy) | `.exists.first`, `.pluck("f").all`, `.sum("f").first` |
| A misspelled validation option (`"presense"`) stops the app/test server from booting | read the boot error |
| `resources("posts")` routes DELETE to `destroy`; scaffold writes `delete` | keep action names in sync with `soli routes` |
| `record.destroy` (a Rails habit) | a model has no `destroy`: it **raises** (after 2.40.0; it used to return nil and leave the row). Write `record.delete` |
| `record.save` / `update` on a row deleted meanwhile | writes nothing and answers `false` (after 2.40.0, every adapter; the SQL adapters used to re-insert the row). `touch` raises; `Model.update(id, …)` returns the `"Error: … not found"` string. Create-if-missing is `Model.upsert(key, data)` |
| `redirect("https://…")` raises (500) | local paths only (`redirect_external` per docs) |
| `halt` / `forbidden()` / `find` miss inside `rescue`/`try` get swallowed | keep them outside |
| `<%= h(x) %>` / `<%= attr(x) %>` escape twice | plain `<%= x %>` |
| `<%= partial(...) %>` shows markup as text | `<%- partial(...) %>` |
| Model classes in `.slv` → 500 | compute in the controller |
| Two files with the same top-level `def`/`class` → `soli serve` refuses to boot | put shared logic on a class |
| `soli test` without `.env.test` exits immediately — apps made by `soli new` up to 2.34.0 have none | create it (SQLite works); later `soli new` writes one |
| Form POST from a browser without CSRF token → 403 (`SOLI_CSRF_TOKENS=require` in new apps) | `form_with` / `csrf_field()` |
| `post = …` in a spec replaces the `post()` helper | `post_record = …` |
| `set_request_cookie("locale", "fr")` then `set_request_cookie("locale", "en")` in one spec on soli 2.39.1–2.40.1: the request carries both cookies and the server keeps the first, so the second value never arrives | soli 2.40.2+ replaces the cookie; on older, switch back through a real request that sets it |
| Records persist between tests in a worker | reset state in `before_each` |
| `@_private` controller fields are still readable as `@_private` in the view | don't rely on the underscore to hide data from templates |
| An action, hook or middleware that prints, mails, calls an API or writes the cache, then raises: in production the request is re-run on the interpreter (unless it wrote to the database first), so the effect happens twice. Since 2.29.4 a request that ran out of `SOLI_HANDLER_TIMEOUT_SECS` is not re-run: it answers 500 at the budget, and a `rescue` cannot keep it looping | raise before the side effect, or catch the error in the action (`try`/`rescue`) |
| A helper that reads `req`, `params`, `session`, `cookies` or `headers`, called from a custom `errors/404.html.slv` for a path no route matches: they are nil there (no request exists yet) | guard it: `return "guest" if req.nil?` |
| Production keeps rendered pages per worker, keyed on the view, layout, locale and render data: a view that reads the database, a file or the time itself (`<%= DateTime.now.year %>`) shows its first value | compute it in the action and pass it as an `@field`; not cached at all: requests with a session, renders that set a cookie/session/flash or embed a CSRF token, data holding a record or other object |
| A helper in `app/helpers` calls a function only views have: `substring(...)`, `t(...)`, `l(...)`, `datetime_format(...)`, `public_path(...)`, `number_with_delimiter(...)`, `paginate(...)`, `component(...)`, `form_with`/`csrf_field`/`button_to`, a route helper (`posts_path()`) — "Cannot call non-function value", 500 | string methods (`text.substring(0, 3)`, `text.upcase`), or compute the value in the view and pass it in |
| In an app made by `soli new` up to 2.29.3, the starter `truncate_text(...)` (when it cuts) and `capitalize(...)` answer 500 | replace their bodies: `return text if len(text) <= length` then `text.substring(0, length - len(suffix)) + suffix`; `text.substring(0, 1).upcase + text.substring(1, len(text))` |

## Doc drift (bundled docs vs the 2.36.3 binary)

The app's `CLAUDE.md`/`docs/` are mostly right, but these statements were
still wrong when tested on 2.36.3 (`soli new` output, SQLite):

- `docs/soli-language.md`: "`/` always returns a float" (`10 / 3` is `3`), and
  `.upper()` / `.snake_case()` examples (no such methods: `upcase`/`downcase`).
- `CLAUDE.md`: "`x || default` is wrong because 0 and \"\" are TRUTHY" — they
  are falsy; prefer `??` anyway, the reason is inverted. Also "run `soli
  generate` with no argument to see the list" — it prints the general help.
- `docs/models.md`: "has_many returns a plain array" (a QueryBuilder).
- `CLAUDE.md`: "`_errors` is `[]` after a successful `save` — `[]` is truthy" —
  `[]` is falsy, `if record._errors` is fine.
- `app/controllers/CLAUDE.md`, `app/views/CLAUDE.md`: "`@_internal` fields are
  not exposed to the view" — `<%= @_secret %>` renders.
- Apps whose docs predate 2.35/2.36 also say "Always use `h()`" (double
  escapes), "`[[` opens a raw string" (nested array), and
  show `raise(...)` (use `throw`).
