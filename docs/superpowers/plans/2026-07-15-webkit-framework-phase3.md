# webkit Framework - Phase 3 (State, Forms & Templating) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development or executing-plans. Checkbox steps.

**Goal:** Finish the framework - flash messages, form validation, the `e` escaping alias, and unify on native templates by removing the old `{{ }}` `render` engine (with the escaping guide).

**Architecture:** Pure-Ecko additions to `webkit/main.ecko` over `std.encoding`/`std.re` and the global `json_encode`/`json_decode`. Flash rides a signed, url-encoded `_flash` cookie. No `ecko-core`/`ecko-std` changes.

**Tech Stack:** Ecko, `std.test`, `std.encoding`, `std.re`, existing webkit `sign`/`unsign`/`cookie`/`cookies`.

## Global Constraints

- **No `ecko-core`/`ecko-std` changes.** Work only in `~/Development/ecko/webkit` (branch `feature/framework-phase1`).
- **Ecko map literals reject string-literal keys** - build maps with `insert(empty_map(), "key", v)`; identifier keys (`{ status: 200 }`) are fine.
- **Map iteration order is non-deterministic** - `sort` any list built from map iteration used in output/assertions.
- **Cookie values must be url-encoded** (signed values contain JSON/punctuation): store `encoding.url_encode(sign(...))`, read `unsign(encoding.url_decode(...))`.
- **Removing `render` is breaking** - also delete its tests in `webkit_test.ecko` and its use in `example.ecko`.
- Use `/home/sean/Development/ecko/core/target/debug/ecko` for all commands. **Stage only; the owner commits.** `fmt --check` before staging.
- Existing helpers to reuse: `escape`, `sign`, `unsign`, `cookie`, `with_header`, `cookies` (Phase 2), `get_or` (Phase 1). Globals: `json_encode`, `json_decode`.

---

## Task 1: `e` alias + remove the old `render` engine

**Files:**
- Modify: `main.ecko` (add `e`; delete `render` and `lookup`; exports)
- Modify: `webkit_test.ecko` (delete the 4 `render` tests; add an `e` test)
- Modify: `example.ecko` (replace the `render` demo with a native template + `e`)

**Interfaces:**
- Produces: `e(s) -> string` (alias of `escape`). Removes: `render`, `lookup`.

- [ ] **Step 1: Add the failing `e` test** - in `webkit_test.ecko`, DELETE the four cases whose names start with `render` and the case `"triple braces opt out..."`, and add:

```ecko
test.case("e is an alias for escape", || {
    test.eq(main.e("<b>&"), "&lt;b&gt;&amp;")
})
```

- [ ] **Step 2: Run to verify** - `cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test webkit_test.ecko` → the `e` case FAILS (undefined `e`); the deleted render cases are gone.

- [ ] **Step 3: Implement in `main.ecko`** - delete `fn render(tmpl, data) { ... }` and `fn lookup(data, key) { ... }` entirely. Add next to `escape`:

```ecko
# e(s) - alias of escape(), for terse use inside native templates: {e(name)}.
fn e(s) = escape(s)
```

Update `export { ... }`: remove `render`, add `e`.

- [ ] **Step 4: Update `example.ecko`** - replace the templating demo (the `page = main.render(...)` block and its `print`) with:

```ecko
# 1. Native templates + explicit escaping (webkit.e on untrusted data).
template page(title, note) = """<h1>{title}</h1><p>{note}</p>"""
rendered = page("Welcome", main.e("<script>alert(1)</script>"))
print("rendered: " + rendered)
```

- [ ] **Step 5: Run to verify pass** - `.../ecko test webkit_test.ecko` PASS; `.../ecko example.ecko` runs; `.../ecko fmt --check main.ecko webkit_test.ecko example.ecko` clean.

- [ ] **Step 6: Stage** - `git add main.ecko webkit_test.ecko example.ecko` (owner commits: `feat(webkit)!: e() alias; remove {{ }} render (use native templates)`).

---

## Task 2: Flash messages

**Files:**
- Modify: `main.ecko` (add `import std.encoding`; `flash`, `flashes`, `clear_flash`, private `flash_pending`; exports)
- Test: `framework_test.ecko` (append)

**Interfaces:**
- Consumes: `encoding.url_encode`/`url_decode`, `json_encode`/`json_decode`, `sign`/`unsign`, `cookie`, `with_header`, `cookies`.
- Produces: `flash(resp, message, secret) -> resp` (appends `message` to a signed `_flash` cookie); `flashes(req, secret) -> list` (messages, or `[]`); `clear_flash(resp) -> resp` (expires `_flash`).

