# Controllers and routing (verified on soli 2.28.2 with a throwaway app; spot-checked on 2.36.3)

## Routes — `config/routes.sl`

```soli
get("/", "home#index")
get("/health", "home#health")
get("/about", "pages#about", name: "about")   # → about_path() / about_url()
post("/api/posts", "posts#api_create")
resources("posts")

middleware("authenticate", -> {               # opt-in scoped middleware
  get("/admin", "home#admin")
  namespace("admin", -> {                     # admin/users#index, admin_users_path (unverified)
    resources("users")
  })
})
```

`resources("posts")` expands to (check with `soli routes`):

| Verb | Path | Action | Helper |
|---|---|---|---|
| GET | /posts | posts#index | `posts_path()` |
| POST | /posts | posts#create | |
| GET | /posts/new | posts#new | `new_post_path()` |
| GET | /posts/:id | posts#show | `post_path(post)` |
| PUT/PATCH | /posts/:id | posts#update | |
| POST | /posts/:id/update | posts#update | |
| DELETE | /posts/:id | posts#**destroy** | |
| POST | /posts/:id/delete | posts#destroy | |
| GET | /posts/:id/edit | posts#edit | `edit_post_path(post)` |

The delete action is **`destroy`** under `resources`. (`soli generate scaffold`
writes explicit routes and a `delete` action instead — keep them consistent.)
`post_path(record)` takes the record (uses `id`, else `_key`). Route helpers
are bare functions: call them with `()` (`posts_path()`).

## Controller anatomy

```soli
class PostsController < Controller
  static {                       # braces REQUIRED; `static … end` does not parse
    this.layout = "application"

    # hooks are wired by a startup scan of this exact syntax; restart after editing
    this.before_action(:show, :edit, :update, :destroy) = fn(req) {
      @post = Post.find(params["id"])    # a miss raises → 404
      req                                # return req to continue
    }
  }

  # GET /posts
  def index
    @posts = Post.order("title").all
    @title = "Posts"
  end

  # GET /posts/:id
  def show
    @title = @post.title
  end

  # GET /posts/new
  def new
    @post = Post.new()           # () required
  end

  # POST /posts
  def create
    @post = Post.create(@_permit_params(params))
    return render("posts/new", {}, {"status": 422}) if @post._errors

    redirect(post_path(@post))
  end

  # GET /posts/:id/edit
  def edit
  end

  # PATCH /posts/:id
  def update
    @post.update(@_permit_params(params))
    return render("posts/edit", {}, {"status": 422}) if @post._errors.length > 0

    redirect(post_path(@post))
  end

  # DELETE /posts/:id
  def destroy
    @post.delete
    redirect(posts_path())
  end

  private

  def _permit_params(params)
    permit(params, {"title": true, "body": true})
  end
end
```

- Actions may omit `(req)`; `req` and `params` are available as globals.
  `def index(req)` also works.
- Every public action gets a `# VERB /path` comment. Helpers start with `_`
  and sit under `private`.
- Models are auto-loaded: no `import` (lint: `style/redundant-model-import`).
- **`_errors` after `update`/`save` is `[]` on success, `nil` after a
  successful `create`.** `if @post._errors` is right after `create`; after
  `update` test `.length > 0`.

## Reading the request

| Read | Gives |
|---|---|
| `params["id"]` | route + query + body params merged |
| `req["json"]` | parsed JSON body (`?? {}` when absent) |
| `req["headers"]["x-api-key"]` | lowercase header names |
| `req["method"]`, `req["path"]` | |
| `cookies["theme"]` / `cookies.theme` | global, parsed request cookies |
| `find_uploaded_file(req, "photo")` | upload hash or nil — pass `req`, not `params` (unverified here) |

## Response shapes

| Return | Result |
|---|---|
| nothing special (sets `@fields`) | implicit render of `views/<controller>/<action>.html.slv` |
| `render("posts/new")` | another view, same `@fields` |
| `render("posts/new", {}, {"status": 422})` | with status |
| `render("feed", {}, {"headers": {"Cache-Control": "public, max-age=60"}})` | extra headers, replacing a default of the same name (any case); options are `layout`/`status`/`headers` only — another key, a CR/LF in a value or a non-token name raises (unreleased after 2.32.0; up to 2.32.0 `headers` was silently dropped) |
| `render("misc/bare", {"layout": false})` | no layout |
| `redirect(post_path(@post))` | 302; **local paths only** — a full URL raises (500) |
| `render_json(data)` / `render_json(data, 201)` | JSON, `Content-Type: application/json` |
| `render_text("OK")` | text/plain |
| `{"status": 200, "headers": {...}, "body": "..."}` | raw response (any hash with `"status"`) |
| `halt(403, "Forbidden here")` | raises → 403 text/plain, no `return` needed |
| `throw SomeDomainError.new("msg")` | 422 (class `< DomainError`) |
| `Post.find(missing)` | 404 |

`render("v", data)` never modifies `data`: `req`, `params`, `cookies` and the
`@fields` go into a copy, so a `const` of defaults is safe to pass.

`halt`, `forbidden()` and a `find` miss are exceptions: a `rescue`/`try`
around them swallows them and the request carries on.

Content negotiation (verified):

```soli
# GET /misc/neg
def neg
  respond_to(req, {
    "html": fn() { render_text("html version") },
    "json": fn() { render_json({"v": "json"}) }
  })
end
```

## View data

Every `@field` reaches the view and the layout as `@field` and as bare
`field`. Don't pass a data hash to `render` for view data. Specs see them via
`assigns()`.

## JSON API example (verified)

```soli
# POST /api/posts
def api_create
  data = req["json"] ?? {}
  post_record = Post.create(permit(data, {"title": true, "body": true, "meta": {"tag": true}}))
  return render_json({"errors": post_record._errors}, 422) if post_record._errors

  render_json({"key": post_record._key}, 201)
end
```

`permit` drops unlisted keys, including unlisted keys inside nested hashes
(`{"meta": {"tag": true}}` keeps `meta.tag`, drops `meta.bad`). Describe nested
shapes fully: a container declared as `true` is dropped (per framework docs).

JSON requests are not CSRF-token-gated; browser form posts are (see
middleware-sessions-security.md).

## Grouping reads

```soli
# GET /
def index
  grouped(fn() {
    @posts = Post.all
    @total = Post.count
  })
  # read @posts / @total only AFTER the block
end
```

## Function-style controllers and other files

Top-level `def` actions in `app/controllers/*.sl` also work (per docs,
unverified here). Shared logic belongs on a base controller/model/service —
two files declaring the same top-level name make `soli serve` refuse to boot.
`app/helpers/*.sl` functions are visible in views only; model classes are not
resolvable from `.slv` templates — compute in the controller.
