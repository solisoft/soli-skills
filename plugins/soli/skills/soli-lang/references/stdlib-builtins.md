# Builtins and core methods (verified on soli 2.28.2; rechecked on 2.36.3)

Rule of thumb: methods listed here run without `()` when they take no
arguments. If a bare call prints `<method array.tally>` or similar, the
method exists but was not auto-called — add `()`. A method you made up
raises `Cannot access property 'upper' on String (did you mean …)` on both
engines (since 2.34.0; up to 2.33 a bare read on an untyped value gave a
silent `<method string.upper>`). The full list lives in the app's `docs/builtins.md`.

## Output and introspection

```soli
print("x")            # also println, puts
type(5)               # "int"   (float, string, null, array, hash)
5.class               # "int"
defined("url")        # true — is this global name taken?
clock                 # float seconds; works bare: elapsed = clock - start
```

## String methods

```soli
s = "  Hello, World!  "
s.trim / s.strip      # "Hello, World!"   (also lstrip, rstrip)
s.upcase / s.downcase # also uppercase / lowercase
s.length              # bytes! "é".length == 2   (also len, size) — s[i], substring, truncate count characters
"hello world".capitalize   # "Hello world"
"abc".reverse         # "cba"
t = "Hello, World!"
t.contains?("Hello")  # also includes?, include?, contains
t.starts_with?("Hell") / t.ends_with?("!")   # also without "?"
t.split(", ")         # ["Hello", "World!"]
t.replace("World", "Soli")   # replaces all
"Hello".gsub("l", "L")       # "HeLLo"
"Hello".sub("l", "L")        # "HeLlo"  (first only)
"Hello".index_of("l")        # 2  — no start-offset argument
"Hello".substring(0, 3)      # "Hel"   (s[0] works; s[1..3] and s.slice do NOT)
"Hello".substring(2)         # "llo"   (to the end; substring(-3) = last 3)
"héllo wörld".truncate(8)    # "héllo..." — counts characters, suffix (default "...") included; 2nd arg = suffix
"hello".chars                # ["h","e","l","l","o"]
for ch in "héllo" … end      # one-char Strings, both engines; `for ch, i in s` adds the index
"l1\nl2".lines               # ["l1", "l2"]
"x" * 3                      # "xxx"
"42".lpad(5, "0")            # "00042" — PADDING EXISTS: lpad/rpad/center
"ab".rpad(5, ".")            # "ab..."  (rjust/ljust are aliases; pad defaults to " ")
"Café & Restaurant".slugify  # "cafe-restaurant"
"hello_world".camelize       # "helloWorld"; camelize(true) → "HelloWorld"
"prefix-x".delete_prefix("prefix-")   # also delete_suffix
"abc123".match("\\d+")       # ["123"]
"a1b2".scan("\\d")           # ["1", "2"]
"ff".to_i(16)                # 255; "42".to_i, "nope".to_i == 0
"3.5".to_f                   # 3.5
"{\"q\": 3}".parse_json      # hash
"abc".to_sym                 # :abc
"".blank? / "  ".blank? / nil.blank?  # true — .present? is the inverse
nil.to_s                     # ""
42.to_s                      # "42"
```

Not available (type error or no such method): `upper`, `lower`,
`snake_case`, `titleize`, `pad_start`/`padStart` (use `lpad`/`rpad` — the
error says so), `=~`, `"%s" % [...]`, `.format`.

## Array methods

```soli
xs = [3, 1, 2]
xs.length / xs.first / xs.last / xs.sum / xs.min / xs.max
xs.first(2) / xs.last(2)   # arrays (bare first/last = the element)
xs.sort[0]            # 1
xs.sort.first         # 1
xs.reverse / xs.uniq / xs.flatten / xs.compact / xs.shuffle
xs.include?(2) / xs.includes?(2) / xs.index_of(2) / xs.empty?
xs.push(4)            # mutates; also xs << 5
xs.pop                # returns the last element
xs.take(2) / xs.drop(2)
[1, 2, 3, 4].slice(1, 2)   # [2]  — (start, end-exclusive)
xs.find { |x| x > 2 }
xs.any? { |x| x > 3 }      # any?/all?/none? need a block
xs.count { |x| x > 1 }
xs.select { |x| x > 1 } / xs.reject { |x| x > 1 }   # select == filter
xs.sort_by { |x| -x }
xs.sort_by { |x| [x.page, x.y] }   # tuple = composite key (unreleased; through 2.38.1 arrays compared equal and the order was kept)
xs.each_with_index do |x, i|
  print("#{i}:#{x}")
end
[1, 2].zip([3, 4])         # [[1, 3], [2, 4]]
["a", "b", "a"].tally()    # {a => 2, b => 1} — needs ()
[].sum                     # 0
[1, 2, 3].reduce(fn(acc, x) acc + x, 0)   # 6
```

Field-name helpers on arrays of hashes/records take a **string**, not a block
(a block raises "takes a field name as a string"):

