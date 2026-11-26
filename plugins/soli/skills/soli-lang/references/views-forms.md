# Views and forms (verified on soli 2.28.2; rechecked on 2.36.3)

Templates are ERB-style `.html.slv` files in `app/views/<controller>/<action>.html.slv`.
Layout: `app/views/layouts/application.html.slv`, which outputs the page with
`<%= yield %>`.

## Tags

| Tag | Behaviour |
|---|---|
| `<%= expr %>` | evaluate and HTML-escape |
| `<%- expr %>` | evaluate, output raw (framework-built HTML: partials, form builder, `csrf_field()`) |
| `<% stmt %>` | code, no output |
| `<%# comment %>` | dropped |
| `-%>` | swallows the following newline |

**Never wrap in `h()` or `attr()` inside `<%= %>`** — it escapes twice
(`<%= h("<i>") %>` → `&amp;lt;i&amp;gt;`; `<%= attr("/a?x=1&y=2") %>` →
`&amp;amp;`). `soli lint` flags it (`idiom/redundant-template-escape`).
Untrusted HTML through `<%-` must go through `sanitize_html(...)` first
(documented, not verified here).

## Data

Controller `@fields` are readable as `@title` or bare `title` (verified both).
Partials and layouts see them too. `@x ?? "fallback"` works for missing ones.

In production (`soli serve` without `--dev`) views compile to the VM; a
layout, a template too small to gain from it, and a view the VM fails on
render on the interpreter instead. `SOLI_ENGINE_LOG=1 soli serve .` names each
one once on stderr with the reason (`[soli engine] view 'posts/_post.html.slv'
renders on the interpreter: too small to gain from the VM`), and the view
helpers that run there (`view helper 'x' runs on the interpreter: …`).

## Control flow

```erb
<% for post in @posts %>
  <article><a href="<%= post_path(post) %>"><%= post.title %></a></article>
<% end %>

<% for item, i in @items %><li><%= i %>: <%= item %></li><% end %>

<% @items.each do |item| %><li><%= item %></li><% end %>   <%# also works %>

<% if @posts.length > 0 %>
  ...
<% else %>
  <p>No posts yet.</p>
<% end %>

<% unless @missing %>...<% end %>
```

## Partials

File name starts with `_`: `app/views/home/_card.html.slv`.

```erb
<%- partial("home/card", {"label": "L1"}) %>
```

```erb
<%# _card.html.slv — locals are bare names; controller @fields are visible %>
<span class="card"><%= label %>/<%= @title %></span>
```

Use `<%-`: `<%= partial(...) %>` escapes the partial's markup into visible
text. Pass the form builder to a form partial: `partial("posts/form", {"f": f})`.

`partial()` / `render_partial()` / `component()` never modify the hash you
pass, from a view, a helper, a service or an action. Inside the partial the
passed locals win, then the controller's `@fields`, then helpers; `locals`
holds the passed keys plus the inherited `@fields` (not the helpers).

## Helpers available in views

`posts_path()`, `post_path(post)`, `new_post_path()`, `edit_post_path(post)`,
named routes (`about_path()`), `csrf_field()`, `csrf_meta_tag()`,
`json_script(value)`, `button_to(...)`, `form_with(...)`, `time_ago(unix)`,
anything defined in `app/helpers/*.sl`. There is no builtin `link_to` (an app
made by `soli new` has `link_to(text, url)` in its starter helpers) — else
write `<a>`. A helper of yours wins over a framework name (`button_to`,
`csrf_field`, `posts_path`) in every view and partial, `<%= render %>`
included; `form_with`/`button_to` still embed the framework's CSRF field.

A top-level `const` in a helper file is visible to the helpers
(`const STATUSES = ["up", "down"]` + `def status_name(i) STATUSES[i] end`),
evaluated once at load after every helper is defined (since 2.33.0;
up to 2.32.0 it read as nil — "cannot index null with int").

`<% h = "x" %>` in a view, or a bare `h = …` in a helper body, is a local of
that render or call even when the name is a builtin's or a helper's: the next
render's `h("<b>")` still escapes (verified on 2.29.4; up to 2.29.3 it
replaced `h` for every later render on the worker).

Model classes are **not** reachable from templates (`<%= Post.count %>` → 500).
Compute in the controller and pass a value.

JSON for client JS:

