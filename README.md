# Webkit - Ecko Std Lib Package

Web-app batteries for Ecko: HTML templates that escape by default, CSRF
protection, sessions that can be revoked, and the middleware around them.

It builds on `std.web` rather than replacing it. The router stays where it is,
so `web.get("/u/:handle", profile)` is still how you declare a route, and you
reach for webkit only where you need it.

## Install

```bash
ecko get github.com/ecko-lang/webkit
```

```ecko
import webkit
```

Needs Ecko 0.27.0 or later, for `hash.constant_eq`.

## Usage

```ecko
import std.web
import std.http
import std.os
import webkit

SECRET = secret(os.env("APP_SECRET"))

fn home(req) {
    webkit.html(
        webkit.render(
            r"""<h1>Hello, {name}</h1>
            <form method="post" action="/post">{csrf}
              <textarea name="body"></textarea>
              <button type="submit">Post</button>
            </form>""",
            { name: webkit.query(req, "name", "world"), csrf: webkit.csrf_field(req) },
        ),
    )
}

fn create(req) {
    print(webkit.form(req, "body", ""))
    webkit.redirect("/", 303)
}

app = webkit.app(
    {
        routes: [web.get("/", home), web.post("/post", create)],
        security: true,
        csrf: reveal(SECRET),
    },
)

http.serve(8080, app)
```

That is a whole program. The form posts, `{name}` is escaped on the way out,
and the POST is refused without the token `{csrf}` puts in the form.

## Templates that escape by default

`render` takes a raw string and a map of values, and escapes every value it
substitutes. It returns markup, not a string, and `webkit.html` accepts only
markup. That is the whole design:

> An XSS hole requires you to type `raw()`. It cannot happen by forgetting.

```ecko
webkit.render(r"""<p>{body}</p>""", { body: "<script>alert(1)</script>" })
# <p>&lt;script&gt;alert(1)&lt;/script&gt;</p>

webkit.html("<p>hi</p>")   # refused: expected markup from render() or raw()
```

**Write templates as `r"""..."""`.** A plain `"..."` string has Ecko interpolate
`{body}` at parse time, before escaping could happen, which is the hole this
closes. The triple-quoted raw form leaves both `{name}` and HTML's double
quotes alone, and dedents multi-line blocks.

Fragments compose, because a value that is already markup is spliced rather
than escaped again:

```ecko
rows = []
for p in posts {
    rows = push(rows, webkit.render(r"""<li>{body}</li>""", { body: p.body }))
}
webkit.render(r"""<ul>{rows}</ul>""", { rows: rows })   # a list joins itself
```

`{{` and `}}` are literal braces, for inline CSS and JavaScript. An unknown
placeholder raises rather than rendering blank, so a typo is a failure you see
instead of a hole in the page you do not.

There is **no control flow in templates** - no loops, no conditionals, no
filters. Build fragments with Ecko code and compose them, as above. Ecko is
already a capable language, and a second weaker one inside string literals
would be less predictable, invisible to `ecko fmt` and unreachable by
`ecko check`. The cost is real and worth knowing: a page with deep conditional
structure reads as Ecko functions returning markup, not as one template file.

`raw(s)` is the audited escape hatch and the only way unescaped text reaches a
page. Grep for it in review.

## CSRF

Give `app` a secret and every state-changing route is protected:

```ecko
app = webkit.app({ routes: routes, csrf: reveal(SECRET) })
```

Put `{csrf}` in every form that POSTs, from `webkit.csrf_field(req)`. A
`POST`, `PUT`, `PATCH` or `DELETE` without a valid token is answered 403.

The token is a signed double-submit cookie, so it works on a login form too,
where there is no session yet. Login CSRF is a real attack and a session-bound
token would leave exactly that form unprotected.

`GET`, `HEAD` and `OPTIONS` are exempt, on the assumption they are side-effect
free. An app that changes state on `GET` is outside what this can defend.

## Sessions

Two stores, one interface - `start(resp, data)`, `read(req)`, `end(req, resp)` -
so swapping between them does not touch handler code.

```ecko
# Zero setup: the data travels in a signed cookie.
sessions = webkit.cookie_store(reveal(SECRET))

# Server-side: only a random id travels, the data lives wherever you put it.
sessions = webkit.store(
    {
        load: fn(id) db.session_user(DB, id),
        save: fn(id, data) db.put_session(DB, id, data.user),
        delete: fn(id) db.end_session(DB, id),
    },
)

resp = sessions.start(webkit.redirect("/", 303), { user: id })
who = sessions.read(req)
resp = sessions.end(req, webkit.redirect("/", 303))
```

**Pick the cookie store for convenience, the server store for control.** A
cookie session is visible to whoever holds the cookie. Signing proves it was not
altered, it does not hide it.

It also **cannot be revoked**. The only copy is the one the client holds, so
expiring the cookie asks a browser to forget it, and an attacker who kept the
value still has a session. A server store is what makes logout-everywhere
real.

`require_session` turns anonymous requests away, and attaches the session so
handlers below read it with `session_of(req)`:

```ecko
webkit.blueprint("", authed_routes, [webkit.require_session(sessions)])
```

Being middleware is the point: a route added to that list is protected by being
in the list, rather than by remembering to call something.

## API

| call | what it does |
|---|---|
| `render(tpl, values?)` | a template with every `{name}` substituted and escaped |
| `raw(s)` / `is_safe(v)` | mark text as already-safe; test whether a value is |
| `join_safe(items, sep?)` | render and join fragments into one piece of markup |
| `escape(s)` | escape one value, for markup built by hand |
| `html(body, opts?)` | an HTML response; refuses anything but markup |
| `json(v, opts?)` / `text(s, opts?)` | JSON and plain-text responses |
| `file(path, opts?)` / `static(prefix, dir)` | serve a file; serve a directory |
| `content_type(path)` | the MIME type for a file extension |
| `redirect(url, status?)` / `abort(status, body?)` | redirect; give up mid-request |
| `cache(resp, v)` / `cache_control(rules)` | set cache-control, directly or by prefix |
| `csrf_protect(secret, opts?)` | middleware enforcing the token |
| `csrf_field(req)` / `csrf_token(req)` | the hidden input; the raw token |
| `cookie_store(secret, opts?)` / `store(handlers, opts?)` | the two session stores |
| `require_session(st, opts?)` / `session_of(req)` | the guard; the session it attached |
| `cookie(name, v, opts)` / `parse_cookies(h)` / `cookies(req)` | cookies |
| `sign(v, secret)` / `unsign(signed, secret)` | HMAC-signed values |
| `query(req, k, d?)` / `query_int(req, k, d?)` | query and path parameters |
| `form(req, k, d?)` / `json_body(req)` | submitted form fields; a JSON body |
| `flash(resp, msg, secret)` / `flashes(req, secret)` / `clear_flash(resp)` | flash messages |
| `validate(values, schema)` | form validation |
| `app(spec)` / `blueprint(prefix, routes, mw?)` | assemble an app; group routes |
| `url_for(pattern, params?, query?)` | build a URL from a route pattern |
| `cors(opts)` / `security_headers()` / `with_headers(resp, h)` | headers |

## Notes

**Comparisons are constant-time.** Signatures, session ids and CSRF tokens are
compared with `hash.constant_eq`, which is native, so how long a check takes
does not reveal how much of a forged value was right. Length is not hidden, only
content, and every value compared here is fixed-length.

**Not included:** multipart form data, so no file uploads.

## Testing

```bash
ecko test tests/
```

## License

MIT