- [ ] **Step 1: Write the failing tests** - append to `framework_test.ecko`:

```ecko
test.case("flash then flashes round-trips messages in order", || {
    r1 = main.flash({ status: 200, headers: empty_map(), body: "" }, "Saved!", "sek")
    r2 = main.flash(r1, "And again", "sek")
    val = split(split(get(r2.headers, "set-cookie"), ";")[0], "=")[1]
    req = { headers: insert(empty_map(), "cookie", "_flash=" + val) }
    test.eq(main.flashes(req, "sek"), ["Saved!", "And again"])
})

test.case("flashes is empty with no cookie or a bad secret", || {
    test.eq(main.flashes({ headers: empty_map() }, "sek"), [])
    r = main.flash({ status: 200, headers: empty_map(), body: "" }, "x", "sek")
    val = split(split(get(r.headers, "set-cookie"), ";")[0], "=")[1]
    req = { headers: insert(empty_map(), "cookie", "_flash=" + val) }
    test.eq(main.flashes(req, "wrong"), [])
})

test.case("clear_flash expires the cookie", || {
    r = main.clear_flash({ status: 200, headers: empty_map(), body: "" })
    test.eq(contains(get(r.headers, "set-cookie"), "Max-Age=0"), true)
})
```

- [ ] **Step 2: Run to verify failure** - `.../ecko test framework_test.ecko` → 3 new cases FAIL.

- [ ] **Step 3: Implement in `main.ecko`** - add `import std.encoding` to the top imports, then a `# --- flash messages ---` section:

```ecko
# Read the flash list already staged on THIS response's set-cookie (so repeated
# flash() calls append instead of overwrite). [] if none/invalid.
fn flash_pending(resp, secret) {
    sc = get(get_or(resp, "headers", empty_map()), "set-cookie")
    if is_null(sc) { return [] }
    kv = split(split(sc, ";")[0], "=")
    if len(kv) < 2 or kv[0] != "_flash" { return [] }
    v = unsign(encoding.url_decode(kv[1]), secret)
    if is_null(v) { [] } else { json_decode(v) }
}

# flash(resp, message, secret) -> resp with `message` appended to the signed
# _flash cookie (read on the next request with flashes()).
fn flash(resp, message, secret) {
    updated = push(flash_pending(resp, secret), message)
    signed = encoding.url_encode(sign(json_encode(updated), secret))
    with_header(resp, "set-cookie", cookie("_flash", signed, { path: "/", http_only: true, same_site: "Lax" }))
}

# flashes(req, secret) -> the list of flashed messages ([] if none/invalid).
fn flashes(req, secret) {
    raw = get(cookies(req), "_flash")
    if is_null(raw) { return [] }
    v = unsign(encoding.url_decode(raw), secret)
    if is_null(v) { [] } else { json_decode(v) }
}

# clear_flash(resp) -> resp that expires the _flash cookie.
fn clear_flash(resp) {
    with_header(resp, "set-cookie", cookie("_flash", "", { path: "/", max_age: 0 }))
}
```

Add to `export { ... }`: `flash, flashes, clear_flash`.

- [ ] **Step 4: Run to verify pass** - `.../ecko test framework_test.ecko` PASS; `.../ecko fmt --check main.ecko framework_test.ecko` clean.

- [ ] **Step 5: Stage** - `git add main.ecko framework_test.ecko` (owner commits: `feat(webkit): flash messages`).

---

## Task 3: Form validation

**Files:**
- Modify: `main.ecko` (add `import std.re`; `validate`, private `validate_field`; exports)
- Test: `framework_test.ecko` (append)

**Interfaces:**
- Consumes: `std.re` (`re.test`), `get_or`, global `int`/`len`/`starts_with`.
- Produces: `validate(form, schema) -> { valid, errors, values }`. `schema` is `{ field: { required?, type?, min?, max?, pattern? } }` (build with `insert`). `type` ∈ `int`/`email`/`url`/`string`; `min`/`max` compare the int value (type int) or string length otherwise; `pattern` is a regex string. `values` holds coerced values for valid fields; `errors` is `{ field: message }`.

- [ ] **Step 1: Write the failing tests** - append:

