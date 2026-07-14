# webkit

SaaS web-app batteries for [Ecko](https://ecko.sh), written in Ecko. Everything
you reach for on top of the built-in HTTP server and router:

- **Auto-escaping HTML templates** — XSS-safe by default (`{{ }}` escapes,
  `{{{ }}}` opts out).
- **HMAC-signed cookies** and a **session** helper (built on `hash.hmac_sha256`
  + `random.token`).
- **CORS** and **security-header** middleware for `web.router`.

Pure Ecko, no capabilities required.

## Install

```bash
ecko add https://github.com/ecko-sh/webkit
```

## Templates

```ecko
import webkit

webkit.render(r"<h1>Hi {{ name }}</h1>", { name: "Ada <3" })
# -> "<h1>Hi Ada &lt;3</h1>"      (escaped by default)

webkit.render(r"<div>{{{ body }}}</div>", { body: trusted_html })
# -> raw insertion, only for HTML you trust
```

Pass templates as **raw strings** (`r"..."`) or file content, so Ecko's own
`{expr}` interpolation leaves the `{{ }}` holes alone.

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

## Testing

```bash
ecko test          # offline: escaping, templates, cookies, middleware (13 cases)
```

## License

MIT
