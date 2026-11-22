# The `soli` CLI (verified on soli 2.28.2; rechecked on 2.36.3)

`soli --help` is the authoritative list. Most commands take an optional
trailing `[folder]` (default `.`).

## Running code

```bash
soli script.sl              # type-check, then run on the VM
soli -e 'print(1 + 1)'      # evaluate a snippet — bare interpreter: no .env, no models
soli runner -e 'print(User.count)'  # after 2.38.1: same, with the app loaded (.env, DB config,
soli runner tmp/fix.sl      #  app/models, app/services; runs from the app's folder; not
                            #  controllers/helpers/routes). `soli runner x.sl <folder>` from elsewhere.
soli --tree script.sl       # tree-walking interpreter (HTTP.* in scripts runs on the VM too since 2.36.0)
soli --no-type-check x.sl   # skip the static checker
soli check app/             # type-check only (exit 1 on errors); a subdir sees the whole app's autoloads
soli check --level strict app/   # off | loose | standard (default) | strict; `# soli: <level>` atop a file wins, `[check] level` in soli.toml is the app default
soli check --no-views .     # views (.slv) are checked by default: helpers, arity, members; @x and passed locals stay open
                            # (models, services, policies, mailers, controllers, middleware, jobs, subscribers,
                            #  config, stdlib — jobs/subscribers since #691, since 2.34.0)
                            # (migrations see `_db`; app/helpers do NOT see view-only helpers like datetime_format)
soli check --format json x.sl   # one JSON document: code, message, file, line, column, hint
soli explain E0301          # what an error code (or lint rule: smell/undefined-local) means; bare = list
soli                        # REPL
soli x.sl --profile         # sampling profiler (#697, since 2.34.0): top-15 self/total
                            #  table on stderr + soli-profile.folded; --profile-out f.json = speedscope;
                            #  --profile-hz N (1-10000, default 1000). Also `soli -e … --profile` and
                            #  `soli test --profile` (per spec file `<main> (file)` roots; not with --watch).
                            #  --profile=alloc: bytes allocated per Soli stack instead of time (same
                            #  outputs, KB table; counts allocated, not live; <main> includes loading).
                            #  Flags after the script stay the runner's until the first script argument.
```

`debug()` (dev error page with a locals snapshot) stops the request — no
pause/step/resume, no DAP yet — and outside `--dev` it fails the request with
a 500 page; never leave it in committed code. The VM does not compile it (the
handler/script falls back to the tree-walker; `--vm` refuses).

An error prints `Error: <message> at L:C` (unchanged, still greppable), then
a code frame (since 2.34.0): ` --> file:L:C`, the source line, a
caret, `= help: did you mean 'x'?` and `= note: error[E0301] … soli explain
E0301`. An error inside an imported module is framed in that module. `soli
check` frames every error of each file. No frame means the file was not
certain. Exit codes unchanged (70 for run, 1 for check).

## New app

```bash
soli new blog < /dev/null   # writes the app (CLAUDE.md, docs/, .claude/ with the solidb skill since 2.36.1, git repo, .env)
```

With stdin redirected from `/dev/null` it finished in about 2 s; if it seems
to hang in an agent shell, it is waiting on stdin — redirect it, or run it in
the background and stop it once `blog/config/routes.sl` exists. Never run `soli new`
or `soli generate` inside another project's tree: generators write into the
**nearest project**, not the cwd.

Fresh app layout (2.36.3): `app/{controllers,models,models/concerns,views,middleware,helpers,jobs,assets}`,
`config/{routes.sl,application.sl}`, `db/{migrations,seeds}`, `public/`, `stdlib/`,
`tests/` (specs for what `soli new` wrote: 100% coverage out of the box since
2.35.0), `docs/` (bundled reference), `CLAUDE.md` + `AGENTS.md` + per-directory
`CLAUDE.md`, `.claude/{commands,skills/solidb,settings.json}`, `.env` (SoliDB settings,
`SOLI_CSRF_TOKENS=require`), `.env.test` (`SOLIDB_DATABASE=<app>_test`; not
written up to 2.34.0, when `soli test` refused to start — see testing.md),
`soli.toml`.

## Server

```bash
soli serve . --dev                  # hot reload, dev bar, X-Soli-* headers, /openapi
                                    # (up to 2.34.0 an edited validates() was ADDED
                                    #  beside the old rule on reload: restart there)