```soli
rows = [{"k": "a", "n": 1}, {"k": "b", "n": 2}, {"k": "a", "n": 3}]
rows.group_by("k")        # {a => [...], b => [...]}
rows.sum_by("n")          # 6
rows.max_by("n")          # {k => a, n => 3}   (min_by too)
rows.index_by("k")
rows.filter_by("k", "a")
rows.pluck("n")
rows.map(&:n).sum         # &:name reads a key, a field, or calls a method
```

Missing: `each_slice`, `reduce(init) { }`, `max_by { }`, array `-`.

## Hash methods

```soli
h = {"a": 1, "b": 2}
h.keys / h.values / h.length / h.empty?
h.has_key("a")              # true — NOT has_key?, key?, include?
h.fetch("z", 0)             # 0
h["z"]                      # nil
h.dig("a")                  # nested: {"a": {"b": 1}}.dig("a", "b")
h.merge({"c": 3})
h.delete("a")               # returns the value, mutates
h.slice(["a"]) / h.except(["a"])
h.transform_values { |v| v * 10 }
h.filter { |k, v| v > 1 }   # returns a hash
h.map { |k, v| [k, v * 2] } # block must return [key, value]; result is a HASH
h.values.map { |v| v * 2 }  # use this for an array result
h.each do |key, value|
  print("#{key}=#{value}")
end
h.to_json                   # "{\"a\":1,\"b\":2}"
```

`h.map { |k, v| v * 2 }` (block not returning a pair) silently returns `{}`.

## JSON

```soli
data = json_parse("{\"a\": [1, 2]}")    # or JSON.parse(...)
json_stringify({"x": 1})                # or JSON.stringify(...), or {"x": 1}.to_json
```

In views, `json_script(value)` emits JSON safe inside `<script>` (escapes `</`).

## XML (`Xml`, `XmlNode`, `XmlDocument`, `XmlReader`)

Since 2.30.0; verified, both engines and `soli check`.

```soli
doc = Xml.parse("<rss><channel><title>A &amp; B</title><item id=\"1\"><title>One</title></item><item id=\"2\"/></channel></rss>")
doc.root.name                          # "rss"
doc.find("channel/title").text         # "A & B"  (find on a document starts at the ROOT element)
doc.find_all("channel/item/@id")       # ["1", "2"] (@attr and text() give Strings)
doc.find("channel/item")["id"]         # "1"  (attributes by index; nil when absent)
doc.xpath("/rss/channel/item[2]")      # strict XPath 1.0: relative paths start at the document node
doc.at_xpath("//item[title='One']/@id")   # "1"; at_xpath/find give nil on a miss, xpath/find_all []
doc.root.children                      # element children only; `nodes` has text/comments too
doc.to_h   # {"rss": {"channel": {"title": "A & B", "item": [{"@id": "1", "title": "One"}, {"@id": "2"}]}}}

feed = Xml.build("rss", version: "2.0") do |rss|      # returns the document
  rss.add("channel") do |channel|                     # add(name, content, attrs) returns the child
    channel.add("item", "Tom & Jerry", id: 7)         # named args = attributes; escaping is automatic
  end
end
feed.to_xml        # "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<rss version=\"2.0\"><channel><item id=\"7\">Tom &amp; Jerry</item></channel></rss>"
feed.to_xml(indent: 2)

Xml.each("big.xml", "product[@status='active']") do |product|   # streams a FILE, returns the count
  print(product.find("name").text)                              # each match is a detached node
end
reader = Xml.reader("big.xml", "product")    # pull form
while (product = reader.next)
  break if product["sku"] == "b2"
end
```

