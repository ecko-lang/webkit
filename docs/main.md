# webkit

## `with_headers(resp, hmap)`

with_headers(resp, hmap) -> `resp` with every name/value merged into headers.

Existing headers are kept unless `hmap` names them.

## `escape(s)`

escape(s) -> `s` with the five HTML-significant characters replaced.

`&`, `<`, `>`, `"` and `'` become entities, which makes the result safe in
element text and in quoted attribute values. It is not safe unquoted, nor
inside a `<script>` or `<style>` body, nor in a URL - those need their own
encoding.

## `e(s)`

e(s) -> the same as `escape`, named short for use inside templates.

```ecko
html("<p>Hello, {e(name)}</p>")
```

## `sign(value, secret)`

sign(value, secret) -> "value.mac", the value with an HMAC-SHA256 appended.

The value is readable by anyone holding the cookie - signing proves it was
not altered, it does not hide it. Do not sign anything you would not show the
user.

## `unsign(signed, secret)`

unsign(signed, secret) -> the original value, or null if it does not verify.

Null covers every failure the same way: tampered value, wrong secret,
truncated token, or not a signed string at all. The MAC comparison does not
stop at the first differing character, so it does not leak how much of a
forged MAC was correct.

## `cookie(name, value, opts)`

cookie(name, value, opts) -> a Set-Cookie header string.

`opts` is all optional: `path`, `max_age` in seconds, `http_only`, `secure`,
and `same_site` ("Strict", "Lax" or "None"). Nothing is set by default, so
for a session cookie pass `http_only` and, over HTTPS, `secure`.

```ecko
cookie("sid", tok, { path: "/", http_only: true, same_site: "Lax" })
```

## `parse_cookies(header)`

parse_cookies(header) -> a { name: value } map from a request Cookie header.

An empty map for a null or unparseable header.

## `session_new(secret)`

session_new(secret) -> { id, cookie } for a fresh session.

`id` is a 32-byte random token to key your own store by; `cookie` is the
Set-Cookie header carrying it signed, HttpOnly, SameSite=Lax, for a week.

```ecko
s = session_new(secret)
with_headers(redirect("/"), { "set-cookie": s.cookie })
```

## `session_read(header, secret)`

session_read(header, secret) -> the session id from a Cookie header, or null.

Null means no session, not a forged one - a tampered cookie reads the same as
an absent one, which is what you want at a route boundary.

## `content_type(path)`

content_type(path) -> a MIME type for the file's extension.

Covers the web's common types and falls back to
`application/octet-stream`, which browsers download rather than render.

## `file(path, opts = empty_map())`

file(path, opts?) -> a response serving `path`, or a 404 if it is missing.

The file is read on every call, so edits show up without a restart; that also
means it is not free. `opts.cache` sets cache-control. For a whole directory
use `static`, which is traversal-safe.

## `cache(resp, value)`

cache(resp, value) -> `resp` with a cache-control header.

## `redirect(url, status = 302)`

redirect(url, status?) -> a redirect response, 302 by default.

Use 301 or 308 only when the move is permanent - browsers cache those hard
enough that a mistake outlives the fix.

## `abort(status, body = "")`

abort(status, body?) -> raises an error the app's error layer turns into a page.

This is how a handler gives up mid-request: `abort(404)` from three calls deep
reaches the registered 404 handler without every caller checking a return.

```ecko
post = find(id) ; if post == null { abort(404) }
```

## `html(body, opts = empty_map())`

html(body, opts?) -> an HTML response. `opts.cache` sets cache-control.

The body is sent as given, so escape anything user-supplied with `e`.

## `json(value, opts = empty_map())`

json(value, opts?) -> a JSON response. `opts.cache` sets cache-control.

## `text(s, opts = empty_map())`

text(s, opts?) -> a plain-text response. `opts.cache` sets cache-control.

## `static(prefix, dir)`

static(prefix, dir) -> a route serving `dir` under `prefix`.

