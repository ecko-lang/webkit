# webkit Framework - Phase 1 (Core) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the core of a Flask-convenient web framework to the `webkit` package - file/static serving, response & error helpers, and a `webkit.app(spec)` assembly - then prove it by rebuilding the splash server on it.

**Architecture:** Pure-Ecko Layer-3 additions to `webkit/main.ecko`, built over `std.http`/`std.web`/`std.fs`. `webkit.app(spec)` compiles a declarative spec into one `http.serve`-compatible handler by wrapping `web.router` with middleware + error handling. No changes to `ecko-core`/`ecko-std`.

**Tech Stack:** Ecko (`.ecko`), `std.test` (offline test runner), `std.http`, `std.web`, `std.fs`.

## Global Constraints

- **No `ecko-core`/`ecko-std` changes.** Everything lands in the `webkit` repo (`~/Development/ecko/webkit`).
- **Responses are `{ status, headers, body }` maps.** `headers` is a `{ name: value }` map (lowercase names). Bytes bodies pass through raw.
- **Middleware are `|req, next|` functions** (as `web.router`'s 2nd arg expects).
- **Error-code map keys are strings** (`"404"`, not `404`) - Ecko map keys are strings.
- **File helpers require the `fs:read` capability** when webkit is imported as a package; `webkit`'s `ecko.json` declares `["fs:read"]`.
- **Tests are offline & deterministic** via `ecko test` (discovers `*_test.ecko`); use `test.case(name, || { test.eq(actual, expected) })`.
- **Commits:** the repo owner runs `git commit` themselves. Commit steps below show the intended message/rhythm; stage the files and stop for the owner to commit.
- **Format gate:** run `ecko fmt --check main.ecko framework_test.ecko` before each commit; run `ecko fmt` to fix.

---

## File Structure

- `~/Development/ecko/webkit/main.ecko` - MODIFY: add imports (`std.http`, `std.fs`, `std.web`); add helpers (`content_type`, `file`, `static`, `cache_control`, `with_headers`(public), `cache`, `redirect`, `abort`, `json`, `html`, `text`, `app`, plus private `ext_of`, `handle_error`, `default_error_page`, `get_or`); extend `export {}`.
- `~/Development/ecko/webkit/framework_test.ecko` - CREATE: offline tests for the new helpers.
- `~/Development/ecko/webkit/ecko.json` - MODIFY: `"capabilities": ["fs:read"]`.
- `~/Development/ecko/webkit/framework_example.ecko` - CREATE (Task 5): an end-to-end app assembled with the framework, exercised offline.
- `~/Development/ecko/webkit/README.md` - MODIFY (Task 5): add a "Framework" section.
- `~/Development/ecko/website/splash/server.ecko` + `ecko.json` - MODIFY (Task 5): rebuild on webkit (real-world validation).

---

## Task 1: Content types & single-file serving

**Files:**
- Modify: `main.ecko` (imports; `ext_of`, `content_type`, `get_or`, `file`; exports)
- Modify: `ecko.json` (add `fs:read`)
- Test: `framework_test.ecko` (create)

**Interfaces:**
- Produces: `content_type(path) -> String`; `file(path, opts = empty_map()) -> { status, headers, body }` where `opts.cache` is an optional `cache-control` string; private `ext_of(path) -> String` (lowercased extension, no dot) and `get_or(m, k, d)`.

- [ ] **Step 1: Write the failing test**

Create `framework_test.ecko`:

```ecko
# Offline tests for the webkit framework core (ecko test).
import std.test
import std.fs
import "./main.ecko"

test.case("content_type maps by extension", || {
    test.eq(main.content_type("a/b/index.html"), "text/html; charset=utf-8")
    test.eq(main.content_type("style.css"), "text/css; charset=utf-8")
    test.eq(main.content_type("app.js"), "text/javascript; charset=utf-8")
    test.eq(main.content_type("logo.svg"), "image/svg+xml")
    test.eq(main.content_type("data"), "application/octet-stream")
})

test.case("file serves an existing file with content-type + cache", || {
    path = "/tmp/webkit_t1.html"
    fs.write(path, "<h1>hi</h1>")
    r = main.file(path, { cache: "public, max-age=60" })
    test.eq(r.status, 200)
    test.eq(get(r.headers, "content-type"), "text/html; charset=utf-8")
    test.eq(get(r.headers, "cache-control"), "public, max-age=60")
})

test.case("file 404s a missing path", || {
    test.eq(main.file("/tmp/does-not-exist-xyz.html").status, 404)
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ~/Development/ecko/webkit && ecko test framework_test.ecko`
Expected: FAIL - `content_type`/`file` undefined (unknown member on `main`).

- [ ] **Step 3: Implement in `main.ecko`**

Change the imports line at the top from:

```ecko
import std.hash
import std.random
```

to:

```ecko
import std.hash
import std.random
import std.http
import std.fs
import std.web
```

Add these helpers (near the top, after the `export` line):

```ecko
# get(m, k) with a default when the key is absent or null.
fn get_or(m, k, d) {
    v = get(m, k)
    if is_null(v) { d } else { v }
}

# The lowercased file extension of a path, without the dot ("" if none).
fn ext_of(path) {
    parts = split(string(path), ".")
    if len(parts) < 2 { "" } else { lower(last(parts)) }
}

# content_type(path) -> a MIME type string for the file's extension.
fn content_type(path) {
    types = {
        html: "text/html; charset=utf-8", htm: "text/html; charset=utf-8",
        css: "text/css; charset=utf-8",
        js: "text/javascript; charset=utf-8", mjs: "text/javascript; charset=utf-8",
        json: "application/json", xml: "application/xml",
        txt: "text/plain; charset=utf-8", md: "text/markdown; charset=utf-8",
        svg: "image/svg+xml", png: "image/png",
        jpg: "image/jpeg", jpeg: "image/jpeg", gif: "image/gif", webp: "image/webp",
        ico: "image/x-icon", woff: "font/woff", woff2: "font/woff2",
        wasm: "application/wasm", pdf: "application/pdf",
    }
    get_or(types, ext_of(path), "application/octet-stream")
}

# file(path, opts?) -> a response serving the file FRESH (read per call), with
# its content-type and an optional opts.cache ("cache-control"). 404 if absent.
fn file(path, opts = empty_map()) {
    unless fs.exists(path) {
        return http.response(404, "not found")
    }
    mut h = insert(empty_map(), "content-type", content_type(path))
    c = get(opts, "cache")
    unless is_null(c) { h = insert(h, "cache-control", c) }
    http.response(200, fs.read_bytes(path), h)
}
```

Extend the `export { ... }` line to include the new public names:

```ecko
export { escape, render, sign, unsign, cookie, parse_cookies, session_new, session_read, cors, security_headers, content_type, file }
```

- [ ] **Step 4: Add `fs:read` to `ecko.json`**

Change `"capabilities": []` to:

```json
  "capabilities": ["fs:read"],
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `ecko test framework_test.ecko` → Expected: 3 cases PASS.
Run: `ecko fmt --check main.ecko framework_test.ecko` → Expected: clean (run `ecko fmt` if not).

- [ ] **Step 6: Stage for commit**

```bash
git add main.ecko framework_test.ecko ecko.json
# owner commits, e.g.: feat(webkit): content_type + fresh single-file serving
```

---

## Task 2: Response helpers

**Files:**
- Modify: `main.ecko` (`with_headers` public, `cache`, `redirect`, `abort`, `json`, `html`, `text`; exports)
- Test: `framework_test.ecko`

**Interfaces:**
- Consumes: `http.json/html/text/response`.
- Produces: `with_headers(resp, map) -> resp`; `cache(resp, value) -> resp`; `redirect(url, status = 302) -> resp`; `abort(status, body = "")` (raises `{ kind:"http", status, message }`); `json(value, opts = empty_map())`, `html(body, opts = empty_map())`, `text(s, opts = empty_map())` (each honors `opts.cache`).

- [ ] **Step 1: Write the failing tests**

Append to `framework_test.ecko`:

```ecko
test.case("with_headers merges headers onto a response", || {
    r = main.with_headers({ status: 200, headers: empty_map(), body: "x" }, { "x-a": "1", "x-b": "2" })
    test.eq(get(r.headers, "x-a"), "1")
    test.eq(get(r.headers, "x-b"), "2")
})

test.case("cache sets cache-control", || {
    r = main.cache({ status: 200, headers: empty_map(), body: "x" }, "public, max-age=30")
    test.eq(get(r.headers, "cache-control"), "public, max-age=30")
})

test.case("redirect builds a 302 with a location header", || {
    r = main.redirect("/login")
    test.eq(r.status, 302)
    test.eq(get(r.headers, "location"), "/login")
})

test.case("html honors an opts.cache", || {
    r = main.html("<p>hi</p>", { cache: "no-store" })
    test.eq(r.status, 200)
    test.eq(get(r.headers, "content-type"), "text/html; charset=utf-8")
    test.eq(get(r.headers, "cache-control"), "no-store")
})

test.case("abort raises a structured http error", || {
    caught = try {
        main.abort(404, "nope")
        "no-error"
    } catch (e) {
        get(e, "kind") + ":" + string(get(e, "status"))
    }
    test.eq(caught, "http:404")
})
```

- [ ] **Step 2: Run to verify failure**

Run: `ecko test framework_test.ecko` → Expected: the 5 new cases FAIL (undefined members).

- [ ] **Step 3: Implement in `main.ecko`**

The private `with_header(resp, key, value)` already exists (used by `cors`). Add the public helpers below it:

```ecko
# with_headers(resp, map) -> resp with every name:value merged into headers.
fn with_headers(resp, hmap) {
    mut h = if is_null(get(resp, "headers")) { empty_map() } else { resp.headers }
    for (k, v) in hmap { h = insert(h, k, v) }
    insert(resp, "headers", h)
}

# cache(resp, value) -> resp with a cache-control header.
fn cache(resp, value) = with_header(resp, "cache-control", value)

# redirect(url, status?) -> a redirect response (302 by default).
fn redirect(url, status = 302) {
    { status: status, headers: insert(empty_map(), "location", url), body: "" }
}

# abort(status, body?) -> raises an http error the app error layer maps to a page.
fn abort(status, body = "") {
    error({ kind: "http", status: status, message: body })
}

# Response builders that also accept an opts.cache.
fn html(body, opts = empty_map()) {
    r = http.html(body)
    c = get(opts, "cache")
    if is_null(c) { r } else { cache(r, c) }
}

fn json(value, opts = empty_map()) {
    r = http.json(value)
    c = get(opts, "cache")
    if is_null(c) { r } else { cache(r, c) }
}

fn text(s, opts = empty_map()) {
    r = http.text(s)
    c = get(opts, "cache")
    if is_null(c) { r } else { cache(r, c) }
}
```

Extend `export { ... }` to add: `with_headers, cache, redirect, abort, html, json, text`.

- [ ] **Step 4: Run to verify pass**

Run: `ecko test framework_test.ecko` → Expected: all cases PASS.
Run: `ecko fmt --check main.ecko framework_test.ecko` → clean.

- [ ] **Step 5: Stage for commit**

```bash
git add main.ecko framework_test.ecko
# owner commits, e.g.: feat(webkit): response helpers (redirect/abort/cache/json/html/text)
```

---

## Task 3: Static passthrough & cache-control middleware

**Files:**
- Modify: `main.ecko` (`static`, `cache_control`; exports)
- Test: `framework_test.ecko`

**Interfaces:**
- Consumes: `web.static`, `cache` (Task 2), `with_header`.
- Produces: `static(prefix, dir) -> route` (passthrough to `web.static`); `cache_control(rules) -> middleware`, where `rules` is a list of `{ prefix, value }` and the first prefix that `req.path` starts with wins.

- [ ] **Step 1: Write the failing tests**

Append to `framework_test.ecko`:

```ecko
test.case("static returns a GET route for the prefix", || {
    r = main.static("/assets", "public")
    test.eq(get(r, "method"), "GET")
    test.eq(get(r, "path"), "/assets")
})

test.case("cache_control middleware stamps cache-control by path prefix", || {
    mw = main.cache_control([{ prefix: "/assets", value: "public, max-age=86400" }])
    next = |req| { status: 200, headers: empty_map(), body: "x" }
    hit = mw({ method: "GET", path: "/assets/logo.png", headers: empty_map() }, next)
    miss = mw({ method: "GET", path: "/", headers: empty_map() }, next)
    test.eq(get(hit.headers, "cache-control"), "public, max-age=86400")
    test.eq(get(miss.headers, "cache-control"), null)
})
```

- [ ] **Step 2: Run to verify failure**

Run: `ecko test framework_test.ecko` → Expected: 2 new cases FAIL.

- [ ] **Step 3: Implement in `main.ecko`**

```ecko
# static(prefix, dir) -> a route serving `dir` under `prefix` via the hardened
# native web.static (traversal-safe; content-types handled). Add caching with
# cache_control middleware.
fn static(prefix, dir) = web.static(prefix, dir)

# cache_control(rules) -> middleware that sets cache-control on the response for
# the first rule whose prefix matches req.path. rules: [{ prefix, value }].
fn cache_control(rules) {
    |req, next| {
        resp = next(req)
        path = string(get(req, "path"))
        mut out = resp
        mut done = false
        for rule in rules {
            if not done and starts_with(path, rule.prefix) {
                out = cache(resp, rule.value)
                done = true
            }
        }
        out
    }
}
```

Extend `export { ... }` to add: `static, cache_control`.

- [ ] **Step 4: Run to verify pass**

Run: `ecko test framework_test.ecko` → PASS. Then `ecko fmt --check main.ecko framework_test.ecko` → clean.

- [ ] **Step 5: Stage for commit**

```bash
git add main.ecko framework_test.ecko
# owner commits, e.g.: feat(webkit): static passthrough + cache_control middleware
```

---

## Task 4: App assembly & error handling

**Files:**
- Modify: `main.ecko` (`app`, private `handle_error`, `default_error_page`; exports)
- Test: `framework_test.ecko`

**Interfaces:**
- Consumes: `web.router`, `security_headers`, `cors`, `get_or` (Task 1).
- Produces: `app(spec) -> handler` (a `|req| -> resp`). `spec` keys (all optional): `routes` (list), `middleware` (list of `|req,next|`), `security` (bool → adds `security_headers()`), `cors` (opts → adds `cors(opts)`), `errors` (map `{ "404": |req|->resp, "500": |req,e|->resp, ... }`). Dispatch: a thrown `{kind:"http",status}` (from `abort`) → that status's handler; any other thrown value → `"500"`; a router 404 → `"404"`; missing handlers fall back to a plain default page.

- [ ] **Step 1: Write the failing tests**

Append to `framework_test.ecko`:

```ecko
test.case("app routes a request through web.router", || {
    a = main.app({ routes: [web.get("/", |req| main.text("home"))] })
    r = a({ method: "GET", path: "/", headers: empty_map(), params: empty_map() })
    test.eq(r.status, 200)
    test.eq(r.body, "home")
})

test.case("app maps a router 404 to a custom error page", || {
    a = main.app({
        routes: [web.get("/", |req| main.text("home"))],
        errors: { "404": |req| { status: 404, headers: empty_map(), body: "custom missing" } },
    })
    r = a({ method: "GET", path: "/nope", headers: empty_map(), params: empty_map() })
    test.eq(r.status, 404)
    test.eq(r.body, "custom missing")
})

test.case("abort inside a handler routes to that status page", || {
    a = main.app({
        routes: [web.get("/secret", |req| main.abort(403, "no"))],
        errors: { "403": |req, e| { status: 403, headers: empty_map(), body: "forbidden" } },
    })
    r = a({ method: "GET", path: "/secret", headers: empty_map(), params: empty_map() })
    test.eq(r.status, 403)
    test.eq(r.body, "forbidden")
})

test.case("a thrown bug becomes a 500", || {
    a = main.app({ routes: [web.get("/boom", |req| error("kaboom"))] })
    r = a({ method: "GET", path: "/boom", headers: empty_map(), params: empty_map() })
    test.eq(r.status, 500)
})

test.case("security + cors options add middleware", || {
    a = main.app({
        routes: [web.get("/", |req| main.text("ok"))],
        security: true,
        cors: { origin: "*" },
    })
    r = a({ method: "GET", path: "/", headers: empty_map(), params: empty_map() })
    test.eq(get(r.headers, "x-content-type-options"), "nosniff")
    test.eq(get(r.headers, "access-control-allow-origin"), "*")
})
```

- [ ] **Step 2: Run to verify failure**

Run: `ecko test framework_test.ecko` → Expected: 5 new cases FAIL (`app` undefined).

- [ ] **Step 3: Implement in `main.ecko`**

```ecko
# A plain fallback error response when no custom handler is registered.
fn default_error_page(status) {
    label = if status == 404 { "Not Found" } else if status == 500 { "Internal Server Error" } else { "Error" }
    {
        status: status,
        headers: insert(empty_map(), "content-type", "text/plain; charset=utf-8"),
        body: string(status) + " " + label,
    }
}

# Map a caught error to a response: an http error (from abort) uses its status;
# anything else is a bug -> 500. Uses the errors map ({ "<status>": handler }).
fn handle_error(errors, req, e) {
    status = if get(e, "kind") == "http" { get(e, "status") } else { 500 }
    handler = get(errors, string(status))
    if is_null(handler) { default_error_page(status) } else { handler(req, e) }
}

# app(spec) -> a handler for http.serve. Wires routes + middleware + error pages.
fn app(spec) {
    routes = get_or(spec, "routes", [])
    errors = get_or(spec, "errors", empty_map())
    mut mw = []
    if get(spec, "security") == true { mw = push(mw, security_headers()) }
    cors_opts = get(spec, "cors")
    unless is_null(cors_opts) { mw = push(mw, cors(cors_opts)) }
    for m in get_or(spec, "middleware", []) { mw = push(mw, m) }
    router = web.router(routes, mw)
    |req| {
        result = try {
            router(req)
        } catch (e) {
            return handle_error(errors, req, e)
        }
        if get(result, "status") == 404 {
            h = get(errors, "404")
            if is_null(h) { result } else { h(req) }
        } else {
            result
        }
    }
}
```

Extend `export { ... }` to add: `app`.

- [ ] **Step 4: Run to verify pass**

Run: `ecko test framework_test.ecko` → all PASS. Then `ecko fmt --check main.ecko framework_test.ecko` → clean.
Also run the full suite: `ecko test` → the original 13 cases + all framework cases PASS.

- [ ] **Step 5: Stage for commit**

```bash
git add main.ecko framework_test.ecko
# owner commits, e.g.: feat(webkit): app() assembly + error handling
```

---

## Task 5: Acceptance - example + splash rebuild + docs

**Files:**
- Create: `framework_example.ecko`
- Modify: `README.md`
- Modify: `~/Development/ecko/website/splash/server.ecko`, `~/Development/ecko/website/splash/ecko.json`

**Interfaces:**
- Consumes: all of Tasks 1-4.

- [ ] **Step 1: Write an offline end-to-end example that also acts as a smoke test**

Create `framework_example.ecko`:

```ecko
# webkit framework demo, offline (ecko example.ecko / ecko framework_example.ecko).
import std.web
import "./main.ecko"

fn home(req) = main.html("<h1>Home</h1>")

app = main.app({
    routes: [
        web.get("/", home),
        main.static("/assets", "public"),
    ],
    middleware: [main.cache_control([{ prefix: "/assets", value: "public, max-age=86400" }])],
    security: true,
    errors: { "404": |req| { status: 404, headers: empty_map(), body: "Not found: " + req.path } },
})

# Exercise it offline by calling the handler directly.
home_resp = app({ method: "GET", path: "/", headers: empty_map(), params: empty_map() })
print("GET / -> " + string(home_resp.status) + " " + get(home_resp.headers, "content-type"))
print("secure? " + get(home_resp.headers, "x-content-type-options"))

miss = app({ method: "GET", path: "/nope", headers: empty_map(), params: empty_map() })
print("GET /nope -> " + string(miss.status) + " " + miss.body)
```

- [ ] **Step 2: Run it - verify deterministic output**

Run: `cd ~/Development/ecko/webkit && ecko framework_example.ecko`
Expected exactly:

```
GET / -> 200 text/html; charset=utf-8
secure? nosniff
GET /nope -> 404 Not found: /nope
```

- [ ] **Step 3: Add a README "Framework" section**

Append to `README.md` (after the Middleware section):

````markdown
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
    errors: { "404": |req| webkit.html("<h1>Not found</h1>") },
})
http.serve(8080, app)
```

- `webkit.file(path, { cache })` serves one file fresh (no read-once staleness).
- `webkit.redirect(url)`, `webkit.abort(404)`, `webkit.json/html/text(v, { cache })`,
  `webkit.with_headers`, `webkit.cache`, `webkit.content_type`.
- **Escaping is explicit** in native templates - call `webkit.escape(x)` (alias
  `webkit.e`) on untrusted data. A forgotten escape is an XSS hole.

Static and file helpers read the disk, so grant `fs:read` to webkit in your
app's `ecko.json`: `"dependencies": { "webkit": { "grant": ["fs:read"] } }`.
````

- [ ] **Step 4: Rebuild the splash server on the framework (real-world validation)**

Rewrite `~/Development/ecko/website/splash/server.ecko`:

```ecko
# Ecko splash - coming-soon page, on the webkit framework.
#   cd website/splash && ecko server.ecko   # http://localhost:8082
import std.http
import std.web
import std.os
import webkit

port = int(os.env_or("PORT", "8082"))

fn page(req) = webkit.file("public/index.html", { cache: "public, max-age=60" })

app = webkit.app({
    routes: [
        web.get("/", page),
        web.get("/healthz", |req| webkit.text("ok")),
        web.get("/health", |req| webkit.text("ok")),
        webkit.static("/icons", "public/icons"),
        webkit.static("/brand", "public/brand"),
    ],
    middleware: [webkit.cache_control([
        { prefix: "/icons", value: "public, max-age=86400" },
        { prefix: "/brand", value: "public, max-age=86400" },
    ])],
    security: true,
})

print("ecko splash listening on :" + string(port))
http.serve(port, app)
```

Add webkit to `~/Development/ecko/website/splash/ecko.json` dependencies with the grant (create the file if absent):

```json
{
  "name": "splash",
  "version": "0.1.0",
  "entrypoint": "server.ecko",
  "dependencies": { "webkit": { "source": "local", "sha256": "-", "grant": ["fs:read"] } }
}
```

Vendor webkit locally and run:

```bash
cd ~/Development/ecko/website/splash
ecko add ../../webkit          # or: ecko install, if a lockfile exists
ecko server.ecko &             # starts on :8082
curl -s localhost:8082/ | head -1        # serves public/index.html (fresh)
curl -sI localhost:8082/icons/favicon.ico | grep -i cache-control   # max-age=86400
# edit public/index.html, curl / again -> new content immediately (fresh read)
kill %1
```

Expected: `/` serves the page with `cache-control: public, max-age=60`; `/icons/*` carry `max-age=86400`; editing `public/index.html` shows on the next request (no restart) because `webkit.file` reads fresh.

- [ ] **Step 5: Stage for commit**

```bash
# in webkit repo:
git add framework_example.ecko README.md
# owner commits, e.g.: feat(webkit): framework example + docs
# in website repo (separate commit):
#   git add website/splash/server.ecko website/splash/ecko.json vendor/ ecko.lock
```

---

## Self-Review

**Spec coverage (Phase 1 items):** `content_type` ✓T1, `file` ✓T1, `static` ✓T3, `cache_control` ✓T3, `redirect`/`abort`/`with_headers`/`cache`/`json`/`html`/`text` ✓T2, `app` + error handling ✓T4, `fs:read` capability ✓T1, splash rebuild ✓T5. Phase 2 (request accessors, `blueprint`, `url_for`) and Phase 3 (`flash`, `validate`, `e` alias, remove `render`, escaping guide) are **out of scope for this plan** - separate plans.

**Placeholder scan:** none - every step has concrete code/commands.

**Type consistency:** response shape `{ status, headers, body }` used consistently; `errors` keyed by string status (`"404"`) in both `app` (T4) and its tests; `cache_control` rules are `{ prefix, value }` in impl (T3) and example/splash (T5); `handle_error` reads `e.kind == "http"` / `e.status`, matching `abort`'s `{ kind:"http", status, message }` (T2). `get_or` defined in T1 and reused in T4.

**Note for the implementer:** default params (`opts = empty_map()`, `status = 302`) and map iteration destructuring (`for (k, v) in hmap`) are used - both are supported Ecko features; if `ecko test` reports a parse error on a default like `empty_map()`, fall back to a required `opts` argument and pass `empty_map()` at call sites (adjust the tests accordingly).