- `find`/`find_all` are lenient: unprefixed names match in **any** namespace
  (Atom's default namespace just works). `xpath`/`at_xpath` match unprefixed
  names in **no** namespace: write `//xmlns:entry`, `//*:entry`, or pass a
  table `xpath("//a:entry", {"a": uri})` (named args work too: `a: uri`).
  Other prefixes fall back to the root element's declarations.
- `doc.find("rss")` is `nil` (relative to the root); `doc.xpath("rss")` finds it.
- `break` inside an `Xml.each` block stops the walk, which then answers nil.
- `Xml.parse` takes a String only; files: `Xml.parse_file(path)`, `Xml.each`,
  `Xml.reader` (jailed to the app root under serve/jobs).
- A `<!DOCTYPE …>` raises by default. For APIs that always send one (Qualys
  v2, PingCastle, ORCQ), pass `doctype: "skip"` instead of stripping it by
  hand: `Xml.parse(body, doctype: "skip")` (also `parse_file`, and
  `Xml.each(path, "HOST", doctype: "skip") do … end` / `Xml.reader`). The
  DOCTYPE is read past, never interpreted: an entity its internal subset
  declares still raises `unknown entity`. `doc.doctype` is its name or `nil`.
  `SOAP.*` and c14n always refuse one.
- `node.is_a?("XmlNode")` takes a String; `XmlNode.new` raises (use
  `Xml.parse`, `Xml.element(name, content, attrs)` or `node.add`).
- Mutation: `node["a"] = v` (`nil` removes), `node.text = s`, `node.name = s`,
  `append`/`prepend`/`insert_before`/`insert_after`/`replace`/`remove`/`dup`.
  Any other `node.foo = 1` raises (and `soli check` refuses it).
- `==` is identity (same node of the same tree), not structure.
- JSON of a node is its XML string; use `to_h` for structured JSON.
- A **literal** XPath is compiled by `soli check` (and before `soli file.sl`
  runs): `doc.xpath("sum(//a)")` stops the script before it starts. Outside
  the subset: axes `child::`, functions other than `contains`/`starts-with`/
  `not`/`position()`/`last()`, numeric comparison, paths inside predicates,
  `(//a)[1]` (use `at_xpath("//a")`).
- Don't name the result `lines`: `lines = doc.xpath(...)` is a type error
  (it is a builtin function). `line_items` is fine.
- `SOAP.parse` returns a Hash (attributes dropped, `_text` for mixed text,
  `nil` for `<a/>`); for attributes or XPath, `Xml.parse(result["body"])`.

## Spreadsheet (csv, xlsx)

```soli
Spreadsheet.csv("name,age\nAda,36")        # [{"name": "Ada", "age": "36"}] — csv cells are Strings
Spreadsheet.csv_file(path)  Spreadsheet.to_csv(rows)  Spreadsheet.csv_write(rows, path)
                                           # to_csv/csv_write quote cells with , " or a newline and
                                           # prefix a formula cell (= + - @) with ' ; numbers untouched
Spreadsheet.excel_write(rows, "out.xlsx")              # one sheet "Sheet1", columns = first row's keys SORTED
Spreadsheet.excel_write(rows, "out.xlsx", ["N°", "Total"])   # header order of your choice
Spreadsheet.excel("out.xlsx")                          # first sheet -> array of hashes (row 1 = keys)
# Several sheets (after 2.39.4): a hash name => rows, or name => {"rows", "columns"}
Spreadsheet.excel_write({"Clients": clients, "Ventes": {"rows": sales, "columns": ["total", "n"]}}, "book.xlsx")
Spreadsheet.excel("book.xlsx", "Ventes")               # a sheet by name; unknown name raises
Spreadsheet.excel_sheets("book.xlsx")                  # {"Clients": [...], "Ventes": [...]}, tab order
```

- Int/Float are written as numbers and read back as **Float** (`2` -> `2.0`);
  everything else is text, so `"007"` keeps its zeros.
- Sheet names Excel refuses raise: empty, > 31 chars, `: \ / ? * [ ]`, a
  leading/trailing `'`, `History`, duplicates ignoring case. With a hash of
  sheets there is no third argument — put `columns` inside the sheet.

## DateTime and Duration

```soli
dt = DateTime.parse("2024-01-15T10:30:00Z")   # also "2024-01-15"
dt.year / dt.month / dt.day / dt.hour
dt.format("%Y-%m-%d %H:%M")    # "2024-01-15 10:30"
dt.to_iso                      # "2024-01-15T10:30:00+00:00"
DateTime.parse("2024-01-15T10:30:00+02:00").hour   # 10 — keeps its offset
dt.local.hour / dt.utc.hour    # re-read the same instant in server zone / UTC
dt.to_unix                     # 1705314600
dt.add_days(1)                 # also add_hours, add_minutes (negative to subtract)
DateTime.from_unix(1704067200)
DateTime.utc / DateTime.now
d2 > d1                        # comparison works; d2 - d1 does NOT
Duration.hours(2).total_seconds    # 7200 (also seconds/minutes/days)
time_ago(DateTime.utc.to_unix - 120)   # "2 minutes ago"
```

A DateTime reads in its own zone — `format`, `hour`, `to_iso` all agree:
UTC for `parse` without an offset, `from_unix`, `epoch`, `DateTime.utc`; the
written offset for `parse("…+02:00")`; the server's `$TZ` only for
`DateTime.now` and `.local` (whose `to_iso` then prints the local offset).
JSON / SQL always store the UTC instant. (Up to 2.32.0 `to_iso` said UTC while
`format`/`hour` read the server zone.)

`dt.format(pattern, "fr")` (locale) works at runtime but the static checker
rejects the 2-arg form on a value it knows is a DateTime (`soli check`/scripts);
fine in a function with an untyped parameter or in served code.

## Regex

```soli
Regex.matches("^a.c$", "abc")                  # true
Regex.find("\\d+", "ab 42 cd")                 # {match => 42, start => 3, end => 5}
Regex.find_all("\\d+", "1 and 22")             # array of those hashes
Regex.replace_all("\\s+", "a   b  c", " ")     # "a b c"
Regex.split(",\\s*", "a, b,c")                 # ["a", "b", "c"]
Regex.escape("a.b")                            # "a\\.b"
```

Patterns are strings: double the backslashes. `\\w` in some patterns hits a
compiled-size limit (Unicode classes) — prefer `[a-z0-9_]`.

## Crypto, passwords, tokens, encoding

```soli
Crypto.sha256("hello")          # hex
Crypto.sha256_base64(pdf_b64)   # hex SHA-256 of the decoded BYTES (= sha256sum of the file); Crypto.sha256 hashes text (unreleased)
Crypto.random_hex(16)           # 32 hex chars (CSPRNG)
Crypto.random_token(16)
uuid_v4()                       # 36-char UUID
h = password_hash("secret")     # Argon2id
password_verify("secret", h)    # true
Base64.encode("hello") / Base64.decode("aGVsbG8=")
token = JWT.sign({"user_id": 7, "sub": "u7"}, secret, {"expires_in": 3600})
# JWT.verify and JWT.decode_unsafe are the other two methods. jwt_sign,
# jwt_verify and jwt_decode_unsafe still run and each warns once per process
# (deprecated). jwt_decode stays removed.
# secret must be >= 32 bytes (raises otherwise); without an expiry, verify
# answers {"error": true, "message": "Missing required claim: exp"}
# Time claims come from the OPTIONS: {"exp": ts} or {"expires_in": s},
# {"iat": ts} (default now), {"nbf": ts}. An "exp"/"iat" key in the payload
# raises ("JWT.sign() takes `iat` from the options, not the payload";
# the jwt_sign alias still says jwt_sign() in that message).
# asymmetric: {"algorithm": "RS256" | "ES256" | "EdDSA", "key": pem} with secret ""
# (ES256 is NOT in 2.36.3 — unreleased on main: a P-256 key, PKCS#8 PEM to sign — Apple's .p8 —, public PEM to verify)
JWT.verify(token, secret)       # payload hash, or {"error": true, "message": ...}
                                # (a non-string "sub" is dropped from the payload)
JWT.verify(token, "", {"algorithm": "RS256", "key": pem, "audience": "client-id",
  "issuer": "https://op.example", "subject": "42", "authorized_party": "client-id", "leeway": 60})
# After 2.39.3 an unknown or mistyped option RAISES — `aud`/`iss` are JWT.sign's
# names, JWT.verify's are `audience`/`issuer` (before, a wrong key skipped the
# check silently). A list of audiences is any-of. `authorized_party` checks
# `azp` (and a multi-valued `aud` then needs one). An `iat` beyond the leeway
# fails "InvalidIssuedAt"; a bare base64 public key as an HMAC secret raises
# (a heuristic: URL-safe base64, hex DER or a JWK get through). `leeway` is
# 0..31536000 (raises above). `authorized_party` does not replace `audience`:
# pass both.
Markdown.to_html("# Hi")        # "<h1>Hi</h1>"
# TOTP (since 2.36.3 for totp_secret / totp_step / qr_svg)
secret = Crypto.totp_secret                 # 32 Base32 chars
code = Crypto.totp_generate(secret)         # "287082"; (secret, unix_time?, period?)
Crypto.totp_verify(secret, code)            # Bool, ±1 step; RAISES unless code is 6 digits
Crypto.totp_step(secret, code)              # the matching time step (Int) or nil — never raises on a
                                            # bad code; store the last accepted step, refuse <= it (replay)
Crypto.totp_uri(secret, "ada@example.com", "MyApp")   # otpauth://…
qr_svg("otpauth://…", 220)                  # inline <svg>, server-rendered; <%- %> in a view; raises if too long
Toml.parse("a = 1") / Yaml.parse("a: 1")
I18n.format_currency(12.5, "EUR", "fr")   # "12,50 €"
```

### Certificates (`X509`)

```soli
X509.info(pem)                 # subject, issuer, not_after, days_left, dns_names…
X509.fingerprint(pem)          # hex SHA-256 of the DER; X509.spki_pin(pem) for TLS pinning
# Issue from a CA the app holds (unreleased on main): EC P-256 only, issuer key in PKCS#8
# (`openssl pkcs8 -topk8 -nocrypt -in ca.key`; a SEC1 "EC PRIVATE KEY" is refused).
id = X509.issue({"issuer_cert": ca_pem, "issuer_key": ca_key_pem,
  "subject": {"common_name": "Adèle Martin", "email": "a@x.fr", "country": "FR"},
  "days": 1, "policies": ["1.3.6.1.4.1.99999.1"]})    # ca: true → intermediate CA
# id: cert, key (PKCS#8 PEM), serial, not_after_unix — feed cert/key to PDF.sign
# with "chain": [ca_pem, root_pem]; one key per signer = eIDAS advanced signature.
```

### SAML sign-in (`Saml`, since 2.36.3)

Service-provider side, stateless; every failed check **raises**.

```soli
request = Saml.auth_request({"idp_sso_url": sso, "sp_entity_id": entity_id, "acs_url": acs})
# request: id ("_" + 40 hex — keep it), url (redirect there), xml
assertion = Saml.verify_response(params["SAMLResponse"], {"certificates": [pem],
  "idp_entity_id": idp, "audience": entity_id, "acs_url": acs, "request_id": request["id"]})
# after 2.39.3 idp_entity_id is required (an empty one raises); skip the Issuer
# check only by name: "any_issuer": true
# assertion: name_id, name_id_format, attributes ({name => [values]}, always arrays), issuer,
#   session_index, session_expires_at, authn_context, assertion_id, response_id,
#   in_response_to, expires_at, signed ("assertion" | "response" | "both")
# after 2.40.0 also: name_qualifier, sp_name_qualifier (nil when absent),
#   certificate_expires_at (unix), certificate_expired (Bool); expected option
#   "require_valid_certificate": true refuses an expired/not-yet-valid IdP cert
# after 2.39.3: response_id is nil when signed == "assertion" (the wrapper is
# unsigned); expires_at = earlier of Conditions and bearer NotOnOrAfter
Saml.metadata({"sp_entity_id": entity_id, "acs_url": acs})   # SP metadata XML (String)
Saml.parse_metadata(xml)                                     # entity_id, sso_url, certificates (PEM)
Saml.test_response({"private_key": pem, "issuer": idp, "acs_url": acs, "audience": entity_id,
  "name_id": "ada@example.com", "request_id": id, "attributes": {"groups": ["admin"]}})   # base64, for specs
```

`request_id` is required unless `allow_unsolicited: true`. Refused by name:
encrypted assertions, ECDSA/PSS, SHA-384, SHA-1 (unless `allow_sha1: true`),
an RSA certificate under 2048 bits, (after 2.38.0) a prefix bound to
`""` or used undeclared, and (after 2.39.3) a document over 10,000 elements,
64 distinct namespace prefixes or 30,000 namespace declarations, an
InclusiveNamespaces PrefixList over 64 prefixes, and a `SignedInfo` other than
exactly CanonicalizationMethod, SignatureMethod and one Reference. The pinned
certificate's validity dates are not checked. `verify_response` keeps no state: refusing a second
use of `assertion_id` is the app's job (`SamlSso.spend` in the scaffold).
The ACS route is a cross-site POST: `skip_csrf("/saml/acs")`, and keep the
request id in an encrypted `SameSite=None; Secure` cookie, not the session
(a Lax session cookie does not come with that POST; the scaffold names it
`__Host-saml_request`, `Path=/`, behind `https://`). `soli generate saml`
scaffolds all of it (cli.md).

### Passkeys (`WebAuthn`, since 2.34.0)

Stateless relying-party half of WebAuthn. Options calls return
`{challenge, options}`; keep `challenge` in the session (single use, delete it
before verifying) and `render_json(result["options"])`. Verify calls take the
browser's `PublicKeyCredential.toJSON()` (`req["json"]`) and **raise** on any
failed check — `rescue nil` for the fail-closed one-liner.

```soli
result = WebAuthn.registration_options({"rp_id": "example.com", "user_id": user._key, "user_name": user.email})
result["options"]["user"]["id"]          # base64url of user_id; attestation is always "none"
result["options"]["pubKeyCredParams"]    # algs [-7, -8, -257] (ES256, EdDSA/Ed25519, RS256)
login = WebAuthn.authentication_options({"rp_id": "example.com"})   # allowCredentials [] → discoverable
record = WebAuthn.verify_registration(req["json"], {"challenge": challenge, "origins": "https://example.com", "rp_id": "example.com"})
# record: credential_id, public_key (COSE b64url), algorithm, sign_count, transports, aaguid,
#         user_verified, backup_eligible, backed_up, attestation_format, attestation_type ("none" | "self" = packed self-attestation, checked | "unverified" = chain/other format, never trusted)
check = WebAuthn.verify_authentication(req["json"], {"challenge": challenge, "origins": [origin],
  "rp_id": "example.com", "public_key": passkey.public_key, "sign_count": passkey.sign_count})
# check: credential_id, user_handle (the user_id string, nil if the authenticator sent none —
#   for a username-less sign-in require check["user_handle"] == the passkey owner's id), sign_count, counter_regressed, flags
# errors read "WebAuthn.verify_authentication(): expected public_key is missing",
# "...: origin mismatch (expected X, got Y)", "...: challenge mismatch", "...: signature is invalid"
```

`origins` is an exact match (scheme, host, port; String or Array).
`require_user_verification: true` enforces UV. `soli generate auth --passkeys`
scaffolds all of it (see cli.md). Native apps: `AppLinks.android(pkg, [fp],
{"passkeys": true})` adds `get_login_creds`, `AppLinks.apple(app_id, paths?,
{"passkeys": true})` a `webcredentials` section; an Android app's ceremony
reports `AppLinks.android_origin(fp)` (`android:apk-key-hash:…`) as origin —
add it to `origins` / `WEBAUTHN_ORIGINS`.

## Files and environment

```soli
barf("out.txt", "content")     # write
slurp("out.txt")               # read
File.read(path) / File.write(path, s) / File.append(path, s) / File.exists(path)
getenv("HOME")                 # nil when unset
hasenv("HOME")
```

Under `soli serve`, file paths are jailed to the app directory; scripts and
the test runner are not jailed. There is no `setenv`.

## Path, Bytes, Zip (unreleased on main)

`Path` is a lexical `/` string. It does not touch the disk and it does not
normalize `.` or `..`.

```soli
Path.join("a", "b", "c.txt")     # "a/b/c.txt"
Path.join("/var", "app")         # "/var/app"
Path.join("a//b")                # "a/b"
Path.dirname("/var/app/x")       # "/var/app"   (no slash → "."; "/" stays "/")
Path.basename("a/b.txt")         # "b.txt"      ("/" stays "/")
Path.extname("archive.tar.gz")   # ".gz"        (".gitignore" → "")
```

`Bytes` is one binary value. `from_string` is UTF-8, `from_array` is `0..255`,
`from_base64` is standard base64. `slice(start, end)` is exclusive. `to_string`
raises when the bytes are not UTF-8. It does not change what `File`, `Image`,
`HTTP` or `PDF` return.

```soli
hello = Bytes.from_string("hello")
hello.length                         # 5
hello.slice(1, 4).to_string          # "ell"
hello.to_base64                      # "aGVsbG8="
```

`Zip` holds an archive in memory (the `office` feature). A name given to
`Zip.create` must be relative, with no empty, `.` or `..` segment; `Zip.read`
takes any name `Zip.entries` lists. A read inflates at most 32 MiB of one
entry and raises past it: `Zip.read(a, name, {"max_size": n})` to raise it.

```soli
archive = Zip.create({"hello.txt": "hi", "bin.dat": {"base64": "YWJj"}})
Zip.read_text(archive, "hello.txt")   # "hi"
Zip.read(archive, "bin.dat")          # base64
Zip.entries(archive)                   # [{name, size, compressed_size, directory}, …]
```

## HTTP client

```soli
body = HTTP.get(url)                       # String body; RAISES on non-2xx
data = HTTP.get_json(url)                  # parsed; raises on non-2xx
created = HTTP.post_json(url, {"a": 1})
response = HTTP.request("GET", url)        # never raises on status:
response["status"]                         #   {status, status_text, headers, body}
page = HTTP.get(url) rescue nil
HTTP.get(url, {"headers": {"Authorization": "Bearer x"}, "timeout": 5})
```

- A call's result is pending until used; indexing it (`HTTP.request(...)["status"]`)
  or comparing it (`HTTP.get(url) == "ok"`) waits for it, on both engines.
- `TCP.check("db.example.com", 5432, timeout: 2)` → `{"open" => true,
  "latency_ms" => 3.4, "error" => nil}`: connect-only port probe (no data
  sent); a closed port is `open: false` + reason, not an exception. Same SSRF
  policy as HTTP (raises on loopback/private unless allowlisted). No other
  socket API exists.
- SSRF guard: loopback/private hosts are refused. To reach a local sidecar,
  allow it explicitly, e.g. `SOLI_HTTP_ALLOW_HOSTS=127.0.0.1:9090` in the
  environment, and use the literal host from that list in the URL.
- Branch on status with `HTTP.request`; a `try` around `get_json` catches the
  whole error page as a string.
- Response header names are lowercase. `response["headers"]["set-cookie"]` is
  **always an Array** of strings, one per `Set-Cookie` line, even with a single
  cookie (up to 2.30.1 it was a String holding only the last cookie); any
  other repeated header is one String joined with `", "`. Same for
  `HTTP.parallel`, `stream.headers`, `SOAP.call`, spec responses and
  `res_header(r, "set-cookie")`. Read a cookie with
  `.map { |line| line.split(";")[0] }`.
- Redirects are followed (up to 10 hops, each SSRF-checked). To read the 3xx
  yourself, pass `"follow_redirects": false` (2.31.0+):
  `r = HTTP.request("GET", url, {"follow_redirects": false})` gives
  `r["status"] == 302` and `r["headers"]["location"]` (as sent: may be
  relative). Same key on the verb helpers (a 3xx then raises
  `HTTP 302 error: …`), `HTTP.parallel` items and `HTTP.stream`; it is never
  sent as a header; a non-Bool raises. Refused on the edge (wasm) build.
- A body is read as UTF-8 by default, and a non-UTF-8 one (PNG, PDF, ZIP,
  Office file, Latin-1 page) **raises** `invalid UTF-8 in response body`.
  Pick the shape with the `response` key (since 2.31.0), consumed
  like `timeout`, never sent as a header:
  `HTTP.get(url, {"response": "base64"})` → String, identical to
  `Base64.encode(slurp(path, "binary"))` (the light way to forward a file);
  `{"response": "binary"}` → `Array<Int 0-255>`, equal to
  `slurp(path, "binary")` (`r["body"]` with `HTTP.request`);
  `{"response": "latin1"}` (any charset label) → decoded UTF-8 String;
  `"text"` is the default. Taken by the verb helpers, `HTTP.request`'s headers
  hash, `get_all` and each `HTTP.parallel` item; `get_json` & co. refuse
  `"binary"`/`"base64"`; an unknown value raises. Still capped by
  `SOLI_HTTP_MAX_RESPONSE_BYTES` (default 50 MiB). An empty body is `[]` / `""`.
- In a spec, `mock_http_route(path, 200, bytes)` with an `Array<Int>` body
  serves raw bytes.
- With OpenTelemetry on (`SOLI_OTEL=1` / `OTEL_EXPORTER_OTLP_ENDPOINT`), every
  `HTTP.*` call (and `SOAP.call`) already sends `traceparent` (+ the inbound
  `tracestate`), naming its own CLIENT span (since 2.32.0). Don't
  forward `req["headers"]["traceparent"]` by hand: a header you set is kept
  as-is, so copying the inbound one makes the downstream span a sibling of
  yours instead of its child. `SOLI_OTEL_PROPAGATE=0` turns injection off.

### Streaming answers (SSE, AI models)

```soli
stream = HTTP.stream("POST", url, {"Authorization": "Bearer k", "timeout": 120}, body)
stream.status                              # Int, read when it opened; non-2xx does not raise
line = stream.next_line                    # String without its \n; nil at the end
while !line.nil?
  line = stream.next_line                  # raises after `timeout` s with no new bytes
end

# Bytes as they came (no line splitting, \r\n kept): Array<Int>, or base64
# with {"response": "base64"}; nil at the end. Since 2.31.0.
stream = HTTP.stream("GET", url, {"response": "binary"})
chunk = stream.next_chunk

# Without holding the thread: returns an id at once, the HTTP runtime reads it
id = HTTP.stream_to("POST", url, headers, body, {
  "format": "anthropic",                   # "anthropic" | "openai" | "sse" | "lines" (default)
  "topic": "chat:#{chat._key}",            # sse_subscribe topic: "delta" events, then "done"/"failed"/"cancelled"
  "job": "AnswerDoneJob",                  # enqueued once at the end, failures included
  "args": {"chat": chat._key}              # + stream_id, state, status, text, error, stop_reason, usage
})
HTTP.stream_status(id)                     # {id, state, text, status, error, stop_reason}; nil if unknown
HTTP.stream_cancel(id)                     # true if it was running here
```

- `timeout` is the wait for the **next bytes**, not the whole answer: a
  model may think for minutes, then stream.
- A job reading a long answer with `HTTP.stream` holds a job worker for all
  of it; use `stream_to` + `job` instead. `stream_to` raises at once on a bad
  URL, `format`, unknown key or a `job` that is not a class name; anything
  later ends the stream as `failed` (its job still runs).
- A `stream_to` stream lives in the process that started it: `stream_status`
  and `stream_cancel` see only this process's streams, and a restart loses
  the stream (not resumed, no job). A plain script waits for its streams
  before exiting.
- Both work in a plain script on the default engine (verified with a local
  server allowed through `SOLI_HTTP_ALLOW_HOSTS`).

### Realtime: WebSockets, `Channel`, SSE pub/sub

`Channel` is the room. `sse` and `stream` stay response helpers. `Model.broadcast`
is a different method (the collection name is the channel). The `ws_*`,
`broadcast` and `sse_broadcast` / `sse_subscribe` functions still run and each
warns once per process.

```soli
# config/routes.sl
router_websocket("/ws/room", "room#handle")

# app/controllers/room_controller.sl — a top-level def; always return a hash
def handle(event)                          # event: type, connection_id, message, headers, query
  if event["message"] == "join"
    Channel.join("lobby")                  # THIS connection; lands before the next line
    return {"send": "#{Channel.clients_in("lobby").length} here"}
  end
  return {"broadcast_room": event["message"]} if event["type"] == "message"

  {}
end

# From an action, job or callback (any engine):
Channel.broadcast("lobby", {"text": "hi"})       # WS channel AND SSE topic "lobby"; returns local SSE count
Channel.broadcast("lobby", "x", {"local": true}) # never published to the realtime bus
Post.broadcast({"event": "created"})             # channel = collection name ("posts")
Channel.broadcast_room("lobby", "x")             # WS only; Channel.broadcast_all(msg) = every socket
Channel.sse_broadcast("news", "data", "event-name") # SSE only
Channel.count                                    # Int; Channel.clients / Channel.clients_in(ch) -> Array<String> ids
realtime_bus()                                   # {backend, connected, published, received, dropped, last_rx_at}
```

- Handler return keys: `send`, `broadcast` (every socket), `join`, `leave`,
  `broadcast_room` (most recently joined room), `broadcast_channel`
  (`{"channel": "user:42", "message": payload}`, a named room), `close`, `track` / `untrack` /
  `set_presence` (Phoenix-style presence). Non-string payloads JSON-encode.
- `Channel.join` / `Channel.leave` (and `ws_join` / `ws_leave`) raise outside a
  WebSocket handler ("only works inside a WebSocket handler") — from an action,
  there is no "current" connection. Outside `soli serve` (script, spec,
  `soli worker`) the senders do nothing locally and the readers answer `[]` /
  `0` / `nil`.
- **One process by default.** Rooms, SSE subscribers and presence live in the
  process that accepted the socket. `SOLI_REALTIME_BUS=solidb` (env, on every
  instance and on `soli worker`) carries `Channel.broadcast`, `Model.broadcast`,
  `Channel.broadcast_all` / `broadcast_room`, `Channel.sse_broadcast` and the
  handler broadcast actions to every instance through the engine-owned
  `_rt_bus` SoliDB collection's changefeed;
  `SOLI_REALTIME_BUS=solikv` (or `redis` / `valkey`) does it with RESP
  `PUBLISH`/`SUBSCRIBE` on the SoliKV of `SOLIKV_RESP_HOST` /
  `SOLIKV_RESP_PORT` / `SOLIKV_TOKEN` — no database needed, so the choice for
  SQL apps and chatty rooms; apps sharing one SoliKV set distinct
  `SOLI_REALTIME_CHANNEL`s (default `soli:rt`). Publishing happens on a
  background thread. Fire and forget (a message can be lost). With a bus,
  presence (`track`, `ws_list_presence`, `ws_presence_count`,
  `ws_get_presence`, the `presence_diff`s) spans instances — eventually
  consistent (since 2.40.3 a node's older snapshot no longer undoes a later
  join or leave), a crashed instance's users leave after 30–40 s — and
  `Channel.send` / `Channel.close` (`ws_send` / `ws_close`) reach a connection
  held by another instance. `Channel.broadcast`'s return value,
  `Channel.clients` and `Channel.count` stay per process;
  `Channel.count({"cluster": true})` adds the others' (as of their last
  10 s heartbeat). `HTTP.stream_to` topics are not carried. Any other value is
  reported at boot and treated as `local`, as is `solidb` on a SQL app.