```ecko
test.case("validate flags a missing required field", || {
    schema = insert(empty_map(), "email", insert(empty_map(), "required", true))
    r = main.validate(empty_map(), schema)
    test.eq(r.valid, false)
    test.eq(get(r.errors, "email"), "required")
})

test.case("validate coerces an int and enforces min", || {
    schema = insert(empty_map(), "age", insert(insert(empty_map(), "type", "int"), "min", 18))
    ok = main.validate(insert(empty_map(), "age", "21"), schema)
    test.eq(ok.valid, true)
    test.eq(get(ok.values, "age"), 21)
    test.eq(main.validate(insert(empty_map(), "age", "5"), schema).valid, false)
    test.eq(main.validate(insert(empty_map(), "age", "x"), schema).valid, false)
})

test.case("validate checks email format", || {
    schema = insert(empty_map(), "email", insert(empty_map(), "type", "email"))
    test.eq(main.validate(insert(empty_map(), "email", "a@b.co"), schema).valid, true)
    test.eq(main.validate(insert(empty_map(), "email", "nope"), schema).valid, false)
})
```

- [ ] **Step 2: Run to verify failure** - `.../ecko test framework_test.ecko` → 3 new cases FAIL.

- [ ] **Step 3: Implement in `main.ecko`** - add `import std.re` to the top imports, then:

```ecko
# validate_field(raw, rules) -> { value, error }: coerce by type, then apply
# min/max (numeric for int, else length) and an optional regex pattern.
fn validate_field(raw, rules) {
    t = get_or(rules, "type", "string")
    mut value = raw
    mut err = null
    if t == "int" {
        n = try { int(raw) } catch (ex) { null }
        if is_null(n) { err = "must be an integer" } else { value = n }
    } else if t == "email" {
        unless re.test("^[^@ ]+@[^@ ]+\\.[^@ ]+$", raw) { err = "must be an email" }
    } else if t == "url" {
        unless starts_with(raw, "http://") or starts_with(raw, "https://") { err = "must be a URL" }
    }
    if is_null(err) {
        measure = if t == "int" { value } else { len(raw) }
        mn = get(rules, "min")
        unless is_null(mn) { if measure < mn { err = "too small" } }
        mx = get(rules, "max")
        if is_null(err) { unless is_null(mx) { if measure > mx { err = "too large" } } }
        pat = get(rules, "pattern")
        if is_null(err) { unless is_null(pat) { unless re.test(pat, raw) { err = "invalid format" } } }
    }
    { value: value, error: err }
}

# validate(form, schema) -> { valid, errors, values }.
fn validate(form, schema) {
    mut errors = empty_map()
    mut values = empty_map()
    for (field, rules) in schema {
        raw = get(form, field)
        present = not is_null(raw) and string(raw) != ""
        if get(rules, "required") == true and not present {
            errors = insert(errors, field, "required")
        } else if present {
            checked = validate_field(string(raw), rules)
            if is_null(get(checked, "error")) {
                values = insert(values, field, checked.value)
            } else {
                errors = insert(errors, field, checked.error)
            }
        }
    }
    { valid: len(keys(errors)) == 0, errors: errors, values: values }
}
```

Add to `export { ... }`: `validate`.

- [ ] **Step 4: Run to verify pass** - `.../ecko test framework_test.ecko` PASS. Then FULL suite `cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test` → all cases pass. `fmt --check main.ecko framework_test.ecko` clean.

- [ ] **Step 5: Stage** - `git add main.ecko framework_test.ecko` (owner commits: `feat(webkit): form validation`).

---

## Task 4: Templating & escaping guide

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a "Templating & escaping" section** under the Framework docs:

````markdown
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
````

- [ ] **Step 2: Update the Testing count** in `README.md` to the new total from `.../ecko test`, and stage - `git add README.md` (owner commits: `docs(webkit): templating/escaping + flash/forms guide`).

---

## Self-Review

**Spec coverage (Phase 3):** `e` alias ✓T1; remove `render` ✓T1 (+ tests/example); `flash`/`flashes`/`clear_flash` ✓T2; `validate` ✓T3; escaping + templating guide ✓T4. This completes the spec's phased scope.

**Placeholder scan:** none - every step has verified code.

**Type consistency:** `sign`/`unsign`/`cookie`/`cookies`/`with_header`/`get_or` reused with their existing signatures; flash stores `url_encode(sign(json_encode(list)))` and reads the inverse consistently in `flash`/`flash_pending`/`flashes`; `validate_field(raw, rules)` signature matches its call in `validate`. The flash round-trip, cookie-value recovery, map-of-maps iteration, and `re.test` were all verified against the runtime before writing.
