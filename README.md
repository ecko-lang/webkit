# Webkit - Ecko Std Lib Package

SaaS web-app batteries for [Ecko](https://ecko.sh), written in Ecko. Everything
you reach for on top of the built-in HTTP server and router:

- **Native `template` functions** for views, with explicit escaping
  (`webkit.escape` / `webkit.e`) so XSS holes are visible in the code.
- **HMAC-signed cookies**, a **session** helper, and **flash messages** (built
  on `hash.hmac_sha256` + `random.token`).
- **Form validation** (`webkit.validate`) for required fields, types, and
  min/max/pattern rules.
- **CORS** and **security-header** middleware for `web.router`.

## Install

```bash
ecko get github.com/ecko-sh/webkit
```

## Signed cookies & sessions

```ecko
signed = webkit.sign("user=42", SECRET)   # "user=42.<hmac>"
webkit.unsign(signed, SECRET)             # "user=42", or null if tampered

s = webkit.session_new(SECRET)            # { id, cookie }
# ...send s.cookie as a Set-Cookie header...
webkit.session_read(req.headers.cookie, SECRET)   # the id, or null
```

`webkit.cookie(name, value, { path, max_age, http_only, secure, same_site })`
builds a `Set-Cookie` string; `webkit.parse_cookies(header)` reads a request
`Cookie` header into a map.

## Middleware

Drop these into `web.router`'s middleware list:

```ecko
import std.web
import webkit

app = web.router(
    routes,
    [
        webkit.security_headers(),        # nosniff, DENY framing, referrer policy
        webkit.cors({ origin: "*" }),     # allow-origin + OPTIONS preflight
    ],
)
```

## Framework

Assemble a whole app with `webkit.app`:

```ecko
import std.http
import std.web
import webkit

app = webkit.app({
    routes: [
        web.get("/", |req| webkit.html("<h1>Home</h1>")),
        webkit.static("/assets", "public"),
    ],
    middleware: [webkit.cache_control([{ prefix: "/assets", value: "public, max-age=86400" }])],
    security: true,
    errors: insert(empty_map(), "404", |req| webkit.html("<h1>Not found</h1>")),
})
http.serve(8080, app)
```

- `webkit.file(path, { cache })` serves one file fresh (no read-once staleness).
- `webkit.redirect(url)`, `webkit.abort(404)`, `webkit.json/html/text(v, { cache })`,
  `webkit.with_headers`, `webkit.cache`, `webkit.content_type`.
- **Escaping is explicit** in native templates - call `webkit.escape(x)` (alias
  `webkit.e`) on untrusted data. A forgotten escape is an XSS hole.

The framework serves files and calls the built-in HTTP/router (both `net`-gated),
so grant webkit `fs:read` **and** `net` in your app's `ecko.json`:
`"dependencies": { "webkit": { "path": "github.com/ecko-sh/webkit", "version": "v0.9.1", "grant": ["fs:read", "net"] } }`.

### Requests & routing

```ecko
# In a handler, read the request:
q    = webkit.query(req, "search", "")     # query string param + default
n    = webkit.query_int(req, "page", 1)    # parsed int + default
name = webkit.form(req, "name")            # posted form field
body = webkit.json_body(req)               # decoded JSON body (or null)
uid  = webkit.session(req, SECRET)         # signed session id (or null)

# Group routes with a blueprint (prefix + optional group middleware):
api = webkit.blueprint("/api", [
    web.get("/users", list_users),
    web.post("/users", create_user),
], [require_auth])

app = webkit.app({ routes: [web.get("/", home), api] })

# Build URLs without hardcoding:
webkit.url_for("/user/:id", insert(empty_map(), "id", 42))   # "/user/42"
```

### Templating & escaping

Views are **native Ecko `template` functions** - variables, loops, conditionals,
and partials (a template is a function, so `{header(title)}` includes one):

```ecko
template layout(title, body) = """<!doctype html><title>{title}</title>{body}"""
template row(item) = """<li>{webkit.e(item.name)} - {item.qty}</li>"""
template list(items) = """<ul>{for it in items}{row(it)}{end}</ul>"""
```

**Escaping is explicit.** Native `{expr}` is raw (so AI-prompt templates stay
untouched), so wrap untrusted data in `webkit.escape(x)` / `webkit.e(x)`:

```ecko
template greet(name) = """<h1>Hi {webkit.e(name)}</h1>"""   # safe
```

A forgotten `e()` on user data is an XSS hole - escape every value that came
from a request, a database, or a model.

### Flash & forms

```ecko
resp = webkit.flash(webkit.redirect("/"), "Profile saved", SECRET)   # set on the response
msgs = webkit.flashes(req, SECRET)                                   # read on the next request

result = webkit.validate(req.form, insert(insert(empty_map(),
    "email", insert(empty_map(), "type", "email")),
    "age", insert(insert(empty_map(), "type", "int"), "min", 18)))
# result.valid / result.errors / result.values
```

## Testing

```bash
ecko test          # offline: escaping, cookies, sessions, flash, validation, middleware, framework (41 cases)
```

## License

MIT