- A script that calls `Channel.broadcast` with the bus on must not exit at
  once: the publish runs on a background thread (`sleep(1)` is enough to see
  it land).

## Score engraver (`eui_score_*`, checked on soli 2.40.4)

- `eui_score_layout(src, width, {})` engraves LilyPond source with the
  engine of the `eui` feature (in the released binaries). It answers a hash:
  `layout`, `pages`, `systems`, `width`, `height`, `src`… on success, and
  `{"error": "score: a '{' that is never closed (at character 0)"}` when the
  source is refused — test `r["error"].nil?`, the call itself does not raise
  for bad LilyPond. Callers still write `… rescue {...}` for engine panics.
  `eui_score_bands`, `eui_score_midi` and `eui_score_pdf` take the same source.
- What it ignores (fonts, custos, baroque heads, `two-sided`…) is skipped with
  a warning, not refused. Since 2.40.3 a file that only *defines* music
  (`x = { c4 d e f }`, no `\score`) engraves its last music variable instead
  of answering "there is no music".
- 2.40.4 (eui `019fc24`) reads the rest of LilyPond's scripts after a note
  (`\upmordent`, `\downprall`, `\haydnturn`, `\thumb`, organ heels and
  toes, the early-music signs…) and, between `\\`, a part of spacer rests
  as a voice of its own. A source the engraver of an older Soli refused
  may engrave on a newer one: compare `eui_score_layout(src, …)["error"]`
  on the Soli the server runs, not on your local one.
