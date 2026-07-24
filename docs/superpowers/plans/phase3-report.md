# webkit Framework - Phase 3 Report

Executed per `docs/superpowers/plans/2026-07-15-webkit-framework-phase3.md`,
Tasks 1-4, in order, test-first, on branch `feature/framework-phase1`. All
work staged only; owner commits.

## Task 1: `e` alias + remove the old `render` engine

**webkit_test.ecko**: deleted the 5 test cases that exercised `main.render`
(`"render escapes interpolated values by default"`, `"render auto-escapes an
XSS payload"`, `"triple braces opt out of escaping for trusted HTML"`,
`"render fills multiple holes and leaves text intact"`, `"a missing key
renders empty"` - the last one's title doesn't start with "render" but it
calls `main.render`, so it had to go too), then added:

```ecko
test.case("e is an alias for escape", || {
    test.eq(main.e("<b>&"), "&lt;b&gt;&amp;")
})
```

Verify-fail: `ecko test webkit_test.ecko` → `FAIL e is an alias for escape -
Runtime error: Module has no export 'e'`, 8 passed / 1 failed, confirming the
5 render cases were gone and the new one failed as expected.

**main.ecko**: deleted `fn lookup` and `fn render` entirely (the `{{ }}` /
`{{{ }}}` templating engine). Added `fn e(s) = escape(s)` next to `escape`.
Updated `export { ... }`: removed `render`, added `e`.

**example.ecko**: replaced the `main.render(...)` templating demo with:

```ecko
template page(title, note) = """<h1>{title}</h1><p>{note}</p>"""
rendered = page("Welcome", main.e("<script>alert(1)</script>"))
print("rendered: " + rendered)
```

Verify-pass: `ecko test webkit_test.ecko` → 9/9 passed. `ecko example.ecko`
runs, output: `rendered: <h1>Welcome</h1><p>&lt;script&gt;alert(1)&lt;/script&gt;</p>`.
`ecko fmt --check main.ecko webkit_test.ecko example.ecko` → clean.

Staged: `main.ecko webkit_test.ecko example.ecko`.

## Task 2: Flash messages

**framework_test.ecko**: appended the 3 tests from the plan verbatim
(`"flash then flashes round-trips messages in order"`, `"flashes is empty
with no cookie or a bad secret"`, `"clear_flash expires the cookie"`).

Verify-fail: `ecko test framework_test.ecko` → 3 new cases FAIL (`Module has
no export 'flash'` / `'flashes'` / `'clear_flash'`), 26 passed / 3 failed.

**main.ecko**: `import std.encoding` was already present from Phase 1/2 (no
change needed there). Added `# --- flash messages ---` at the end of the
file (after `url_for`) with `flash_pending`, `flash`, `flashes`, `clear_flash`
exactly as specified in the plan. Added `flash, flashes, clear_flash` to
`export { ... }`.

Verify-pass: `ecko test framework_test.ecko` → 29/29 passed.
`ecko fmt --check` initially flagged `main.ecko` (not formatted); ran
`ecko fmt main.ecko` (reflowed the `flash` `with_header(...)` call across
multiple lines), re-checked clean, re-ran the suite to confirm the
reformatting didn't change behavior - still 29/29.

Staged: `main.ecko framework_test.ecko`.

## Task 3: Form validation

**framework_test.ecko**: appended the 3 tests from the plan verbatim
(`"validate flags a missing required field"`, `"validate coerces an int and
enforces min"`, `"validate checks email format"`).

Verify-fail: `ecko test framework_test.ecko` → 3 new cases FAIL (`Module has
no export 'validate'`), 29 passed / 3 failed.

**main.ecko**: added `import std.re`. Added `# --- form validation ---`
section (`validate_field`, `validate`) at the end of the file, exactly as
specified. Added `validate` to `export { ... }`.

Verify-pass: `ecko test framework_test.ecko` → 32/32 passed. Then full suite:
`ecko test` → **2 file(s), 41 case(s): 41 passed, 0 failed**.
`ecko fmt --check` again flagged `main.ecko`; ran `ecko fmt main.ecko`
(collapsed the `else if` chain in `validate_field` onto one long line - this
is the formatter's canonical output, confirmed idempotent via `fmt --check`
passing afterward), re-ran the suite - still 41/41.

Staged: `main.ecko framework_test.ecko`.

## Task 4: Templating & escaping guide

**README.md**:
- Replaced the top bullet list's `{{ }}` / `{{{ }}}` templating description
  with native-template + explicit-escaping wording, and added bullets for
  flash messages and form validation (new Phase 3 surface).
- Removed the old `## Templates` section, which documented the now-deleted
  `webkit.render`/`{{ }}` API - leaving it would have been misleading
  reference docs for code that no longer exists. This wasn't spelled out as
  a literal step in the plan (Task 4 lists only additions), but it follows
  directly from Task 1's breaking change and the plan's stated Goal
  ("removing the old `{{ }}` render engine (with the escaping guide)").
- Added `### Templating & escaping` and `### Flash & forms` under the
  `## Framework` section (after `### Requests & routing`), using the plan's
  markdown verbatim.
