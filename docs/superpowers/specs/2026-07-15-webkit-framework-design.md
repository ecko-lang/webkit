# webkit → an Ecko web framework - design

Status: draft for review · Date: 2026-07-15 · Owner: Sean

## Context & goal

`webkit` today is a small set of web-app helpers (auto-escaping `{{ }}`
templates, HMAC-signed cookies, sessions, CORS and security-header middleware).
Building an actual app on `std.web`/`std.http` still means hand-assembling a lot
of boilerplate - the `website/splash` server enumerates every static asset as an
explicit route, hand-rolls content-types and `cache-control`, reads its HTML
once at startup (so edits don't show), and repeats health-check and
response-header code.

Goal: grow `webkit` into a **decent, Flask-convenient web framework**, expressed
in Ecko's idioms (immutable, declarative, no decorators) rather than a literal
Flask port. The centerpiece is a declarative `webkit.app(spec)` that compiles a
whole app into one `http.serve`-compatible handler, plus the missing helpers for
static files, responses, requests, errors, routing composition, and forms.

### What Ecko already provides (don't rebuild)

- **Routing:** `web.router([...])`, `web.get/post/put/delete/patch`, `:params`.
- **Request map:** `{ method, path, params, headers, body, json, form, files }`.
- **Responses:** `http.text/html/json/response(status, body, headers)`; bytes bodies.
- **Middleware:** `|req, next|` chains via `web.router`'s second argument.
- **Templates:** the native `template name(...) = """..."""` feature - variables
  (`{expr}`), loops (`{for x in xs}…{end}`), conditionals (`{if c}…{else}…{end}`),
  and partials (a template is a first-class function, so `{header(t)}` includes it).
- **Sessions/cookies/CORS/security headers/escaping:** existing webkit.
- **Dev server:** `http.serve` + `ecko dev`.

## Key decisions (resolved during design)

1. **API shape: Ecko-idiomatic declarative.** A `webkit.app(spec)` assembly that
   builds on `web.router`, not a mutable `@app.route` object.
2. **Templating: unify on the native engine.** Drop webkit's separate `{{ }}`
   engine. Views are native `template` functions. **No kernel change.**
3. **Escaping: explicit, not automatic.** Native `{expr}` stays raw everywhere
   (AI-prompt templates must not be HTML-escaped). webkit re-exports `escape`
   and a short alias `e`; the web convention is `{escape(x)}` for untrusted data.
   **This drops webkit's former escaped-by-default guarantee** - surfaced loudly
   in docs/examples as an XSS responsibility.
4. **Static files: reuse the hardened native server, no core change.** Serve
   directories through `std.web`'s `web.static` (path-traversal protection was
   security-audited; its mime table already covers html/css/js/json/svg/images/
   fonts). Caching is added by a **pure-webkit middleware** that wraps the router
   and stamps `cache-control` by path - so no `std.web`/kernel change is needed.
5. **`url_for`: pattern-based**, not a named-endpoint registry
   (`url_for("/user/:id", {id: 5})`), to avoid webkit owning route identity.
6. **Layer:** `webkit` stays a Layer-3 pure-Ecko package. File-reading helpers
   need the `fs:read` capability granted; everything else is cap-free.

## No core change needed

`std.web`'s `web.static` already serves correct content-types (its `mime_for`
covers `text/html`, `text/css`, `text/javascript`, `application/json`,
`image/svg+xml`, images, and fonts), and its path-traversal protection is
already hardened. Caching - the one thing it doesn't do - is added by a
**webkit middleware** (`webkit.cache_control`) that wraps the router and stamps
`cache-control` on responses by path. So the whole framework is a self-contained
Layer-3 package: **zero changes to `ecko-core`/`ecko-std`.**

## Architecture

`webkit.app(spec)` compiles a spec map into a single handler
`|req| -> response` suitable for `http.serve(port, handler)`:

