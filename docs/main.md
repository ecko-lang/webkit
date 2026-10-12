# webkit

## `with_headers(resp, hmap)`

with_headers(resp, hmap) -> `resp` with every name/value merged into headers.

Existing headers are kept unless `hmap` names them.

## `is_safe(v)`

is_safe(v) -> true when `v` came from `render` or `raw`.

A plain string is not safe, and neither is a map that happens to have an
`html` key - only the real type passes.

## `raw(s)`

raw(s) -> `s` marked safe, spliced into templates without escaping.

The audited escape hatch, and the only way unescaped text reaches a page.
Everything passed here is trusted completely, so it must never carry user
input that has not already been escaped. Grep for it in review.

```ecko
raw("<br>")
```

## `join_safe(items, sep = "")`

join_safe(items, sep?) -> the items rendered and joined into one Safe.

Each item follows the same rule as a substitution: already-Safe passes
through, anything else is escaped. This is how a list of rows becomes one
fragment when you want a separator between them; a bare list substituted
into `render` is joined with nothing.

```ecko
join_safe(["a", "b"], ", ")
```

## `render(tpl, values = empty_map())`

render(tpl, values?) -> a Safe with every `{name}` substituted.

Every substituted value is escaped unless it is already Safe, which is what
makes nesting compose: a fragment built by `render` splices into another
`render` without being escaped twice. `{{` and `}}` are literal braces, for
inline CSS and JavaScript.

An unknown placeholder raises rather than rendering blank, so a typo is a
failure you see rather than a hole in the page you do not.

```ecko
render(r"""<p class="note">{body}</p>""", { body: user_text })
```

## `escape(s)`

escape(s) -> `s` with the five HTML-significant characters replaced.

`&`, `<`, `>`, `"` and `'` become entities, which makes the result safe in
element text and in quoted attribute values. It is not safe unquoted, nor
inside a `<script>` or `<style>` body, nor in a URL - those need their own
encoding.

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
post = find_post(id)
if post == null { abort(404) }
```

## `html(body, opts = empty_map())`

html(body, opts?) -> an HTML response. `opts.cache` sets cache-control.

`body` must be markup from `render` or `raw`, never a plain string. The
refusal is the point. Every route to the wire runs through a value that was
either escaped or explicitly vouched for, so forgetting cannot put unescaped
text on the page.

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

## `require_session(st, opts = empty_map())`

require_session(st, opts?) -> middleware turning anonymous requests away.

Reads the session with `st` and, when there is none, redirects to
`opts.redirect` ("/login" unless given). When there is one it is attached to
the request, so handlers beneath read it with `session_of(req)` instead of
looking it up again.

Being middleware rather than a per-handler call is the point: a whole
blueprint can require a session without every handler restating it, and a
handler added later cannot forget.

```ecko
blueprint("/app", routes, [require_session(st)])
```

## `session_of(req)`

session_of(req) -> the session attached by `require_session`, or null.

## `cookie_store(secret, opts = empty_map())`

cookie_store(secret, opts?) -> a store keeping the session data in the cookie.

Needs nothing but a secret. The data is signed, so it cannot be altered, and
it is readable by whoever holds the cookie, so do not put anything in it you
would not show the user. It cannot be revoked: see the module header.

`opts` takes `max_age` in seconds and `secure` for HTTPS-only.

```ecko
st = cookie_store(SECRET)
resp = st.start(redirect("/"), { user: id })
```

## `store(handlers, opts = empty_map())`

store(handlers, opts?) -> a store keeping only a random id in the cookie.

`handlers` is `{ load, save, delete }`, called as `load(id)`, `save(id, data)`
and `delete(id)`. Back them with whatever you like: `std.sql`, a `cell`, a
cache. The id is a 32-byte random token and is not signed, because guessing
one is the same problem as guessing a signature and the lookup already fails
closed.

```ecko
st = store({
    load:   fn(id) db.session(DB, id),
    save:   fn(id, data) db.put_session(DB, id, data),
    delete: fn(id) db.drop_session(DB, id),
})
```

## `csrf_token(req)`

csrf_token(req) -> the CSRF token for this request, or null outside
`csrf_protect`.

`csrf_protect` puts it on the request, so any handler beneath it can read
it. Most code wants `csrf_field` instead.

## `csrf_field(req)`

csrf_field(req) -> the hidden input carrying the token, as markup.

Splice it into every form that POSTs, since without it the request is
refused. It is Safe markup, so it goes straight into a `render` template.

```ecko
render(r"""<form method="post">{csrf}...</form>""", { csrf: csrf_field(req) })
```

## `csrf_protect(secret, opts = empty_map())`

csrf_protect(secret, opts?) -> middleware refusing unsafe requests without a
token.

Issues the token cookie when there is not already a valid one, checks
`POST`/`PUT`/`PATCH`/`DELETE` against it, and answers 403 on a mismatch. The
comparison does not exit early, so it does not leak how much of a guess was
right.

`app(spec)` installs this for you when given a `csrf` secret, so reach for it
directly only when assembling a router by hand.

## `query(req, key, default = null)`

query(req, key, default?) -> a query-string or path parameter as text.

`default` (null unless given) is returned when the key is absent.

## `query_int(req, key, default = 0)`

query_int(req, key, default?) -> a query parameter as a whole number.

Absent or unparseable both give `default` (0 unless given), so `?page=abc`
cannot make a handler throw.

## `form(req, key, default = null)`

form(req, key, default?) -> a submitted form field as text.

## `json_body(req)`

json_body(req) -> the decoded JSON body, or null when there was none.

## `cookies(req)`

cookies(req) -> a { name: value } map of the request's cookies.

These are raw and unverified. For a signed value use `session` or `flashes`.

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

`spec` is `{ routes, middleware, errors, security, cors, csrf }`. Middleware
runs outermost-first in the order given, after `security`, `cors` and `csrf`
if those are set. `errors` maps a status as text to a handler, so
`{ "404": page }`.

`csrf` is a secret, and giving one installs CSRF protection over every
state-changing route. Forms then need `csrf_field(req)` in them, and a form
without it is refused. Leaving `csrf` out leaves those routes unprotected,
which is almost never what you want for anything with a session.

Every error handler is called as `handler(req, err)`. `err` is null when the
router simply found no route, so one 404 handler serves both that case and
an explicit `abort(404)`.

```ecko
handler = app({ routes: routes, security: true, errors: { "404": not_found } })
http.serve(8080, handler)
```
