# webkit Framework - Phase 2 (Requests & Routing) - Report

Plan executed: `docs/superpowers/plans/2026-07-15-webkit-framework-phase2.md`, Tasks 1-4, in order, TDD (write failing test → run → implement → run → fmt → stage).

Binary used for all commands: `/home/sean/Development/ecko/core/target/debug/ecko` (repo `~/Development/ecko/webkit`, branch `feature/framework-phase1`).

Baseline before Phase 2 work: `framework_test.ecko` had 15 passing cases (Phase 1, already staged).

---

## Task 1 - Request-access helpers

**Test command:** `cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test framework_test.ecko`

- Step 1: Appended the 5 test cases from the plan (`query`, `query_int`, `form`, `cookies`, `session`) to `framework_test.ecko`.
- Step 2: Ran → confirmed failure: `1 file(s), 20 case(s): 15 passed, 5 failed` (all 5 new cases failed with `Module has no export '<name>'`).
- Step 3: Implemented `query`, `to_int_or`, `query_int`, `form`, `json_body`, `cookie_header`, `cookies`, `session` in `main.ecko` exactly as given, plus the export-list addition.
- Step 4: Ran → **1 failure**: `session reads a signed sid cookie` failed with `Runtime error: Can't reassign immutable variable 'cookies' (declared with let/const; use 'mut' to allow reassignment)`.

**Deviation (required fix, not in the plan text):** the plan's new top-level `cookies` function collided with a *local variable* also named `cookies` inside the pre-existing (Phase 1) `session_read` function:

```ecko
fn session_read(header, secret) {
    cookies = parse_cookies(header)   # <-- local var named same as new top-level fn
    raw = get(cookies, "sid")
    ...
}
```

Ecko's module-level function bindings and same-named local assignments in the same file collide: the plain assignment inside `session_read` was being treated as a reassignment of the existing (immutable) global `cookies` binding rather than a fresh local declaration. Fixed by renaming the local variable to `parsed` (implementation detail only, no behavior/signature change, Phase 1 tests unaffected):

```ecko
fn session_read(header, secret) {
    parsed = parse_cookies(header)
    raw = get(parsed, "sid")
    if is_null(raw) { null } else { unsign(raw, secret) }
}
```

- Re-ran → **20/20 passed**.
- `fmt --check main.ecko framework_test.ecko` → clean (exit 0), no reformatting needed at this point.
- Staged: `git add main.ecko framework_test.ecko`.

---

## Task 2 - Blueprints (route groups) + app flattening

**Test command:** same as above.

