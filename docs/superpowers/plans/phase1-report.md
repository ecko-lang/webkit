# webkit Framework - Phase 1 (Core) - Implementation Report

Branch: `feature/framework-phase1` (webkit repo). Binary used for every command:
`/home/sean/Development/ecko/core/target/debug/ecko` (never the system `ecko`).

Overall result: **all 5 tasks completed**, all acceptance criteria met, with a
small number of necessary deviations from the plan's literal code (documented
below, all forced by real parser/capability behavior, not by choice).

---

## Task 1 - Content types & single-file serving

- Created `framework_test.ecko` with the plan's 3 tests verbatim.
- Ran `ecko test framework_test.ecko` → **failed as expected**:
  `Runtime error: Module has no export 'content_type'` / `'file'`.
- Implemented `get_or`, `ext_of`, `content_type`, `file` in `main.ecko` exactly
  as specified; added `import std.http`, `import std.fs`, `import std.web`;
  extended `export { ... }`.
- Added `"capabilities": ["fs:read"]` to `ecko.json` (later revised - see
  Task 5 finding below).
- Ran `ecko test framework_test.ecko` → **3/3 PASS**.
- `ecko fmt --check main.ecko framework_test.ecko` → not formatted; ran
  `ecko fmt` to fix; re-check clean.
- Staged: `main.ecko`, `framework_test.ecko`, `ecko.json`.

## Task 2 - Response helpers

- Appended the plan's 5 tests to `framework_test.ecko`.
- **Deviation (forced by parser):** the plan's `with_headers` test builds
  `{ "x-a": "1", "x-b": "2" }` as a map literal. Ecko's parser only accepts a
  bare identifier or keyword before `:` in a `{ }` literal
  (`peek_is_struct_field` in `crates/ecko-core/src/parser/parser.rs`); a
  quoted string key is a parse error (`Unexpected token: ':'`). Fixed by
  building that map with `insert(insert(empty_map(), "x-a", "1"), "x-b", "2")`
  instead - same value, no literal-syntax change needed elsewhere in Task 2.
- Ran the (corrected) tests → **failed as expected**: `content_type`/`file`
  cases still passed; the 5 new ones failed with `Module has no export
  'with_headers'` etc. (one, `abort`, failed with a slightly different message
  - `expected http:404, got null:null` - because the undefined-member error
  itself gets caught by the test's own `try/catch`; still the correct failure
  mode: `abort` didn't exist yet).
- Implemented `with_headers`, `cache`, `redirect`, `abort`, `html`, `json`,
  `text` exactly as specified (placed below `with_header`, before `cors`).
  Extended exports.
- Ran tests → **8/8 PASS**. `fmt --check` → fixed, then clean.
- Staged: `main.ecko`, `framework_test.ecko`.

## Task 3 - Static passthrough & cache-control middleware

- Appended the plan's 2 tests verbatim (no quoting issues - `prefix`/`value`
  are valid identifier keys).
- Ran → **failed as expected**: `Module has no export 'static'` / `'cache_control'`.
- Implemented `static` and `cache_control` exactly as specified (placed after
  `text`, before `cors`). Extended exports.
- Ran → **10/10 PASS**. `fmt --check` → already clean.
- Staged: `main.ecko`, `framework_test.ecko`.

## Task 4 - App assembly & error handling

- Appended the plan's 5 tests.
- **Same map-literal deviation as Task 2:** `errors: { "404": |req| ... }`
  and `errors: { "403": |req, e| ... }` don't parse (numeric-string keys
  can't be bare identifiers either - same `peek_is_struct_field` limitation).
  Fixed with `errors: insert(empty_map(), "404", |req| { ... })` /
  `insert(empty_map(), "403", |req, e| { ... })`. `handle_error`'s own
  `get(errors, string(status))` lookup is unaffected - the map's *values* and
  runtime shape are identical, only the construction syntax changed.