soli serve . --port 5791            # production mode (caching, no reload)
soli serve . --port 5791 --strict-port   # fail instead of scanning for a free port
soli serve . --workers 4
soli serve . -d                     # daemonize (soli.pid / soli.log)
```

- `SOLI_COROUTINES=1` (experimental, ignored under `--dev`): a request waiting
  on an outbound HTTP call or a SoliDB query hands its worker to other
  requests; app code is unchanged. Two requests on one worker can then interleave at those waits,
  so read-modify-write on a class static field is no longer atomic per
  request (request state — session, cookies, `params`, the open
  transaction — stays each request's own). The call still blocks inside a
  transaction, `grouped`, a render or a `stream` block; the SQL adapters
  (Postgres/MySQL/SQLite) still block. `SOLI_WORKER_CONCURRENCY` (default 32) caps requests per worker.

Default port 5011. `before_action` hooks are wired at startup: restart after
changing them, even under `--dev` (per docs).

## Generators

```bash
soli generate scaffold post title:string body:text   # model, controller, views, migration, routes, spec
soli generate component post_card
soli generate mailer welcome
soli generate auth | oidc_provider | saml | audit | oauth github | devices | client android | app_links | offline
soli db:migrate generate create_comments             # migration stub (the ONLY migration generator)
soli db:seed generate demo_posts
```

`soli generate auth --passkeys` (flag before or after the folder; since
2.34.0) also writes `PasskeyCredential` (+ `webauthn_rp_id()`,
`webauthn_origins()`, `webauthn_store_challenge/take_challenge`), passkey
registration at `/passkeys` (`POST /passkeys/options`, `POST /passkeys`,
`DELETE /passkeys/:id`), sign-in at `POST /passkeys/login/options` +
`POST /passkeys/login`, `app/views/passkeys/_sign_in.html.slv`
(`partial("passkeys/sign_in")`), `public/js/passkeys.js`, a migration and
routes. Re-run on an existing auth app adds only those; an existing
`sessions/new` view is never patched. RP ID/origins come from `APP_BASE_URL`
or `WEBAUTHN_RP_ID` / `WEBAUTHN_ORIGINS`. `generate auth --bogus` now errors
("unknown option").

`soli generate audit [--chain]` (since 2.36.3) writes
`app/models/concerns/audited.sl` + `app/models/audit.sl` + a migration.
`include Audited` in a model logs each create/update/delete as an `Audit`
(`action`, `record_type`, `record_key`, `changes` = `{field: [old, new]}`,
`at`, `actor_type/key/label` from `req["current_user"]` or `"system"`,
`request_id`, `ip`). `encrypts` fields and `password_digest` are redacted;
override `static def audit_redacted` / `audit_ignored`. Read with
`record.audits`, `Audit.by_actor(user)`, `Audit.between(from, to)`. `--chain`
adds `seq`/`prev_hash`/`entry_hash` and `Audit.verify` (nil = intact). Not
logged: `update_all`, `delete_all`, `increment`, raw queries.

`soli generate privacy` (unreleased — not in 2.36.3; needs `generate auth`): the register
`PRIVACY_MODELS = {"Order": {"owner": "user_id", "erase": "delete" |
"anonymize" | "keep", "fields": [...], "omit": [...]}}` in
`app/services/privacy.sl`, `Privacy.export_data(user)` (a hash),
`Privacy.erase(user)` (a report of counts; bulk writes, no callbacks), and
`/privacy` for the user. An unlisted model is neither exported nor erased.
`export` is a reserved word: a method cannot be named `export`.

`soli generate scim` (unreleased — not in 2.36.3; needs `generate auth`): a SCIM 2.0
server under `/scim/v2` (`Users`: list, `filter=userName eq "…"`, create,
read, PUT, PATCH in Okta's and Entra's shapes, DELETE; `Groups` the same with
`displayName eq` and member add/remove/replace — `ScimGroup.names_for(user)`,
`const SCIM_GROUP_ROLES = {"Finance": "accountant"}` in
`app/models/scim_group.sl`; a mapping to `admin` is ignored), bearer token per
directory in `SCIM_TOKEN` / `SCIM_TOKENS=name=token,…` (32+ chars), and
`app/middleware/scim_active.sl` (order 8) that makes a deactivated user
(`deactivated_at` set) a guest on every request. After 2.40.0 a directory
reaches only the accounts it provisioned (`scim_provisioned_at`, owner in
`scim_directory`): admins, local accounts and another directory's are not
listed and answer 404 to reads and writes; `SCIM_ADOPT_EXISTING=on` lets it
adopt a non-admin local account, which calls `User#revoke_credentials`; a
write that does not save answers 400/409 (it answered 200 before).

