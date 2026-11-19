# Testing (verified on soli 2.28.2; rechecked on 2.36.3)

Specs are `tests/**/*_spec.sl`, run with `soli test`. The runner needs a
**`.env.test`** in the app root (it exits with ".env.test file not found"
otherwise). With SQLite:

```bash
# .env.test
SOLI_DB_ADAPTER=sqlite
DATABASE_URL=sqlite://db/test.sqlite3
```

It creates the test database, runs `db/migrations`, starts real test
server(s), runs specs, and drops the database. Models, helpers, middleware,
services, policies, mailers, jobs and subscribers are preloaded so specs call
them directly (mailers and subscribers since 2.33.0, #676 —
before it a service using a mailer raised "Undefined variable"). Since
2.35.0, controllers too: an EUI handler/view or a function-style
action is callable from a spec — except a controller file whose top-level
`def` is named like a spec-client function (`get`, `post`, `delete`…), which
is skipped so it cannot replace it.

A fresh app (since 2.35.0) ships specs for what `soli new` wrote — home
controller, application helper, `authenticate` middleware, the `--eui` counter
— and scaffolds bring a passing controller spec: `soli test` starts at 100%.
`# coverage: off` in a file's leading comment block drops it from the report
(the EUI catalogue carries it). Without `--coverage-min`, a report below 80%
prints ❌ but exits 0.

```bash
soli test                              # all of tests/, coverage on (threshold 80%)
soli test tests/models/post_spec.sl    # one file (or a directory)
soli test --no-coverage                # fast loop
soli test --filter "json"              # describe+test name contains text
soli test --coverage-min 90.0 --show-uncovered
soli test --fail-fast --require-assertions --jobs 4
```

A failing test is listed with its error text (unchanged, so specs matching
messages keep working) and, when the error points into the spec, a code frame
of the failing line with "did you mean" and the error code (since 2.36.0).

`print` output from specs is not shown in the report. To debug, append to a
file: `File.append("/tmp/dbg.log", msg + "\n")` (the runner is not jailed).

## DSL

```soli
post_record = nil            # shared state lives at the top level

describe("Post model") do
  before_each() do           # hooks need the ()
    Post.all.each do |p|     # records persist between tests in a worker: reset here
      p.delete
    end
    post_record = Post.create({"title": "Hello", "body": "x"})
  end

  test("is persisted") do
    assert_not_null(post_record._key)
  end

  test("refuses a missing title, and stores nothing") do
    bad = Post.create({"body": "x"})
    assert_eq(bad._errors[0]["field"], "title")
    assert_null(bad._key)
  end

  test("find raises on an unknown key") do
    message = assert_raises() do
      Post.find("no-such-key")
    end
    assert_contains(message, "no-such-key")
  end

  test("later") do
    pending("not yet")         # counted as pending, not failed
  end
end
```

Rules the runner enforces (from the docs; it reads declarations without
running `describe` bodies): a variable assigned in a `describe` body never
reaches its tests; `test` outside `describe`, non-literal names, two hooks of
the same kind in one `describe`, or `describe` inside an `if` fail the file.
Aliases: `context`, `it`, `specify`; hooks `after_each`, `before_all`,
`after_all`.

## Assertions (verified subset)

```soli
assert(cond)                     # must be the Bool true
assert_eq(actual, expected)      # "expected 200, got 404"
assert_ne(a, b)
assert_null(v) / assert_not_null(v)
assert_contains(string_or_array, item)
assert_hash_has_key(hash, key)
message = assert_raises("boom") do   # fragment optional; returns the message
  throw "boom here"
end
expect(1 + 1).to_equal(2)
expect("hello world").to_match("world")   # substring, not regex
expect([1, 2]).to_contain(2)
```

Also verified: `assert_not(false)`, `assert_gt(3, 2)`, `assert_lt(1, 2)`,
`assert_match("abc123", "\\d+")` (regex), `assert_json("{\"a\": 1}")`, and a
trailing message argument (`assert_eq(count, 3, "the cart size")`).
Documented: `assert_query_count(response, n)`, `assert_max_queries(response, n)`.

## E2E controller specs