- **Deviation (plan omission):** the new tests use `web.get(...)` but
  `framework_test.ecko` had no `import std.web`. First run threw
  `Undefined variable 'web'` even after `app` was implemented. Added
  `import std.web` to the test file's imports (one line, after `import std.fs`).
- Ran → **failed as expected**: `Module has no export 'app'` on all 5 new cases.
- Implemented `default_error_page`, `handle_error`, `app` exactly as specified
  (appended after `security_headers`). Extended exports.
- Ran → **15/15 PASS**. `fmt --check` → fixed, then clean.
- Ran the full suite: `ecko test` → **28 cases, 28 passed** (13 original +
  15 new).
- Staged: `main.ecko`, `framework_test.ecko`.

## Task 5 - Acceptance: example + splash rebuild + docs

- Created `framework_example.ecko`. Same map-literal fix applied to the
  `errors:` block (`insert(empty_map(), "404", |req| ...)`).
- Ran `ecko framework_example.ecko` → output matched the plan **exactly**:
  ```
  GET / -> 200 text/html; charset=utf-8
  secure? nosniff
  GET /nope -> 404 Not found: /nope
  ```
  (verified again after `ecko fmt` reformatted the file - output unchanged.)
- Appended the "Framework" section to `README.md` after Middleware, before
  Testing (same `insert()` fix applied to the `errors:` line in the example
  snippet so it's actually copy-pasteable). Also updated the Testing section's
  case count from "13 cases" to "28 cases" to match reality (not in the plan's
  literal text, but leaving a now-false count seemed worse than a one-line fix).
  One thing intentionally left as the plan wrote it: the README's Escaping
  bullet mentions `webkit.e` (an alias) - that alias doesn't exist yet (it's
  listed as Phase 3 / out of scope in the plan's own self-review). This is a
  pre-existing forward-reference in the plan's text, not something Phase 1
  introduces; flagging it here rather than silently editing the plan's prose.
- Rewrote `website/splash/server.ecko` and created `website/splash/ecko.json`
  exactly as specified.
- **Finding, not anticipated by the plan:** `std.web` (and `std.http`) are
  gated `Gate::All("net")` in `ecko-std`'s module registry - *every* function
  in those modules requires the `net` capability, not just `fs:read`. Since
  `webkit.static`, `webkit.cache_control`→(`web.router` is called inside
  `webkit.app`), and `webkit.html/json/text` (via `http.html/json/text`) all
  call into `std.web`/`std.http` under the hood, webkit needs `net` - `fs:read`
  alone (as the plan's Task 1/Task 5 manifests specify) is not sufficient once
  webkit is used as a *vendored package* rather than run as its own repo root
  (root code always holds full authority regardless of its own manifest, which
  is why `framework_test.ecko`/`framework_example.ecko` passed fine without
  this - they run webkit as root, not as an imported dependency). Confirmed
  with a minimal standalone repro package before touching the real files.
  **Fix:** added `"net"` to `webkit/ecko.json`'s `capabilities` (now
  `["fs:read", "net"]`) and to `website/splash/ecko.json`'s
  `dependencies.webkit.grant` (now `["fs:read", "net"]`).
- **Vendoring mechanics:** `ecko add <dir>` rejects a raw directory
  (`can't read package '...': Is a directory`) - `ecko add` expects a package
  *file* (a zip), matching `pkgcmd.rs`'s local-file-path source handling. Used
  `ecko pack -o webkit.zip` (run from the webkit repo) to produce one, then
  `ecko add /home/sean/Development/ecko/webkit/webkit.zip` from
  `website/splash`, which vendored + locked successfully. `ecko add` writes
  only `source`/`sha256` into the dependency entry (it does not preserve or
  accept a `grant` list), so `grant` was added by hand to
  `website/splash/ecko.json` after each `add`. Verified `ecko install` (after
  deleting `vendor/` and re-running from the committed `ecko.lock`) also
  resolves cleanly from this path.
- `webkit.zip` (the packed artifact `ecko add`/`ecko install` resolve against)
  lives at `/home/sean/Development/ecko/webkit/webkit.zip` - **left on disk,
  intentionally not staged/git-added** (it's a generated build artifact, not
  source). It needs to exist for `ecko install` in `website/splash` to
  re-resolve the dependency; regenerate it any time with
  `cd webkit && ecko pack -o webkit.zip`. The owner should decide whether to
  commit it, `.gitignore` it, or point at a different distribution mechanism
  later - flagging rather than deciding unilaterally.
- **Live verification** (server actually run and exercised, not just written):
  ```
  cd website/splash && ecko server.ecko &
  curl -s -D - -o /dev/null localhost:8082/
    → HTTP/1.1 200 OK, cache-control: public, max-age=60, content-type: text/html; charset=utf-8
  curl -sI localhost:8082/icons/favicon.ico | grep -i cache-control
    → cache-control: public, max-age=86400
  curl -sI localhost:8082/brand/ecko-mark.svg | grep -i cache-control
    → cache-control: public, max-age=86400
  curl -s localhost:8082/healthz
    → ok
  ```
  Freshness check: appended a marker HTML comment to `public/index.html`
  while the server was running, re-curled `/` with no restart - the new
  content appeared immediately (confirms `webkit.file` reads fresh, no
  read-once staleness). The marker line was then removed to restore
  `public/index.html` to its original content (verified: original byte
  length 14722, `Content-Length: 14722` after restore, no `freshness-probe`
  string remains). Server process stopped afterward
  (`pkill -f "target/debug/ecko server.ecko"`; confirmed no process remains).

---

## Test commands run (exact) + pass/fail counts

| Step | Command | Result |
|---|---|---|
| Task 1 red | `ecko test framework_test.ecko` | 0/3 pass (2× "Module has no export") |
| Task 1 green | `ecko test framework_test.ecko` | 3/3 pass |
| Task 2 red | `ecko test framework_test.ecko` | 3/8 pass, 5 fail (undefined members) |
| Task 2 green | `ecko test framework_test.ecko` | 8/8 pass |
| Task 3 red | `ecko test framework_test.ecko` | 8/10 pass, 2 fail (undefined members) |
| Task 3 green | `ecko test framework_test.ecko` | 10/10 pass |
| Task 4 red | `ecko test framework_test.ecko` | 10/15 pass, 5 fail (undefined `app`) |
| Task 4 green | `ecko test framework_test.ecko` | 15/15 pass |
| Full suite | `ecko test` | **28/28 pass** (13 original `webkit_test.ecko` + 15 `framework_test.ecko`) |
| Example | `ecko framework_example.ecko` | exact expected output, byte-for-byte |
| Original example (regression) | `ecko example.ecko` | unchanged, runs clean |

## fmt status

`ecko fmt --check main.ecko framework_test.ecko framework_example.ecko` →
**clean** (exit 0) as of the final pass. `ecko fmt` was applied once per task
where it reported drift; each time, tests were re-run afterward to confirm
formatting didn't change behavior.

## Deviations / concerns summary

1. **Map literals with quoted/non-identifier keys don't parse** in this Ecko
   build (`{ "x-a": "1" }`, `{ "404": ... }`) - affects Task 2's
   `with_headers` test, Task 4's `errors:` tests, the Task 5 example's
   `errors:` block, and the README's Framework snippet. All fixed with
   `insert(empty_map(), key, value)` chains, which is the established idiom
   already used elsewhere in `main.ecko` (e.g. `with_header`). No production
   code semantics changed - only how test/example maps with such keys are
   *constructed*. Worth flagging upstream: the plan's own code (and Ecko
   convention for HTTP status/header maps) assumes quoted-string map-literal
   keys work; they don't in the current parser.
