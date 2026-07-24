# webkit Framework - Phase 2 (Requests & Routing) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Add request-access helpers, route-group blueprints, and `url_for` to the `webkit` framework (Phase 1 already shipped `app`/static/responses/errors).

**Architecture:** More pure-Ecko helpers in `webkit/main.ecko` over `std.web`/`std.http`/`std.encoding`. `blueprint` returns a list of prefixed route entries; `webkit.app` is extended to flatten those lists into `web.router`'s flat route list. No `ecko-core`/`ecko-std` changes.

**Tech Stack:** Ecko, `std.test`, `std.web`, `std.http`, `std.encoding`.

## Global Constraints

- **No `ecko-core`/`ecko-std` changes.** Work only in `~/Development/ecko/webkit` (branch `feature/framework-phase1`, which now holds Phase 1).
- **Ecko map literals reject string-literal keys** (`{ "a-b": 1 }` does NOT parse) - build such maps with `insert(empty_map(), "key", v)`; identifier keys (`{ status: 200 }`) are fine. (Learned in Phase 1.)
- **Map iteration order is non-deterministic** - `sort` any list built by iterating a map before using it in output (so tests are stable).
- Responses are `{ status, headers, body }` maps; middleware are `|req, next|`; routes are maps like `{ method, path, handler }` / `{ method, path, static_dir }`.
- Use the freshly-built binary for all commands: `/home/sean/Development/ecko/core/target/debug/ecko`.
- **Stage only (`git add`); the owner commits.** Run `.../ecko fmt --check main.ecko framework_test.ecko` before staging.
- Helpers here are cap-free except where they call `std.web`/`std.http` (already covered by webkit's `["fs:read","net"]`).
- Private helpers used below: `get_or(m, k, d)` already exists in `main.ecko` from Phase 1.

---

## Task 1: Request-access helpers

**Files:**
- Modify: `main.ecko` (`query`, `query_int`, `form`, `json_body`, `cookies`, `session`, private `to_int_or`, `cookie_header`; exports)
- Test: `framework_test.ecko` (append)

**Interfaces:**
- Consumes: `get_or` (Phase 1), `parse_cookies`, `session_read` (existing webkit).
- Produces: `query(req, key, default = null)`, `query_int(req, key, default = 0)`, `form(req, key, default = null)`, `json_body(req)`, `cookies(req) -> map`, `session(req, secret) -> id|null`.

- [ ] **Step 1: Write the failing tests** - append to `framework_test.ecko`:

```ecko
test.case("query reads a parsed query param with a default", || {
    req = { params: insert(empty_map(), "q", "hi"), form: empty_map(), headers: empty_map() }
    test.eq(main.query(req, "q"), "hi")
    test.eq(main.query(req, "missing", "d"), "d")
})

test.case("query_int parses or falls back", || {
    req = { params: insert(insert(empty_map(), "n", "42"), "bad", "x"), headers: empty_map() }
    test.eq(main.query_int(req, "n"), 42)
    test.eq(main.query_int(req, "bad", -1), -1)
    test.eq(main.query_int(req, "absent", 7), 7)
})

test.case("form reads a form field with a default", || {
    req = { form: insert(empty_map(), "email", "a@b.c"), params: empty_map(), headers: empty_map() }
    test.eq(main.form(req, "email"), "a@b.c")
    test.eq(main.form(req, "nope", ""), "")
})

test.case("cookies parses the request Cookie header", || {
    req = { headers: insert(empty_map(), "cookie", "sid=abc; theme=dark") }
    c = main.cookies(req)
    test.eq(get(c, "sid"), "abc")
    test.eq(get(c, "theme"), "dark")
})

test.case("session reads a signed sid cookie", || {
    signed = main.sign("user9", "sek")
    req = { headers: insert(empty_map(), "cookie", "sid=" + signed) }
    test.eq(main.session(req, "sek"), "user9")
    test.eq(main.session({ headers: empty_map() }, "sek"), null)
})
```

- [ ] **Step 2: Run to verify failure** - `cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test framework_test.ecko` → new cases FAIL (undefined members).

- [ ] **Step 3: Implement in `main.ecko`** (add a `# --- request access ---` section):

```ecko
fn query(req, key, default = null) {
    get_or(get_or(req, "params", empty_map()), key, default)
}

fn to_int_or(v, d) {
    if is_null(v) { d } else { try { int(v) } catch (e) { d } }
}

fn query_int(req, key, default = 0) = to_int_or(query(req, key, null), default)

fn form(req, key, default = null) {
    get_or(get_or(req, "form", empty_map()), key, default)
}

fn json_body(req) = get(req, "json")

fn cookie_header(req) = get_or(get_or(req, "headers", empty_map()), "cookie", null)

fn cookies(req) = parse_cookies(cookie_header(req))

fn session(req, secret) = session_read(cookie_header(req), secret)
```

Add to `export { ... }`: `query, query_int, form, json_body, cookies, session`.

- [ ] **Step 4: Run to verify pass** - `.../ecko test framework_test.ecko` PASS; `.../ecko fmt --check main.ecko framework_test.ecko` clean.

- [ ] **Step 5: Stage** - `git add main.ecko framework_test.ecko` (owner commits: `feat(webkit): request-access helpers`).

---

## Task 2: Blueprints (route groups) + app flattening

**Files:**
- Modify: `main.ecko` (`compose_mw`, `blueprint`; extend `app` to flatten route lists; exports)
- Test: `framework_test.ecko` (append)

**Interfaces:**
- Consumes: route maps `{ method, path, handler }`; `type_of`, `reduce`, `reverse`.
- Produces: `blueprint(prefix, routes, mw = []) -> list of prefixed route maps` (group `mw` wraps each route's handler, outermost first). `app` now accepts blueprint lists inside `routes` and flattens them one level.

- [ ] **Step 1: Write the failing tests** - append:

```ecko
test.case("blueprint prefixes each route's path", || {
    bp = main.blueprint("/api", [web.get("/users", |req| main.text("u")), web.get("/items", |req| main.text("i"))])
    test.eq(get(bp[0], "path"), "/api/users")
    test.eq(get(bp[1], "path"), "/api/items")
})

test.case("blueprint group middleware wraps handlers outermost-first", || {
    tag = |label| |req, next| insert(next(req), label, true)
    bp = main.blueprint("/api", [web.get("/x", |req| { status: 200, headers: empty_map(), body: "x" })], [tag("a")])
    h = get(bp[0], "handler")
    r = h({ path: "/api/x" })
    test.eq(get(r, "a"), true)
})

test.case("app dispatches a blueprint's prefixed routes", || {
    a = main.app({ routes: [main.blueprint("/api", [web.get("/ping", |req| main.text("pong"))])] })
    r = a({ method: "GET", path: "/api/ping", headers: empty_map(), params: empty_map() })
    test.eq(r.status, 200)
    test.eq(r.body, "pong")
})
```

- [ ] **Step 2: Run to verify failure** - `.../ecko test framework_test.ecko` → 3 new cases FAIL.

- [ ] **Step 3a: Add `compose_mw` + `blueprint` in `main.ecko`:**

```ecko
# compose_mw(handler, mw) -> handler wrapped by the middleware list, outermost
# first. Uses reduce over the reversed list so each layer captures the next by
# value (no mutable-loop closure trap).
fn compose_mw(handler, mw) = reduce(reverse(mw), |acc, m| (|req| m(req, acc)), handler)

# blueprint(prefix, routes, mw?) -> a list of routes with `prefix` prepended to
# each path and `mw` (a list of |req,next|) wrapped around each handler.
fn blueprint(prefix, routes, mw = []) {
    mut out = []
    for r in routes {
        mut base = insert(r, "path", string(prefix) + string(get(r, "path")))
        h = get(r, "handler")
        if not is_null(h) and len(mw) > 0 {
            base = insert(base, "handler", compose_mw(h, mw))
        }
        out = push(out, base)
    }
    out
}
```

- [ ] **Step 3b: Extend `app` to flatten blueprint lists.** In `fn app(spec)`, replace the line `routes = get_or(spec, "routes", [])` and the `router = web.router(routes, mw)` call so routes are flattened first. The updated `app` reads:

```ecko
fn app(spec) {
    errors = get_or(spec, "errors", empty_map())
    mut mw = []
    if get(spec, "security") == true { mw = push(mw, security_headers()) }
    cors_opts = get(spec, "cors")
    unless is_null(cors_opts) { mw = push(mw, cors(cors_opts)) }
    for m in get_or(spec, "middleware", []) {
        mw = push(mw, m)
    }
    # Flatten one level so a blueprint (a list of routes) splices in.
    mut flat = []
    for r in get_or(spec, "routes", []) {
        if type_of(r) == "list" {
            for br in r {
                flat = push(flat, br)
            }
        } else {
            flat = push(flat, r)
        }
    }
    router = web.router(flat, mw)
    |req| {
        result = try { router(req) } catch (e) { return handle_error(errors, req, e) }
        if get(result, "status") == 404 {
            h = get(errors, "404")
            if is_null(h) { result } else { h(req) }
        } else { result }
    }
}
```

Add to `export { ... }`: `blueprint`.

- [ ] **Step 4: Run to verify pass** - `.../ecko test framework_test.ecko` PASS (incl. the Phase 1 `app` cases still green); `.../ecko fmt --check main.ecko framework_test.ecko` clean.

- [ ] **Step 5: Stage** - `git add main.ecko framework_test.ecko` (owner commits: `feat(webkit): blueprints + app route flattening`).

---

## Task 3: url_for

**Files:**
- Modify: `main.ecko` (add `import std.encoding`; `url_for`; exports)
- Test: `framework_test.ecko` (append)

**Interfaces:**
- Consumes: `std.encoding` (`url_encode`), `sort`, `replace`, `join`.
- Produces: `url_for(pattern, params = empty_map(), query = empty_map()) -> string` - fills `:name` segments from `params`, appends a URL-encoded, **sorted** query string.

- [ ] **Step 1: Write the failing tests** - append:

```ecko
test.case("url_for fills path params", || {
    test.eq(main.url_for("/user/:id", insert(empty_map(), "id", 5)), "/user/5")
})

test.case("url_for appends a sorted, encoded query", || {
    q = insert(insert(empty_map(), "b", "2"), "a", "x y")
    test.eq(main.url_for("/s", empty_map(), q), "/s?a=x%20y&b=2")
})

test.case("url_for with no query is just the path", || {
    test.eq(main.url_for("/health"), "/health")
})
```

- [ ] **Step 2: Run to verify failure** - `.../ecko test framework_test.ecko` → 3 new cases FAIL.

- [ ] **Step 3: Implement.** Add `import std.encoding` to the imports at the top of `main.ecko`, then add:

```ecko
# url_for(pattern, params?, query?) -> a URL: fills :name segments, then appends
# a sorted, URL-encoded query string.
fn url_for(pattern, params = empty_map(), query = empty_map()) {
    mut path = string(pattern)
    for (k, v) in params {
        path = replace(path, ":" + k, string(v))
    }
    mut parts = []
    for (k, v) in query {
        parts = push(parts, encoding.url_encode(k) + "=" + encoding.url_encode(string(v)))
    }
    parts = sort(parts)
    if len(parts) > 0 { path + "?" + join(parts, "&") } else { path }
}
```

Add to `export { ... }`: `url_for`.

- [ ] **Step 4: Run to verify pass** - `.../ecko test framework_test.ecko` PASS. Then the FULL suite: `cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test` → all Phase 1 + Phase 2 cases pass. `.../ecko fmt --check main.ecko framework_test.ecko` clean.

- [ ] **Step 5: Stage** - `git add main.ecko framework_test.ecko` (owner commits: `feat(webkit): url_for`).

---

## Task 4: Docs

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a "Requests & routing" subsection** under the Framework section:

````markdown
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
````

- [ ] **Step 2: Verify** - `.../ecko fmt --check README.md` is not needed (markdown), but re-run `.../ecko test` to confirm nothing regressed. Stage: `git add README.md` (owner commits: `docs(webkit): request/routing helpers`).

---

## Self-Review

**Spec coverage (Phase 2):** request accessors `query`/`query_int`/`form`/`json_body`/`cookies`/`session` ✓T1; `blueprint` (prefix + group mw) ✓T2 with `app` flattening ✓T2; `url_for` (pattern-based, sorted/encoded query) ✓T3; docs ✓T4. Phase 3 (`flash`, `validate`, `e` alias, remove `render`, escaping guide) is out of scope - a separate plan.

**Placeholder scan:** none - every step has verified code.

**Type consistency:** `get_or(m, k, d)` reused from Phase 1; route maps use `path`/`handler` keys consistently in `blueprint` and `app`; `compose_mw(handler, mw)` signature matches its call in `blueprint`; `to_int_or(v, d)` defined in T1 and used by `query_int`. All four idioms (`type_of` == "list", `reduce`/`reverse` middleware compose, sorted `url_for`, `try`-guarded `int`) were verified against the runtime before writing.