Auth scaffolds after 2.40.0: the confirmation link (`GET /confirm_email`)
shows a button; `POST /confirm_email` confirms (a scanner's GET confirms
nothing); a completed password reset sets `confirmed_at`. OAuth client: no
subject → refused, a non-ASCII address matches no account, an account already
linked at the provider is not linked again by address. OIDC provider: access
tokens carry `sv`/`cid` and are refused at userinfo/introspect once the
session version moves, the user is deactivated or the family is revoked;
`/oauth/logout` forgets remember-me; authorize/device refuse during
impersonation; token endpoints throttled (`OIDC_REQUESTS_PER_IP`…); device
user codes unique.

`soli generate auth --magic-link | --api-tokens | --impersonation` (unreleased —
not in 2.36.3; each adds its own files, edits none). Magic link: `/login/link`,
`MagicLink.issue(user)` → `{token, nonce}`, `MagicLink.redeem(token, nonce)`;
the link only works in the browser holding the `magic_link` cookie. API
tokens: `/api_tokens`, `ApiToken.issue(user, name, scopes, days)` →
`{record, token}`, middleware order 7 sets `current_user()` and
`req["api_token"]` from `Authorization: Bearer soli_pat_…` (401 if unknown),
`api_scope?("write")` in an action (true for a session). Impersonation:
`POST /impersonate/:id` (admin on a non-admin), `DELETE /impersonate`,
helpers `impersonating?()` / `impersonator()`, partial `impersonations/banner`.

`soli generate oauth <apple|github|google|gitlab|discord|linkedin|microsoft>`
(needs `generate auth`; 2.36.3 has github/google/gitlab/discord/linkedin/microsoft;
Apple, account linking and Microsoft multi-tenant are unreleased): `OauthSso` in
`app/services/oauth_sso.sl` (after 2.39.3; it was `OauthClient`, which clashed
with the `oidc_provider` model) — state, PKCE, `find_or_create_user`, `sign_in`,
one `<Name>Oauth` service per provider (`authorize_url(state, challenge)`,
`complete!(code)`), `oauth_controller.sl` (`/auth/:provider`, its callback,
`/oauth/identities` to link/unlink). Accounts it creates are confirmed,
password-less, and only created when `AUTH_SIGNUP` allows. Microsoft: one
tenant (`MICROSOFT_TENANT`) or `organizations` + `MICROSOFT_ALLOWED_TENANTS`.
Apple: `APPLE_CLIENT_ID`, `APPLE_TEAM_ID`, `APPLE_KEY_ID`,
`APPLE_PRIVATE_KEY_FILE` (inside the app — the server reads no file outside
its root) or `APPLE_PRIVATE_KEY`.
After 2.39.3: an address is only ever taken when the provider says it is
verified — GitHub's verified primary from `/user/emails` (and GitHub uses
PKCE), Microsoft's only with the `xms_edov` optional claim true or
`verified_primary_email` (no `preferred_username` fallback; uid is
`tid:sub`; a stored bare-`sub` row is adopted as `tid:sub` when its `raw.tid`
matches, or in a single-tenant app). Linking a provider is
`POST /auth/:provider/link` (`button_to` on `/oauth/identities`; a signed-in
`GET /auth/:provider` just redirects there); it needs a confirmed email and a
sign-in in the last 10 minutes, else `/oauth/reauth` asks for the password and
then continues to the provider; the owner gets an `OauthMailer.provider_linked`
email. Taking over an unconfirmed account calls `user.revoke_credentials`.
`sso_only?` accounts are refused by every rule. `OauthSso.sign_in` sets
`session["auth_time"]`. Provider error bodies are logged, never shown.
After 2.39.4: Apple's and Microsoft's token exchange follows no redirect, and
`APPLE_BASE_URL` / `MICROSOFT_BASE_URL` may only name this machine
(`http://127.0.0.1:<port>`, `localhost`, `[::1]` — a test stand-in; anything
else raises), since their id_token is read unsigned.