2. **`framework_test.ecko` needed `import std.web`** for the Task 4 tests
   (`web.get`) - a one-line omission in the plan's Step 1 code, fixed inline.
3. **webkit needs the `net` capability, not just `fs:read`**, to work as a
   *vendored* package once it calls into `std.web`/`std.http` (both are
   `Gate::All("net")`). `fs:read` alone is necessary for `file()`'s disk
   reads, but insufficient for `static()`, `cache_control()`'s and `app()`'s
   `web.router`/`web.static` calls, and `html()/json()/text()`'s
   `http.html/json/text` calls. Updated `webkit/ecko.json` capabilities to
   `["fs:read", "net"]` and `website/splash/ecko.json`'s grant to match. This
   only manifests when webkit is imported as a dependency (root code has full
   authority, so the webkit repo's own `framework_test.ecko`/
   `framework_example.ecko` never hit this) - confirmed with an isolated
   minimal-package repro before changing the real files.
4. **`ecko add <dir>` doesn't vendor a raw local directory** - it wants a
   packed zip file. Used `ecko pack -o webkit.zip` + `ecko add <path>.zip`
   per the plan's own contingency note. `ecko add` doesn't preserve/accept a
   `grant` field, so it was re-added by hand to `splash/ecko.json` after each
   `add`. This matches the plan's stated ambiguity-resolution guidance.