- Each Soli release pins one eui rev (`Cargo.toml`, `eui-score`): an app
  shipping the same engine to the browser as wasm must pin the same rev.

## App metrics (`Metrics`)

```soli
Metrics.increment("orders.created", tags: {"plan": "pro"})   # counter, +1
Metrics.increment("credits", 0.5)                            # by a value (>= 0)
Metrics.gauge("queue.depth", 12, tags: {"queue": "mail"})
Metrics.histogram("cart.items", 3)                           # |h on DogStatsD
Metrics.distribution("payload.bytes", 512)                   # |d, same histogram locally
total = Metrics.time("checkout.duration", tags: {"step": "pay"}) do   # seconds
  charge(order)
end
```

- Stored only when a sink is set (`SOLI_METRICS=1` → `/_metrics`, DogStatsD via
  `DD_AGENT_HOST` / `DD_DOGSTATSD_URL` / `SOLI_STATSD=host:port`, OTLP via
  `SOLI_OTEL_METRICS=1`) or under `soli test`; otherwise validated and dropped
  (`Metrics.enabled?` says which). In a spec: `Metrics.reset` in `before_each`,
  then `Metrics.value("orders.created", {"plan": "pro"})` → `1`; a histogram
  reads back as `{"count", "sum", "buckets"}`; never recorded → `nil`.