`soli generate saml` (since 2.36.3; needs `generate auth`) writes
`app/services/saml_sso.sl` (`SamlSso.start(return_to)` → redirect URL,
`finish(saml_response)` → `{assertion, return_to}`, `user_for(assertion)`),
`saml_controller.sl` (`GET /saml/login`, `POST /saml/acs`, `GET
/saml/metadata`), `SamlIdentity`, `SamlAssertion` (after 2.38.0: each
assertion signs in once, `SamlSso.spend`), migrations, and
`skip_csrf("/saml/acs")`.
Config: `APP_BASE_URL`, `SOLI_SESSION_SECRET`, and `SAML_IDP_METADATA_FILE` or
`SAML_IDP_ENTITY_ID` + `SAML_IDP_SSO_URL` + `SAML_IDP_CERTIFICATE`;
`SAML_PROVISION` (after 2.39.3: unset follows `AUTH_SIGNUP` — `closed` means
no just-in-time accounts; `on` forces them). Accounts SAML creates are
`sso_only` (`user.sso_only?`): no password sign-in or reset, no magic link,
no API token (create 403, use 401), no passkey, no OAuth link; wrong passwords
are not counted against it (no lockout). After 2.39.4 a SAML session ends
after `SAML_SESSION_TTL` (default 28800 s) or the IdP's `SessionNotOnOrAfter`,
whichever is sooner (session key `sign_in_expires_at`, checked by the auth
scaffold's `load_current_user`); ending it at once takes SCIM deactivation or
`revoke_credentials`. `SAML_PROVISION` reads only
`on`/`off` (anything else = off); in production `first_user` mode SAML waits
until the admin signed up with `AUTH_SETUP_TOKEN`. Linking an existing account
by email is **opt-in** after 2.39.3 (`SAML_LINK_EXISTING=on`; before, on by
default) — never an admin's, never one already linked at that issuer, and only
an ASCII address taken from an email attribute (in `SAML_EMAIL_ATTRIBUTES`
order) or an `emailAddress`-format NameID (lowercased). With linking off, an
unconfirmed account is still taken over (when provisioning is on — and, after
2.40.0, only when `auth_require_confirmed_email` is true: otherwise the
sign-in is refused, since an unconfirmed account may be in use), and a SCIM
account (`scim_provisioned_at` / `scim_external_id`) is linked (and made
`sso_only` if it has no password) — after 2.39.4 only when its
`scim_external_id` equals the NameID, or with `SAML_LINK_DIRECTORY=on`. The SP metadata asks for a persistent
NameID; an `emailAddress` NameID inherits the IdP's address reuse.
`SAML_IDP_ENTITY_ID` must be set (a blank one raises). Request cookie:
`__Host-saml_request` behind `https://` (https required except on localhost).
New apps: unique `(user_id, issuer)` on identities. One IdP per app; per
tenant, scope users per tenant and keep linking off.

`soli generate auth --two-factor` (since 2.36.3; combines with `--passkeys`)
adds an authenticator-app second factor and **edits no existing file**:
`app/models/two_factor.sl` (`TwoFactor`: `for_user`, `enabled?`, `start`,
`confirm(code)` → backup codes or nil, `verify(code)`, `new_backup_codes`),
`app/middleware/two_factor.sl` (`# order: 6`: a session that passed one factor
only runs as a guest — `current_user()` nil — and its GETs redirect to
`/two_factor/challenge`; the session key `two_factor` holds `ok:<user key>` or
`off:<user key>`), `two_factor_controller.sl` (`/two_factor`, `/setup`,
`/confirm`, `/backup_codes`, `DELETE /two_factor`, `/challenge`), four views,
a migration and routes. Needs `SOLI_ENCRYPTION_KEY` (the secret is under
`encrypts`); `TWO_FACTOR_ISSUER` names the app. To sign a user in from your
own code past the gate, set `session_set("two_factor", "ok:#{user._key}")`.

`soli generate auth` sign-up policy is `AUTH_SIGNUP`: `first_user` (default —
the first account is created as admin, then `/signup` answers 403), `open`,
or `closed` (any other value = closed). After 2.39.0, with
`APP_ENV=production` the first sign-up needs `AUTH_SETUP_TOKEN` (16+ chars,
form field `setup_token`) and OAuth refuses to create that first account; an
unconfirmed account answers a sign-in like a wrong password when confirmation
is required, failed attempts are counted with an atomic `increment`, and
`generate saml` strips the password and sessions of an unconfirmed account it
links (as OAuth does). After 2.39.3 both call `User#revoke_credentials`
(no args): deletes the user's rows of `OauthIdentity`, `SamlIdentity`,
`PasskeyCredential`, `ApiToken`, `TwoFactor`, `MagicLink`, `Device` (by
`user_id`) and the OIDC provider's `OauthConsent`, `OauthAuthorizationCode`,
`OauthDeviceCode`, `OauthRefreshToken` (by `user_key`) for each model the app
defines (`const_get(name)` is nil otherwise), then clears password, reset and
remember tokens and bumps `session_version`; raises on failure. Every
scaffold sign-in sets `session["auth_time"]`; impersonation deletes it. The per-IP throttle shared by
login/sign-up/reset reads `AUTH_ATTEMPTS_PER_IP` (15) per
`AUTH_IP_WINDOW_SECONDS` (300). The generated files pass `soli fmt --check`
and `soli lint`.

`soli generate migration x` → "Unknown generate subcommand: migration (try:
scaffold, auth, oidc_provider, saml, audit, oauth, mailer, component, devices,
client, app_links, offline)" (2.36.3). There are no standalone model/controller generators —
write those by hand. Bare `soli generate` prints the general help, not the list.