Delegates to the native `web.static`, which resolves paths safely: `..` in a
request cannot escape `dir`. Add caching with the `cache_control`
middleware.

## `cache_control(rules)`

cache_control(rules) -> middleware setting cache-control by path prefix.

`rules` is `[{ prefix, value }]` and the first matching prefix wins, so order
from most specific to least.

```ecko
cache_control([{ prefix: "/assets", value: "public, max-age=31536000" }])
```

## `cors(opts)`

cors(opts) -> middleware adding access-control headers and answering preflight.

`opts.origin` defaults to `"*"`, which is right for a public API and wrong for
anything using cookies - a browser refuses credentialed requests to a wildcard
origin. An OPTIONS request is answered 204 without reaching your routes.

## `security_headers()`

security_headers() -> middleware hardening every response.

Sets `nosniff`, `X-Frame-Options: DENY` and a strict-origin referrer policy.
Framing is denied outright, so if the page must be embedded, set your own
frame policy instead of using this.

## `query(req, key, default = null)`

query(req, key, default?) -> a query-string or path parameter as text.

`default` (null unless given) is returned when the key is absent.

## `query_int(req, key, default = 0)`

query_int(req, key, default?) -> a query parameter as a whole number.

Absent or unparseable both give `default` (0 unless given), so `?page=abc`
cannot crash a handler.

## `form(req, key, default = null)`

form(req, key, default?) -> a submitted form field as text.

## `json_body(req)`

json_body(req) -> the decoded JSON body, or null when there was none.

## `cookies(req)`

cookies(req) -> a { name: value } map of the request's cookies.

These are raw and unverified. For a signed value use `session` or `flashes`.

## `session(req, secret)`

session(req, secret) -> the verified session id, or null.

## `url_for(pattern, params = empty_map(), query = empty_map())`

url_for(pattern, params?, query?) -> a URL built from a route pattern.

`:name` segments are filled from `params`; `query` is appended sorted and
percent-encoded, so the same inputs always produce the same URL.

```ecko
url_for("/posts/:id", { id: 7 }, { page: 2 })   # "/posts/7?page=2"
```

## `flash(resp, message, secret)`

flash(resp, message, secret) -> `resp` carrying `message` to the next request.

Messages accumulate: calling it twice on one response stages both. Read them
with `flashes` and clear them with `clear_flash`, or they show up again.

## `flashes(req, secret)`

flashes(req, secret) -> the messages staged by the previous response.

An empty list when there are none, or when the cookie fails to verify.

## `clear_flash(resp)`

clear_flash(resp) -> `resp` expiring the flash cookie.

Do this on the response that displays the messages, otherwise the next page
shows them again.

## `validate(form, schema)`

validate(form, schema) -> { valid, errors, values }.

Each field's rules are `{ required, type, min, max, pattern }`, where `type`
is "string" (default), "int", "email" or "url". `min`/`max` compare the number
for an int and the length for anything else.

`values` holds only the fields that passed, coerced to their type, so an int
field arrives as an int rather than as text.

```ecko
r = validate(req.form, { email: { required: true, type: "email" } })
if not r.valid { return html(form_with(r.errors)) }
```

## `blueprint(prefix, routes, mw = [])`

blueprint(prefix, routes, mw?) -> `routes` with `prefix` on each path.

Middleware in `mw` wraps only these routes, which is how a section of the site
gets, say, authentication without the rest paying for it. Pass the result
straight into an app's `routes` and it splices in.

## `app(spec)`

app(spec) -> a request handler for `http.serve`.

`spec` is `{ routes, middleware, errors, security, cors }`. Middleware runs
outermost-first in the order given, after `security` and `cors` if those are
set. `errors` maps a status as text to a handler, so `{ "404": page }`.

Every error handler is called as `handler(req, err)`. `err` is null when the
router simply found no route, so one 404 handler serves both that case and
an explicit `abort(404)`.

```ecko
handler = app({ routes: [...], security: true, errors: { "404": not_found } })
http.serve(8080, handler)
```