- Names `[a-zA-Z_][a-zA-Z0-9_.]*`, scraped with dots as `_`
  (`orders_created`); `soli.`/`soli_` reserved; one name = one kind (reusing a
  counter's name as a gauge raises). Negative `increment` raises. Tag values:
  String/number/Bool/nil. Never tag with an id: 1,000 tag combinations per
  metric, then samples are dropped (counted in `soli_metrics_dropped_total`).
- `Metrics.time` takes its tags positionally or labelled. A raising block is
  still timed and the error reaches you unchanged.
- Built-in `*_seconds` series on `/_metrics` are histograms (`_bucket`, `_sum`,
  `_count`); the old bare-name total is `_sum` since 2.32.0.
  Per-route: `soli_http_request_duration_seconds{method,route,status_class}`.

## Input validation (V)

```soli
schema = {
  "email": V.string().required().email(),
  "password": V.string().required().min_length(8),
  "age": V.int().optional().min(18)
}
result = validate({"email": "a@b.co", "password": "longenough", "age": "20"}, schema)
result["valid"]     # true
result["data"]      # coerced: age => 20
bad = validate({"email": "nope", "password": "x"}, schema)
bad["errors"].map { |e| e["field"] }   # ["email", "password"]
```

Strings use `min_length`/`max_length`; `V.string().min(2)` raises
`Cannot access property 'min' on Hash`.

## Not verified here (need services)

`Cache.*` and `KV.*` need SoliKV (`Cache.set` failed without it);
`RateLimiter`, `S3`, `Imap`, `embed`/`llm_generate` — see the app's
`docs/builtins.md`.