- Step 1: Appended the 3 test cases (`blueprint` prefixing, group-middleware wrapping, `app` dispatching a blueprint) to `framework_test.ecko`.
- Step 2: Ran → confirmed failure: `1 file(s), 23 case(s): 20 passed, 3 failed` (all 3 new cases: `Module has no export 'blueprint'`).
- Step 3a: Added `compose_mw` and `blueprint` to `main.ecko`, verbatim from the plan.
- Step 3b: Replaced the Phase 1 `fn app(spec)` body with the flattening version verbatim from the plan (routes list now flattens one level so a blueprint's list of routes splices in via `type_of(r) == "list"`).
- Added `blueprint` to the export list.
- Step 4: Ran → **23/23 passed**, including all pre-existing Phase 1 `app` cases (`app routes a request through web.router`, `app maps a router 404 to a custom error page`, `abort inside a handler routes to that status page`, `a thrown bug becomes a 500`, `security + cors options add middleware`) - all still green, confirming the `app` rewrite didn't regress Phase 1 behavior.
- `fmt --check` initially reported **not formatted** on both files (long call-expressions from the new multi-line test literals wrapping past the line-length threshold). Ran `.../ecko fmt main.ecko framework_test.ecko` to auto-format (only line-wrapping changes, e.g. `main.blueprint("/api", [...])` calls split across lines); re-ran tests → still 23/23 passed; `fmt --check` → clean.
- Staged: `git add main.ecko framework_test.ecko`.

---

## Task 3 - `url_for`

**Test command:** same as above; final step also runs the full suite.

- Step 1: Appended the 3 `url_for` test cases (path-param fill, sorted/encoded query, path-only) to `framework_test.ecko`.
- Step 2: Ran → confirmed failure: `1 file(s), 26 case(s): 23 passed, 3 failed` (all 3: `Module has no export 'url_for'`).
- Step 3: Added `import std.encoding` to the import block and implemented `url_for(pattern, params = empty_map(), query = empty_map())` verbatim from the plan (fills `:name` path segments via `replace`, builds `k=v` pairs URL-encoded via `encoding.url_encode`, sorts them, joins with `&`). Added `url_for` to the export list. No naming collisions this time (the `query` parameter is a fresh local binding inside `url_for`, not a reassignment of the top-level `query` function, so it did not hit the Task 1 pattern).
- Step 4: Ran `framework_test.ecko` → **26/26 passed**. Ran the **full suite** (`.../ecko test`, both `framework_test.ecko` and `webkit_test.ecko`) → **2 file(s), 39 case(s): 39 passed, 0 failed**. `fmt --check main.ecko framework_test.ecko` → clean.
- Staged: `git add main.ecko framework_test.ecko`.

---

## Task 4 - Docs

- Step 1: Added the "### Requests & routing" subsection to `README.md` under the existing "## Framework" section (before "## Testing"), verbatim from the plan (query/query_int/form/json_body/session snippet, blueprint grouping example, `url_for` example).
- **Minor deviation (doc-accuracy fix, not literally specified by the plan):** the pre-existing "## Testing" section's `ecko test` comment stated `(28 cases)`, which was already stale before Phase 2 and is now off by more (actual count is 39). Updated it to `(39 cases)` so the README doesn't mislead readers. This is a one-line factual correction, not a functional change.
- Step 2: Re-ran the full suite (`.../ecko test`) → **39/39 passed**, confirming nothing regressed from the doc edit. (Markdown has no `fmt --check` target, per the plan.)
- Staged: `git add README.md`.

---

## Final verification (per top-level instructions)

```
$ /home/sean/Development/ecko/core/target/debug/ecko test
./framework_test.ecko: 26 case(s) - all ok
./webkit_test.ecko: 13 case(s) - all ok
2 file(s), 39 case(s): 39 passed, 0 failed (0.02s)

$ /home/sean/Development/ecko/core/target/debug/ecko fmt --check main.ecko framework_test.ecko
(exit 0, no output - clean)
```

All Phase 1 (15 cases in `framework_test.ecko` + 13 cases in `webkit_test.ecko` = 28) and Phase 2 (11 new cases: 5 request-access + 3 blueprint + 3 url_for) cases pass. Total: **39 cases, 0 failures**.

---

## Deviations / concerns summary

1. **`session_read` local-variable rename** (Task 1) - required to resolve a name collision between the new top-level `cookies` export and a pre-existing local variable of the same name inside `session_read`. Renamed the local to `parsed`; purely internal, no external behavior change, verified via the full Phase 1 + Phase 2 suite staying green.
2. **`ecko fmt` reformatting** (Task 2) - the formatter line-wrapped some of the newly-added multi-argument test calls (e.g. `main.blueprint(...)`) across multiple lines. This is expected/routine per the plan's own TDD step ("fix with `.../ecko fmt` if needed") and not a concern.
3. **README test-count correction** (Task 4) - updated a stale `(28 cases)` reference to `(39 cases)`. Small deviation beyond the plan's literal text, done for documentation accuracy; flagged here for visibility.

No other deviations. No `ecko-core`/`ecko-std` changes were made. `compat/` was not touched. No commits were made - all changes are staged only, per the hard rule that the owner commits.

---

## Final `git status --short`

```
M  README.md
A  docs/superpowers/plans/2026-07-15-webkit-framework-phase1.md
A  docs/superpowers/plans/phase1-report.md
A  docs/superpowers/specs/2026-07-15-webkit-framework-design.md
M  ecko.json
A  framework_example.ecko
A  framework_test.ecko
M  main.ecko
?? .superpowers/
?? docs/superpowers/plans/2026-07-15-webkit-framework-phase2.md
?? webkit.zip
```

(The `A`/`M` entries for `README.md`, `framework_test.ecko`, and `main.ecko` include both the Phase 1 work already staged on this branch and this session's Phase 2 additions, staged incrementally per task above. The remaining `A` entries - phase1 plan/report/spec, `ecko.json`, `framework_example.ecko` - are pre-existing Phase 1 staged files, untouched by this session. Untracked `.superpowers/`, the phase2 plan file, and `webkit.zip` were left alone - out of scope for Phase 2 work.)

## Final `git diff --cached --stat`

```
 README.md                                          |  55 +-
 .../plans/2026-07-15-webkit-framework-phase1.md    | 659 +++++++++++++++++++++
 docs/superpowers/plans/phase1-report.md            | 285 +++++++++
 .../specs/2026-07-15-webkit-framework-design.md    | 221 +++++++
 ecko.json                                          |   2 +-
 framework_example.ecko                             |  26 +
 framework_test.ecko                                | 202 +++++++
 main.ecko                                          | 233 +++++++-
 8 files changed, 1678 insertions(+), 5 deletions(-)
```