5. **`webkit.zip`** (the packed dependency artifact) is left, untracked, at
   `/home/sean/Development/ecko/webkit/webkit.zip` so `website/splash`'s
   `ecko install` keeps resolving. It is **not staged** in the webkit repo.
   Regenerate with `cd webkit && ecko pack -o webkit.zip` if it's ever
   deleted.
6. **README**: updated the "13 cases" count to "28 cases" in the Testing
   section (not explicitly requested, but the old number was now false).
   Left the `webkit.e` alias mention as-is per the plan's literal text even
   though that alias doesn't exist until Phase 3 (see item above).

None of these affect the Phase 1 acceptance criteria: all specified functions
exist with the specified signatures/behavior, all tests pass, `fmt --check`
is clean, and the splash server runs for real against the framework with the
exact cache-control/freshness behavior the plan describes.

---

## Final `git status --short` (webkit repo)

```
M  README.md
M  ecko.json
A  framework_example.ecko
A  framework_test.ecko
M  main.ecko
?? .superpowers/
?? docs/
?? webkit.zip
```

`.superpowers/` and `docs/` predate this session's changes (pre-existing
untracked directories, not touched by this work) and were left alone.
`webkit.zip` is the generated pack artifact described above - intentionally
left unstaged.

## Final `git diff --cached --stat` (webkit repo)

```
 README.md              |  32 +++++++++-
 ecko.json              |   2 +-
 framework_example.ecko |  26 ++++++++
 framework_test.ecko    | 128 ++++++++++++++++++++++++++++++++++++++
 main.ecko              | 164 ++++++++++++++++++++++++++++++++++++++++++++++++-
 5 files changed, 349 insertions(+), 3 deletions(-)
```

## website/splash - not a git repository

`/home/sean/Development/ecko/website` has no `.git` (confirmed:
`fatal: not a git repository`), so there is nothing to stage/commit there -
the plan's Task 5 Step 5 commit note for the website repo doesn't apply.
Files written/modified directly on disk:

- `website/splash/server.ecko` - rewritten on the webkit framework, exactly
  per the plan.
- `website/splash/ecko.json` - created, with `webkit` as a dependency
  (`grant: ["fs:read", "net"]` - see deviation #3).
- `website/splash/ecko.lock` - generated by `ecko add`/`ecko install`.
- `website/splash/vendor/webkit/` - vendored copy of webkit (from the packed
  zip), regenerable via `ecko install`.
- `website/splash/public/index.html` - touched transiently for the freshness
  check, then fully restored (verified byte-identical: `Content-Length: 14722`
  before and after, no leftover marker text).

No user-facing production content in `website/splash` was left modified.

## Commits

**None were created** - per the hard rule (never run `git commit`; the repo
owner commits). All webkit-repo changes for Tasks 1-5 are staged as shown
above, ready for the owner to review and commit (in whatever number of
commits they prefer - the plan suggested one per task).