```erb
<script>window.data = <%- json_script(@data) %>;</script>
<%# {"a":"</script>"} renders as {"a":"</script>"} %>
```

## Forms — `form_with`

```erb
<%- form_with(@post) do |f| -%>
  <%- f.error_summary() %>
  <%- f.label("title") %>
  <%- f.text_field("title", {"placeholder": "Title"}) %>
  <%- f.errors_for("title") %>
  <%- f.text_area("body") %>
  <%- f.submit("Save") %>
<%- end -%>
```

Verified output:

- New record (`Post.new()`) → `<form action="/posts" method="POST">`.
- Persisted record → `action="/posts/<key>"` plus a hidden `_method=PATCH`.
- Every form embeds `_csrf_token`, a honeypot field and a signed `_soli_ts`.
- Values prefill from the record and are escaped.
- After a failed create, errored fields get `class="field-error"
  aria-invalid="true"`, `errors_for` emits `<span class="field-error-message">`
  (the bare message), `error_summary` a `<div class="form-errors"><ul>…</ul></div>`
  of full sentences naming the field ("Title can't be blank"; `_base` alone).
  Since main after 2.42.3 — before, `error_summary` listed bare messages.
- `f.label("title")` reads `models.<model>.attributes.title` from the locale
  files when the record is a model (verified served: `<label for="title">Titre</label>`).
- `f.text_field("title", {"locale": "fr"})` on a translated field →
  `id="title_fr" name="translated_fields[title][fr]"`, prefilled with that
  translation (any field helper, and `label(…, {"locale": "fr"})` for `for=`).

Other field helpers (from the docs): `email_field`, `password_field`,
`number_field`, `date_field`, `datetime_field`, `hidden_field`, `file_field`
(+ `{"multipart": true}` on the form), `check_box`, `radio_button`, `select`,
`f.fields_for("author") do |author| … end` (bracket names nest:
`author[name]` → `params["author"]["name"]`). An unchecked `check_box`
submits nothing. Repeated fields need `name="field[]"`.
Without a record, `form_with({"url": "/search", "method": "get"}) do |f| … end`
takes the hash as options (since 2.36.0; before, it was read as a record).

Delete buttons (never a bare `<a>`):

```erb
<%- button_to("Delete", post_path(@post), {"method": "delete", "confirm": "Sure?"}) %>
<%# → <form action="/posts/<key>" method="POST"><input type="hidden" name="_method" value="DELETE">
      … CSRF … <button type="submit" data-confirm="Sure?">Delete</button></form> %>
```

Hand-written form: put `<%- csrf_field() %>` inside it. Layouts:
`<%- csrf_meta_tag() %>` → `<meta name="csrf-token" content="…">` for
fetch/htmx (`X-CSRF-Token` header).

## Layouts

- Per controller: `static { this.layout = "admin" }` → `layouts/admin.html.slv`.
- Per render: `render("misc/bare", {"layout": false})` (verified).
- Generated layouts carry Open Graph/Twitter share tags using `title` and
  `description` — set `@title` and `@description` in each page action.
- `soli serve` injects `/__soli/nav.js` (instant navigation: link clicks swap
  `<body>`). External `<script src>` runs once per tab — re-initialise widgets
  on `document.addEventListener("soli:load", init)`. Opt out per page with
  `<meta name="soli-nav" content="off">` (documented).

## Components

`soli generate component post_card` writes
`app/views/components/post_card.html.slv`; render with
`<%- component("post_card", {"title": "x"}) %>` (generator output; rendering
not verified here). `component("card", {"post": post, "cache": true})`
memoizes the HTML in the KV cache under a key derived from the data (a record
by its class and fields; its relations are not part of the key, so bound it
with `"cache_ttl"`); `"cache": "name"` sets the key. Every key includes the
active locale.
A component file needs no leading `_`, and may have one: `component("x")` and
`partial("components/x")` both find `components/x.html.slv`, else
`components/_x.html.slv` (the `_` fallback since 2.33.0). Only a
call opens a component block (`component(…) do`, `component "card", … do`);
a block variable named `component` (`<% component["list"].each do |x| %>`) is
an ordinary variable (up to 2.32.0 that line failed with "component name must
evaluate to string").

## Formatting

`soli fmt` does not touch `.html.slv`; indent templates by hand (2 spaces).
`soli lint` does check them.