Scaffold output (2.28.2, unchanged on 2.36.3): `app/models/post_model.sl`,
`app/controllers/posts_controller.sl` (actions `index/show/new/edit/create/update/delete`,
explicit routes rather than `resources`), five views including `_form.html.slv`,
`db/migrations/<ts>_create_posts.sl`, `tests/controllers/posts_controller_spec.sl`.
The resource gets explicit `get`/`post`/`put`/`patch`/`delete` routes in
`config/routes.sl`. Up to 2.34.0 the spec was a placeholder that failed (it assumed id 1 and posted
`name`) and the model registered `before_save("normalize_fields")` without
defining it; since then the spec creates its records from a sample value per
field and passes on a fresh app, and the model only shows a commented callback.

## Database

```bash
soli db:migrate up | down | status | redo | reset
soli db:migrate down --step 3
soli db:seed
soli db:create | db:drop        # SQL adapters
soli routes [-g posts] [--json] # route table with helpers and middleware
```

## Quality

```bash
soli fmt [paths]            # in place; --check for CI; only .sl files
soli lint [paths]           # .sl and .html.slv; --frames adds code frames, --format json for tools
soli lsp                    # language server on stdio (.sl and .slv; project-wide definition/hover,
                            # in a view: warns on an @x no rendering action/hook/parent/helper sets,
                            # and `@` completes the ones that are (view/undefined-ivar)
soli dap                    # debug adapter (DAP) on stdio: launch {program, cwd?, stopOnEntry?};
                            # breakpoints, step over/in/out, pause, stack, Locals/Globals, evaluate
                            # (can assign); debug() stops there under it. setBreakpoints takes condition,
                            # hitCondition (N / >= N / == N / > N / % N) and logMessage ({expr}).
                            # VS Code extension declares the "soli" debug type. mode "script" (default)
                            # or "test" (program = spec file/dir, soli test with 1 worker, optional
                            # filter; stops in spec + in-process code, not in the test server).
                            # engine "tree" (default) | "vm" (same stops; evaluate read-only;
                            # top-level vars are <main> locals). Output → DAP output events.
soli serve . --dev --inspect[=127.0.0.1:9229]   # DAP over TCP (loopback, --dev only): attach
                            # {"token": dev token = SOLI_DEV_REPL_SECRET or printed}; breakpoints stop
                            # live requests (one at a time); disconnect resumes and clears them.
                            #  member completion after . and @ — #691, since 2.34.0;
                            #  references/rename across app/, config/, tests/ and views' <% %> code,
                            #  hover shows the # comment above a def — since 2.36.0; renaming a member
                            #  two classes declare is refused;
                            #  serverInfo {name: "soli", version} in initialize — #692)
soli test [paths] [--no-coverage] [--filter TEXT] [--coverage-min N] [--jobs N] [--watch] [--profile]
soli update docs            # refresh the app's CLAUDE.md files and docs/ from this binary
```