```ecko
app = webkit.app({
    routes: [
        web.get("/", home),
        webkit.static("/assets", "public", { cache: "public, max-age=86400" }),
        webkit.blueprint("/api", api_routes),
    ],
    middleware: [ my_mw ],        # extra |req, next|
    cors:       { origin: "*" },  # convenience → webkit.cors
    security:   true,             # → webkit.security_headers
    sessions:   SECRET,           # enables sessions + flash (stashes the secret)
    errors:     { 404: not_found_page, 500: error_page, 405: method_page },
})
http.serve(port, app)
```

Compilation:
1. Expand `routes` (blueprints → prefixed entries; `webkit.static` → route entries).
2. Build the middleware list in order: `security` → `cors` → sessions → user `middleware`.
3. `router = web.router(expanded_routes, middleware)`.
4. Wrap `router` with error handling: `try { router(req) } catch (e) { errors.500(req, e) }`;
   if the result is a 404 and an `errors.404` exists, call it; likewise 405.
5. Return the wrapped handler.

Each concern is an independent unit usable without `app` (you can drop
`webkit.static(...)` straight into a plain `web.router` list).

## Components

Responses are the standard `{ status, headers, body }` maps. All helpers are
pure functions of their inputs (easy to unit-test with mock `req` maps).

### 1. App assembly
- `webkit.app(spec) -> handler` - as above.

### 2. Static & single files (needs `fs:read`)
- `webkit.static(prefix, dir) -> route` - passthrough to native `web.static`
  (kept in webkit so an app declares all routes through one namespace).
- `webkit.cache_control(rules) -> middleware` - a `|req, next|` that stamps
  `cache-control` on the response by path prefix, e.g.
  `cache_control([{ prefix: "/assets", value: "public, max-age=86400" }])`.
  This is how static mounts get caching without a core change.
- `webkit.file(path, opts?) -> response` - serve one file **fresh** (reads on each
  call, so edits show), with `content_type(path)` + optional `opts.cache`.
  Replaces the read-once `page = fs.read(...)` pattern.
- `webkit.content_type(path) -> string` - extension → mime (webkit-local table;
  public so single-file handlers can use it).

### 3. Responses
- `webkit.redirect(url, status?) -> response` - 302 (or given), `location` header.
- `webkit.abort(status, body?)` - raises a structured error `{ kind: "http",
  status, message }`. The `app` error layer inspects a caught error's `kind`: an
  `http` error dispatches to that `status`'s handler (or a default page); any
  other thrown value is a genuine bug and goes to the 500 handler.
- `webkit.json(value, opts?)`, `webkit.html(body, opts?)`, `webkit.text(s, opts?)` -
  thin over `http.*` but accept `opts.cache` / `opts.headers`.
- `webkit.with_headers(resp, map) -> response`, `webkit.cache(resp, value) -> response` -
  attach headers / cache-control (the existing private `with_header`, made public).

### 4. Request access
- `webkit.query(req, key, default?)`, `webkit.query_int(req, key, default?)`
- `webkit.form(req, key, default?)`, `webkit.json_body(req)`
- `webkit.cookies(req) -> map` (wraps `parse_cookies` on the Cookie header)
- `webkit.session(req, secret)` (wraps `session_read`)

### 5. Error handling
- Driven by the `errors: { 404, 405, 500 }` spec entry; each is `|req, ...| -> response`.
- `app` intercepts router 404/405 and thrown-handler exceptions and dispatches to
  the handler, falling back to plain default pages. `webkit.abort(code)` triggers them.

### 6. Blueprints (route groups)
- `webkit.blueprint(prefix, routes, middleware?) -> route-group` - expands to each
  route with `prefix` prepended and (optionally) group-scoped middleware wrapped
  around the group's handlers. Composable inside `app({ routes })`.

### 7. url_for
- `webkit.url_for(pattern, params?, query?) -> string` - fills `:name` segments
  from `params` and appends a query string, e.g.
  `url_for("/user/:id", { id: 5 }, { tab: "x" })` → `/user/5?tab=x`.