```soli
describe("PostsController") do
  before_each() do
    as_guest()
  end

  test("lists posts") do
    Post.create({"title": "First", "body": "x"})
    response = get("/posts")
    assert_eq(res_status(response), 200)
    assert_eq(view_path(), "posts/index.html")
    assert_hash_has_key(assigns(), "posts")
    assert_contains(res_body(response), "First")
    assert_no_n_plus_one(response)
  end

  test("creates and redirects") do
    response = post("/posts", {"title": "Hello", "body": "World"})
    assert_eq(res_status(response), 302)
    assert(res_redirect?(response))
    assert_eq(Post.where({"title": "Hello"}).count, 1)
  end

  test("refuses invalid data, writes nothing") do
    count_before = Post.count
    response = post("/posts", {"body": "World"})
    assert_eq(res_status(response), 422)
    assert_contains(res_body(response), "field-error")
    assert_eq(Post.count, count_before)
  end

  test("404 for an unknown id") do
    assert_eq(res_status(get("/posts/no-such-key")), 404)
  end

  test("JSON API") do
    set_header("Content-Type", "application/json")
    response = post("/api/posts", {"title": "Api", "body": "b"}.to_json)
    clear_headers()
    assert_eq(res_status(response), 201)
    assert_not_null(res_json(response)["key"])
  end

  test("header-gated route") do
    assert_eq(res_status(get("/admin")), 401)
    set_header("x-api-key", "k")
    assert_eq(res_status(get("/admin")), 200)
    clear_headers()
  end
end
```

Verified helpers: `get`, `post(path, hash_or_string)` (a hash body also
arrives as `req["json"]`), `put`, `delete`, `res_status`, `res_body`,
`res_json`, `res_header(r, "Location")`, `res_location`, `res_redirect?`,
`res_not_found?`, `assigns()`, `assign("posts")`, `view_path()`,
`render_template()`, `set_header`, `clear_headers`, `as_guest()`,
`response["query_count"]`.

- CSRF: the request helpers act as a same-origin browser (since 2.33.0, #676).
  `post`/`put`/`patch`/`delete` send `Origin: http://127.0.0.1:<port>`, and a
  form post (`Content-Type: application/x-www-form-urlencoded`/multipart)
  carries the session's token as `X-CSRF-Token`, so specs pass under
  `SOLI_CSRF_TOKENS=require` — no `SOLI_DISABLE_CSRF`. Test refusals by passing
  the header: `{"headers": {"Origin": "https://evil.test"}}` → 403, `"Origin":
  ""` sends none (→ 403 once a cookie is set), `"X-CSRF-Token": ""` on a form
  post → 403 under `require`. On 2.32.0 and older, every unsafe request after
  `login()` (or after any response that set a cookie) was 403 "missing both
  Origin and Referer on cookie-bearing request".
- Signing in (since 2.33.0, #676; verified against SoliDB + `soli generate
  auth`): `login(email, password)`; `as_user(user)` with an Int id, a String
  `_key` or the record (`_key`, then `id`); `as_user(user, {"role": "admin"})`;
  `with_session({"user_id": key})`. All write the test server's session, so
  the next request is signed in under any session driver. On 2.32.0 and older
  `as_user` took an Int only and only set a runner-side marker, and
  `with_session` did not reach the server under the in-memory driver.
- Assert the status of a `post("/login", …)`: a refused sign-in leaves the
  test a guest, and it surfaces later as a 500 from a layout's
  `current_user().email` on `nil`.
- `assigns()` serializes models like `render_json`: `_errors` is not in it.
  Check errors in the HTML (`field-error`) or in a model spec.
- **Don't name a local after a request helper** (`post`, `get`, `delete`…):
  `post = Post.create(...)` replaces `post()`, and the next `post("/posts", …)`
  fails with `Cannot call non-function value`. Use `post_record`.
- **Passkeys in a browser spec** (`soli test --browser`, verified with
  Chromium): `id = virtual_authenticator()` (opts `transport`,
  `user_verification`, `resident_key`) answers the page's
  `navigator.credentials` calls; `virtual_credentials(id)` lists what it holds
  (`credential_id` base64url, `rp_id`, `user_handle`, `sign_count`,
  `resident`); removed before each test. WebAuthn refuses an IP as RP ID:
  `visit(page_url().replace("127.0.0.1", "localhost"))`. The first sign-in
  reports `sign_count` 2 (registration counts), and Chrome refuses a
  discoverable credential on `"usb"` with `user_verification: false`.
- Documented but not verified here: `as_role`, `sign_in`,
  `set_request_cookie`, `requires_solidb()` /
  `requires_solikv()` in `before_each`, `--fail-on-n1`.

## What a spec must prove

Test the refusals next to the success: invalid input (assert 422, which field,
and that nothing was written), a user who may not do it, an unknown id (404,
not 500), the error raised on purpose. Assert the outcome (record, count),
not only the status. Break the code once and watch the spec fail.

## Generated scaffold specs

`soli generate scaffold` writes `tests/controllers/<plural>_controller_spec.sl`,
Up to 2.34.0 it was a placeholder (posted a `name` field, requested
`/posts/1`; 4 of 12 tests failed). Since 2.35.0 it builds records from a
sample value per field and passes on a fresh app (checked on 2.36.3, SQLite:
100% coverage). Still add the refusals your real validations need.