Tree-sitter (#692, since 2.36.0): `editors/tree-sitter-soli/` in the repo —
grammar + queries for Neovim/Helix/Zed (setups in its README), `.slv` via
`embedded_template` + `template-queries/`; not in any editor registry yet.

Editors (#692, since 2.34.0): the VS Code / Cursor extension
`solilang.soli-language` 0.3 (in `editors/vscode/`, not yet on the
Marketplace/Open VSX — install the CI-built `.vsix`) starts `soli lsp`.
Settings `soli.lsp.enabled`, `soli.lsp.path`, `soli.lsp.trace` (same names as
Nova; `soli.lsp.path` is user-settings only, a workspace's
`.vscode/settings.json` cannot set it); without a path it tries `PATH`, then `~/.local/bin`, `~/.cargo/bin`,
`/usr/local/bin`. `soli lint` on save runs only while the server is not
running. `.slv` files are the `soli-template` language (TextMate scope
`text.html.soli`); the canonical grammars are `editors/grammars/*.tmLanguage.json`
(`scripts/sync-grammars.sh` copies them).

What `soli fmt` changed in testing: `if x == nil … return 1 … end` became
`return 1 if x.nil?`, `//` comments became `#`, a blank line was inserted after
guard clauses, long hash literals were broken one key per line.

Lint rules seen firing: `idiom/nil-comparison`, `idiom/prefer-blank`,
`idiom/prefer-to-s`, `idiom/prefer-includes`, `smell/undefined-local`,
`naming/snake-case`, `style/line-length` (120), `idiom/redundant-template-escape`
(in .slv). Others documented: `style/redundant-model-import`,
`smell/duplicate-global`, `smell/deep-nesting`, `smell/unreachable-code`,
`smell/empty-catch`, `idiom/manual-find-guard`,
`security/unfiltered-mass-assignment`, `smell/closure-cycle`, `docs/openapi`,
`smell/shadowed-builtin` (a local named after a builtin — `options`, `stream`,
`page_url` — in a scope that also calls it; rename it, or `let x = …` to
shadow on purpose). `smell/duplicate-methods` does not count `def x` and
`static def x` as duplicates.

## Code graph (for agents; needs SoliDB)

Verified on soli 2.36.0 (2026-11-13) against a throwaway SoliDB.

```bash
soli graph build --no-embed            # structural graph in SoliDB (embeds by default with SOLI_EMBEDDING_API_KEY)
soli graph query "publish post"        # code + neighbours; --json for agents
soli graph note "Publishing goes through Post#publish, which saves the record" \
  --on PostsController#publish,Post#publish --question "how is a post published?"
soli graph notes                       # every note; "(stale …)" when its code changed
soli graph forget <key>
```

- A query lists, per result, `reads`/`writes` edges to `collection:<name>`
  (with the operation: `reads find`, `writes save`, `reads FOR`), `renders
  implicit` for an action's own view and `assigns @x` for what it hands it.
- `--on` takes the `qualified_name`s a query prints; an unknown one is
  refused. Later queries print matching notes under **Notes** and earlier
  questions that share words under **Asked before**.
- Keyword mode (no embedding key) matches substrings of names: "publish"
  finds `Post#publish`, "publishing" does not.
- `soli agent` exposes the same as the `graph` and `graph_note` tools.