- Updated the Testing line to
  `ecko test          # offline: escaping, cookies, sessions, flash, validation, middleware, framework (41 cases)`.

Sanity-checked the new templating snippet against the runtime (a scratch
`.ecko` file with the same `template row`/`template list` + `{for it in
items}{row(it)}{end}` pattern) - output matched what the README claims.

Staged: `README.md`.

## Deviations from the plan (minor)

1. Task 1's step 1 description says "delete the four cases whose names start
   with `render`" - only 3 case titles literally start with the word
   "render"; the 4th render-calling case is titled `"a missing key renders
   empty"`. Deleted all 5 cases that call `main.render` (the 3 "render…"
   titles + "triple braces…" + "a missing key renders empty"), matching the
   plan's intent (no dangling calls to a removed function) and the parent
   task's paraphrase ("delete the 4 `render` tests + the triple braces
   test").
2. `ecko fmt` reformatted `main.ecko` after both Task 2 and Task 3 edits
   (multi-line `with_header` call; a collapsed `else if` chain in
   `validate_field`). Ran `ecko fmt` each time and re-verified tests +
   `fmt --check` afterward, as the process requires.
3. Removed the README's `## Templates` section (see above) - a judgment call
   to keep docs accurate for the breaking change, not a literal plan step.

No other deviations. No `ecko-core`/`ecko-std` changes were made. `compat/`
was not touched (doesn't exist in this repo). No map-literal string keys used
(all built with `insert(empty_map(), "k", v)`). Cookie values are
url-encoded/decoded exactly as constrained (flash cookie:
`encoding.url_encode(sign(...))` / `unsign(encoding.url_decode(...))`).

## Final verification

```
$ cd ~/Development/ecko/webkit && /home/sean/Development/ecko/core/target/debug/ecko test
2 file(s), 41 case(s): 41 passed, 0 failed (0.02s)

$ /home/sean/Development/ecko/core/target/debug/ecko example.ecko
rendered: <h1>Welcome</h1><p>&lt;script&gt;alert(1)&lt;/script&gt;</p>
signed:   user=42.c19695817cf074ef9a39df1735879eb944b02ca87c477a254b2246cbaf565f5f
unsign:   user=42
tampered: null
CORS:     *
nosniff:  nosniff

$ /home/sean/Development/ecko/core/target/debug/ecko fmt --check main.ecko webkit_test.ecko framework_test.ecko example.ecko
(exit 0, no output - clean)
```

### `git status --short` (final)

```
M  README.md
A  docs/superpowers/plans/2026-07-15-webkit-framework-phase1.md
A  docs/superpowers/plans/2026-07-15-webkit-framework-phase2.md
A  docs/superpowers/plans/2026-07-15-webkit-framework-phase3.md
A  docs/superpowers/plans/phase1-report.md
A  docs/superpowers/plans/phase2-report.md
A  docs/superpowers/plans/phase3-report.md
A  docs/superpowers/specs/2026-07-15-webkit-framework-design.md
M  ecko.json
M  example.ecko
A  framework_example.ecko
A  framework_test.ecko
M  main.ecko
M  webkit_test.ecko
?? .superpowers/
?? webkit.zip
```

(The `docs/superpowers/plans/*phase1*`, `*phase2*`, `phase1-report.md`,
`phase2-report.md`, `docs/superpowers/specs/*`, `ecko.json`,
`framework_example.ecko` entries and the untracked `.superpowers/` /
`webkit.zip` predate this session - Phases 1-2 and repo scaffolding, staged
or untracked before Phase 3 work began. Phase 3 touched and staged
`README.md`, `example.ecko`, `framework_test.ecko`, `main.ecko`,
`webkit_test.ecko`, and staged the phase 3 plan doc + this report file for
consistency with how Phases 1-2 staged their own plan/report docs.)

### `git diff --cached --stat` (final)

```
 README.md                                          | 113 +++-
 .../plans/2026-07-15-webkit-framework-phase1.md    | 659 +++++++++++++++++++++
 .../plans/2026-07-15-webkit-framework-phase2.md    | 307 ++++++++++
 .../plans/2026-07-15-webkit-framework-phase3.md    | 297 ++++++++++
 docs/superpowers/plans/phase1-report.md            | 285 +++++++++
 docs/superpowers/plans/phase2-report.md            | 138 +++++
 docs/superpowers/plans/phase3-report.md            | 208 +++++++
 .../specs/2026-07-15-webkit-framework-design.md    | 221 +++++++
 ecko.json                                          |   2 +-
 example.ecko                                       |  10 +-
 framework_example.ecko                             |  26 +
 framework_test.ecko                                | 245 ++++++++
 main.ecko                                          | 354 +++++++++--
 webkit_test.ecko                                   |  24 +-
 14 files changed, 2800 insertions(+), 89 deletions(-)
```

## Spec coverage

- `e` alias - done (Task 1).
- Remove `render`/`lookup` (breaking) + tests + example - done (Task 1).
- `flash`/`flashes`/`clear_flash` - done (Task 2).
- `validate` (+ `validate_field`) - done (Task 3).
- Templating & escaping guide, flash/forms guide, Testing count - done (Task 4).

This completes Phase 3, the final phase of the plan's phased scope.