### 8. Flash messages (needs `sessions`)
- `webkit.flash(resp, message, category?) -> response` - appends a message to a
  signed `_flash` cookie (built on `sign`).
- `webkit.flashes(req, secret) -> list` and clears them (a `Set-Cookie` that
  expires `_flash`, returned via a helper the caller applies to its response).

### 9. Form validation
- `webkit.validate(form, schema) -> { valid, errors, values }` - schema like
  `{ email: { required: true, type: "email" }, age: { type: "int", min: 0 } }`;
  supported rules: `required`, `type` (`int`/`float`/`email`/`url`/`bool`),
  `min`/`max` (numeric or length), `pattern` (regex via `std.re`). `values`
  carries coerced values; `errors` is `{ field: message }`.

### 10. Templating (native engine + escaping helper)
- **No engine in webkit.** Views are native `template` functions in `.ecko`
  modules; loops/conditionals/partials are the native feature.
- `webkit.escape(s)` (kept) and `webkit.e(s)` (alias) - escape untrusted data
  inside a template: `template row(u) = """<td>{webkit.e(u.name)}</td>"""`.
- **Removed:** the old `render(tmpl_string, data)` `{{ }}` engine. (Breaking for
  current webkit template users - see Migration.)
- The guide shows layout + partials via composition:
  ```ecko
  template layout(title, body) = """<!doctype html><title>{title}</title><body>{body}</body>"""
  template home(name) = """<h1>Hi {webkit.e(name)}</h1>"""
  # layout("Home", home(user_name))  - inner HTML interpolates raw into the outer
  ```

## Capabilities

- `webkit`'s `ecko.json` declares `["fs:read", "net"]` (advisory). `fs:read` for
  file reads; **`net`** because `static`/`app`/response helpers call `std.web`/
  `std.http`, which are `net`-gated. Importers grant both. Pure helpers that
  touch neither (sign/unsign, escape, url_for, validate) work without a grant,
  but an app using the framework grants `["fs:read", "net"]`.
  *(Discovered in implementation: found via Phase 1.)*

## Testing

- All offline/deterministic, matching the current 13-case suite: call helpers
  with mock `req` maps and temp fixture files; assert `{ status, headers, body }`.
- Cover: app assembly (routing, middleware order, error dispatch), static
  (content-type, cache, index, 404, traversal still blocked), `file` freshness,
  request accessors, `redirect`/`abort`, blueprints (prefixing), `url_for`,
  flash round-trip, validation rules, and the escaping convention.
- Add an end-to-end example: **rebuild `website/splash` on the framework** as the
  acceptance test (fewer lines, no read-once staleness, cache preserved).

## Migration / breaking changes

- **`webkit.render` (the `{{ }}` engine) is removed.** Existing string-template
  users move to native `template` functions with `{webkit.e(...)}`. webkit is
  early (13 tests, few users), so acceptable; call it out in the CHANGELOG.
- **Escaping posture changes** from auto to explicit - the docs must make the
  `{escape(x)}` convention and its XSS stakes prominent.
- `webkit` gains an `fs:read` capability requirement for its file helpers.

## Build phases (one spec, sequenced implementation)

1. **Core (this plan):** `webkit.content_type`, `file`, `static`,
   `cache_control`, response helpers (`redirect`/`abort`/`with_headers`/`cache`/
   `json`/`html`/`text`), `webkit.app` + error handling. Rebuild `website/splash`
   on it. No `ecko-core`/`ecko-std` change.
2. **Requests & routing:** request accessors, `blueprint`, `url_for`.
3. **State & forms:** `flash`, `validate`. Escaping helper (`e`) + templating
   guide; remove `render`.

## Open items

- Exact `spec` key names for `webkit.app` (e.g. `errors` vs `on_error`) - settle
  during implementation; keep them boring and Flask-recognizable.
- Whether `webkit.static` needs per-extension cache overrides (e.g. long-cache
  images, short-cache HTML) in v1 or a v2 follow-up. Default: single `cache` per
  mount in v1.
