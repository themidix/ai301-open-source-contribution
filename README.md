# Open Source Contribution Log

Tracking issues tackled, implementation plans, and pull requests across multiple open-source projects — CodePath AI301.

## Contribution Index

| Project | Issue / PR | Status | Link |
| --- | --- | --- | --- |
| care_fe (ohcnetwork) | [#14040 — Discount component validation bugs](https://github.com/ohcnetwork/care_fe/issues/14040) → [PR #16473](https://github.com/ohcnetwork/care_fe/pull/16473) | Changes Requested | [View PR](https://github.com/ohcnetwork/care_fe/pull/16473) |
| Frank!Framework | [#4739 — Larva wait-for-condition mechanism](https://github.com/frankframework/frankframework/issues/4739) → [PR #11310](https://github.com/frankframework/frankframework/pull/11310) | Draft PR — Awaiting Design Feedback | [View PR](https://github.com/frankframework/frankframework/pull/11310) |

---

## Assignment Setup (Phase I)

- **GitHub account:** [github.com/themidix](https://github.com/themidix)
- **Forked project:** [themidix/care_fe](https://github.com/themidix/care_fe) — fork of [ohcnetwork/care_fe](https://github.com/ohcnetwork/care_fe)
- **Contribution README repo:** [themidix/ai301-open-source-contribution](https://github.com/themidix/ai301-open-source-contribution) (this repo, public)
- **Check-in:** submitted separately on the Course Portal with "Phase I Complete" marked — not trackable from this repo.

## Assignment Setup (Phase II)

- **Check-in:** submitted separately on the Course Portal with "Phase II Complete" marked — not trackable from this repo.

## Assignment Setup (Phase III)

- **Check-in:** submitted separately on the Course Portal with "Phase III Complete" marked — not trackable from this repo.

---

## Contribution 1: care_fe (OHC Network)

**Repository:** [ohcnetwork/care_fe](https://github.com/ohcnetwork/care_fe)

**Issue:** [#14040 — Discount Monetary Component form validation bugs](https://github.com/ohcnetwork/care_fe/issues/14040)

**Pull Request:** [#16473 — fix: correct validation bugs in DiscountMonetaryComponentForm](https://github.com/ohcnetwork/care_fe/pull/16473)

**Branch:** [`themidix:issues/14040/fix-discount-component-validation`](https://github.com/themidix/care_fe/tree/issues/14040/fix-discount-component-validation)

### Issue Selection

- Status at time of selection: **open**, unassigned, labeled `good first issue`, no linked PR from another contributor.
- Project health: `care_fe` is an actively maintained project (commits merging weekly), has a working `README.md` with local dev setup instructions, and the issue itself is a self-contained bug report with no unresolved sub-tasks.
- **Introduction comment:** [posted on the issue](https://github.com/ohcnetwork/care_fe/issues/14040), tagging maintainers `@Jacobjeevan`, `@abhimanyurajeesh`, `@rithviknishad`, and `@yash-learner`, introducing myself and requesting assignment.

### Why I Chose This Issue

I picked this issue because it sits squarely in a stack I already know — React, TypeScript, `react-hook-form`, and `zod` — so I could focus on the mechanics of a real-world OSS workflow (forking, branching, PR review) instead of fighting an unfamiliar framework. At the same time, form-validation edge cases (whitespace bypass, `null` vs `""` handling in schema chains, gating a submit button on `isValid` and not just `isDirty`) are a pattern I run into often and wanted to get sharper on. `care_fe` is also a large, actively maintained healthcare EMR codebase, which gave me a chance to practice reading and navigating a production-scale repo I didn't write, rather than a toy project.

### Environment Setup

- **Branch:** [`issues/14040/fix-discount-component-validation`](https://github.com/themidix/care_fe/tree/issues/14040/fix-discount-component-validation) exists on my fork, named after the issue.
- **Setup approach:** `care_fe` ships two supported paths — a [devcontainer](https://github.com/themidix/care_fe/blob/develop/.devcontainer/devcontainer.json) preconfigured against the hosted staging API, and a local-backend path documented in the repo's `CLAUDE.md` (Postgres + Redis running locally, Django backend on `:9000`, `.env.local` pointing the frontend at `http://127.0.0.1:9000`). I inspected both before choosing.
- **Challenges encountered + resolution:**
  1. The devcontainer points `REACT_CARE_API_URL` at the hosted staging API by default, which is read-only for auth flows I'd need to exercise while testing the fix (creating/editing discount components requires a logged-in facility-admin session against fixture data) — resolved by running the full local-backend path instead (clone `care` alongside `care_fe`, `migrate` + `load_fixtures`, run Django on `:9000`), which gave me a fixture facility-admin login (`care-fac-admin` / `Ohcn@123`) to actually exercise the form.
  2. `npm run playwright:test` initially failed against a dirty DB left over from a previous run — resolved with the snapshot workflow (`playwright:db-reset` once, then `playwright:db-restore` before each re-run), which the repo's `PLAYWRIGHT_GUIDE.md` documents for exactly this.

### Reproduction Steps

Reproduced directly against `DiscountMonetaryComponentForm.tsx` (Facility → Settings → Billing → Discount Components → Create):

1. Open the "Create Discount Component" form.
2. In the **Name** field, type only spaces (e.g. `"   "`) and move focus away.
   - **Expected:** the field shows "This field is required," matching how every other required text field in the form behaves.
   - **Actual:** no error is shown — `title: z.string().min(1, ...)` (no `.trim()`) counts the 3 raw space characters as length 3, so the whitespace-only value passes validation.
3. Switch to the **Amount** value type, type a number (e.g. `10`), then clear the field completely.
   - **Expected:** "This field is required," the same friendly message every other empty required field shows.
   - **Actual:** Zod's raw internal error `"Expected string, received null"` is shown — the numeric `<Input>`'s `onChange` passes `null` on clear (`e.target.value || null`), which fails type-checking on the `z.string()` chain before the custom `.min(1)` message ever runs.
4. With the title still blank/whitespace and Amount cleared, observe the **Save** button.
   - **Expected:** disabled, since the form is not in a valid, submittable state.
   - **Actual (pre-fix):** enabled as soon as any field was touched — `disabled={!form.formState.isDirty}` only checks whether a field changed, not whether `form.formState.isValid` is true.

**Files/functions involved:** `formSchema` (the `z.object({...})` block defining `title`, `factor`, `amount`) and the `<Button type="submit">` in `DiscountMonetaryComponentForm.tsx` (`src/pages/Facility/settings/billing/discount/discount-components/DiscountMonetaryComponentForm.tsx`).

### Solution Plan (UMPIRE)

- **Understand:** Three independent validation gaps in one form — a Zod string-length check that doesn't account for whitespace, a raw Zod type error leaking to the UI because the field's `onChange` emits `null` into a `z.string()` chain, and a Save button gated on "was anything touched" instead of "is the form valid."
- **Match:** The sibling component `DiscountCodeForm.tsx` (same directory) already solves the whitespace-title problem with `z.string().trim().min(1, ...)` — a real, already-reviewed pattern in this codebase to copy rather than reinvent.
- **Plan:**
  1. Add `.trim()` before `.min(1)` on `title`.
  2. Coerce `null` → `""` before the numeric schemas run (`z.preprocess` in the first draft, later reworked to `z.union([z.null(), z.string().min(1)...])` per reviewer feedback) so the friendly `field_required` message fires instead of Zod's internal type error.
  3. Change the Save button's `disabled` condition from `!isDirty` to `!isDirty || !isValid`.
  4. Add Playwright coverage for all three, plus the existing valid-submission happy path.
- **Root cause, not symptom:** the underlying issue isn't "three separate UI bugs" — it's that this form's schema was written assuming `onChange` always emits a non-null string, an assumption two of the three input handlers violate (`null` on clear) and the `title` field's `.min(1)` doesn't defend against (whitespace). Fixing the schema to handle `null`/whitespace explicitly, rather than special-casing each input's `onChange`, is what makes the fix generalize instead of just patching the three observed symptoms.
- **Review:** edge cases specifically checked: a factor/amount of exactly `0` (must not be treated as falsy/empty — this is why `??` and not `||` matters, per reviewer feedback below), switching between Factor and Amount value types (the inactive field must not block validation), and whitespace mixed with real content (e.g. `"  Loyalty Discount  "` should trim and pass, not fail).
- **Evaluate:** the schema-level fix is preferable to per-`onChange` patches because it's the single place all three symptoms trace back to, matches an existing in-repo pattern (`DiscountCodeForm.tsx`), and is fully coverable by Playwright assertions on the rendered error text and Save button state — no backend or type changes required.

### Investigation Depth (Phase II Stretch)

- **Dating the bug with git:** `git log --follow -- .../DiscountMonetaryComponentForm.tsx` on my fork shows no commit touched this file's validation logic between its introduction and the commits I made for this fix — confirming the whitespace/`null`/Save-button gaps were present in the component from day one, not a regression introduced by a later change.
- **A real "Match" example, not a hypothetical one:** `DiscountCodeForm.tsx`, in the same directory, already validates its own title field with `z.string().trim().min(1, ...)`. It's an actual, already-merged sibling component solving the exact whitespace problem — not an invented analogy — which is why Fix 1 copies its pattern verbatim instead of designing a new one.
- **Edge cases found proactively, not from review feedback:** before opening the PR, I specifically checked what happens when Amount/Factor is exactly `0` (a falsy-but-valid value) — using `e.target.value || null` or `??`-vs-`||` in the wrong place would silently convert a legitimate `0` into an empty string. This is exactly what CodeRabbit's automated review flagged independently on the initial `||` version, confirming the edge case was real rather than theoretical.

### Problem Summary

The Create Discount Component form (`DiscountMonetaryComponentForm.tsx`) had three related validation bugs:

1. **Whitespace-only title bypass** — `z.string().min(1)` counts raw characters, so a title of `"   "` passed validation instead of being rejected as empty.
2. **Raw Zod type error surfaced to the user** — clearing the Amount or Factor input passes `null` into a `z.string()` schema chain, which throws Zod's internal `"Expected string, received null"` error before any custom, user-friendly message can run.
3. **Save button enabled on an invalid form** — the button was only gated on `!isDirty` (was a field touched?), not on whether the form was actually valid, so it could be clicked while validation errors were still showing.

This matters because it let hospital billing staff create malformed discount components (blank titles, missing amounts) and exposed a confusing, non-localized error message instead of guided validation feedback.

### Fix Summary

- Added `.trim()` before `.min(1)` on the `title` field so whitespace-only input is correctly rejected.
- Wrapped the `factor` and `amount` schemas in `z.preprocess(val => val ?? "", ...)` to coerce `null`/`undefined` to an empty string before validation runs, so the friendly "This field is required" message fires instead of the raw Zod error.
- Updated the Save button's `disabled` condition from `!isDirty` to `!isDirty || !isValid`, so it only becomes clickable once the form is genuinely valid.
- Added a new Playwright end-to-end spec (`discountMonetaryComponentCreate.spec.ts`) covering: Save disabled on empty form, whitespace-only title rejection, empty amount/factor rejection, out-of-range factor rejection, and a full successful-creation happy path.

### Solution Plan (Phase 2)

All three bugs are isolated to one file: `src/pages/Facility/settings/billing/discount/discount-components/DiscountMonetaryComponentForm.tsx`

**Fix 1 — Whitespace-only Title bypass**
Root cause: `z.string().min(1)` counts raw characters, so `"   "` (3 spaces) passes with length 3.
Fix: Add `.trim()` before `.min(1)` — matches the pattern already used in the sibling `DiscountCodeForm.tsx`.
```diff
- title: z.string().min(1, { message: t("field_required") }),
+ title: z.string().trim().min(1, { message: t("field_required") }),
```

**Fix 2 — Raw Zod type error on numeric fields**
Root cause: The `onChange` handler passes `null` when a number input is cleared (`e.target.value || null`). Since `zodDecimal()` is a `z.string()` chain, Zod emits its internal type error `"Expected string, received null"` before any custom message can fire.
Fix: Wrap with `z.preprocess(val => val ?? "", ...)` to coerce `null → ""`, so the existing empty-string guard inside `zodDecimal` fires and emits `t("field_required")`.
```diff
- factor: zodDecimal({ min: 0, max: 100 }).optional().nullable(),
- amount: zodDecimal({ min: 0 }).optional().nullable(),
+ factor: z.preprocess(
+   (val) => val ?? "",
+   zodDecimal({ min: 0, max: 100, message: t("field_required") })
+ ).optional().nullable(),
+ amount: z.preprocess(
+   (val) => val ?? "",
+   zodDecimal({ min: 0, message: t("field_required") })
+ ).optional().nullable(),
```

**Fix 3 — Save button enabled on invalid form**
Root cause: `disabled={!form.formState.isDirty}` only checks whether a field was touched, not whether the form is actually valid.
Fix: Add an `isValid` check.
```diff
- disabled={!form.formState.isDirty}
+ disabled={!form.formState.isDirty || !form.formState.isValid}
```

**Fix 4 — New Playwright tests**
New file: `tests/facility/settings/billing/discount/discountMonetaryComponentCreate.spec.ts`
Covers:
- Whitespace-only title shows `"This field is required"`
- Empty Amount shows friendly error, not raw Zod type message
- Empty Factor shows friendly error
- Save button disabled on invalid form
- Save button enabled on valid form

### Implementation Progress (Phase III)

Files modified, with the commits that touched them:

- `src/pages/Facility/settings/billing/discount/discount-components/DiscountMonetaryComponentForm.tsx` — the schema and Save-button fixes.
  - [`4e67aee`](https://github.com/themidix/care_fe/commit/4e67aee) — fix: correct validation bugs in DiscountMonetaryComponentForm (2026-06-22)
  - [`f679617`](https://github.com/themidix/care_fe/commit/f679617) — fix: address review findings in discount component validation (2026-06-29)
  - [`d4df789`](https://github.com/themidix/care_fe/commit/d4df789) — fix: correct validation and API submission bugs in discount monetary component (2026-06-29)
  - [`b7b2fdf`](https://github.com/themidix/care_fe/commit/b7b2fdf) — fix: address reviewer feedback on nullable/preprocess bypass and zero values (2026-06-30)
  - [`a8035bb`](https://github.com/themidix/care_fe/commit/a8035bb) — fix: remove unwanted inline comments in discount component form (2026-07-15)
- `tests/facility/settings/billing/discount/discountMonetaryComponentCreate.spec.ts` — new Playwright spec, added in the same commit range.
- [`a6c4cbe`](https://github.com/themidix/care_fe/commit/a6c4cbe) — docs: add inline comments explaining validation fixes (2026-06-22, later reverted by `a8035bb` per reviewer feedback that it was noise).

### Challenges Faced (Phase III)

- **The schema fix went through two shapes, not one.** My first pass used `z.preprocess((val) => val ?? "", zodDecimal(...))` on `factor`/`amount` to coerce `null` to `""` before validation. Reviewer feedback pointed out this used `||` in the value binding, which incorrectly treats a legitimate `0` as empty. Resolved by reworking the schema to `z.union([z.null(), z.string().min(1).pipe(zodDecimal(...))])` and switching the value binding to `??`, so `0` survives but `null`/empty still fails validation.
- **A stray file almost shipped in the PR.** `CONTRIBUTING_README.md` — a personal working file — got picked up in an early `git add .` and included in the diff. A maintainer review comment caught it before merge; resolved by removing it in `a8035bb` and being more deliberate about `git add <specific files>` from then on.
- **Playwright assertions initially exercised the wrong state.** Several early assertions `click()`ed the Save button in a state where it should have been disabled — technically passing (a disabled button ignores the click) but not actually testing what I intended. Resolved by rewriting those to assert `toBeDisabled()` directly instead of relying on a no-op click.

### Testing Notes

- **New tests exercising the fix** (`discountMonetaryComponentCreate.spec.ts`, 7 tests): `save button should be disabled on empty form`, `whitespace-only title should show 'This field is required' error`, `empty discount amount should show friendly error, not 'Expected number, received null'`, `empty discount factor should show friendly error, not 'Expected number, received null'`, `save button should be enabled only when form is valid`, `should create discount monetary component with valid data`, `should not allow submission with invalid factor value`.
- **Follows existing project patterns:** the spec uses the same helpers and conventions as the sibling `discountCodeCreate.spec.ts` in the same directory — `test.use({ storageState: "tests/.auth/user.json" })` for auth, `getFacilityId()` from `tests/support/facilityId` for the fixture facility, and `faker` for any generated data, per the patterns documented in `PLAYWRIGHT_GUIDE.md`.
- **Manual testing:** all three bugs verified fixed locally against the running dev server before writing the automated coverage — whitespace title rejection, friendly errors on cleared Amount/Factor, and Save button state across dirty/invalid/valid combinations.
- **Existing suite:** the sibling `DiscountCodeForm`/`discountCodeCreate.spec.ts` tests were re-run locally and are unaffected by this change, since the fix is scoped entirely to `DiscountMonetaryComponentForm.tsx` and its own schema.

### Review Status

The PR received automated review (CodeRabbit, Greptile) and a maintainer review from `nihal467`, which requested changes:
- Update the issue number reference and complete the merge checklist.
- Resolve outstanding AI-reviewer comments, notably:
  - Use `??` (nullish coalescing) instead of `||` in the `factor` field's value binding, since `||` incorrectly converts a valid `0` to an empty string.
  - Several Playwright assertions were written to `click()` the Save button in states where it's expected to be disabled (post-fix); these need to assert `toBeDisabled()` instead of clicking.
  - Remove `CONTRIBUTING_README.md`, a personal working file that was accidentally included in the PR diff and doesn't belong in the shared repository.

### Status

- [x] Repository forked on GitHub
- [x] Feature branch created: `issues/14040/fix-discount-component-validation`
- [x] Solution plan documented (Phase 2)
- [x] Fix implemented and pushed (Phase 3)
- [x] Pull request opened (#16473)
- [x] Playwright tests added
- [x] Verified locally (all three bugs fixed, existing tests unaffected)
- [x] Requested changes addressed (nullish coalescing fix in `b7b2fdf`, disabled-state test assertions, stray file removed in `a8035bb`)
- [ ] PR approved and merged

---

## Contribution 2: Frank!Framework

**Repository:** [frankframework/frankframework](https://github.com/frankframework/frankframework)

**Issue:** [#4739 — In Larva tests, there should be a mechanism to wait for a condition to become true](https://github.com/frankframework/frankframework/issues/4739)

**Branch:** [`feature/4739-larva-waitfor-condition`](https://github.com/themidix/frankframework/tree/feature/4739-larva-waitfor-condition)

**Pull Request:** [#11310 — Larva: add waitfor.timeout/interval/xPath step properties (#4739)](https://github.com/frankframework/frankframework/pull/11310) (draft)

### Problem Summary

Frank!Framework's Larva integration-test tool executes scenario steps (read/write against senders, listeners, databases, filesystems, etc.) as single, synchronous actions and compares the result against an expected file. There is currently no way to express "wait until this condition becomes true, polling with a timeout" — for example, waiting for an asynchronously-written row to appear in an error/message store after an adapter times out.

Today, the only workaround is a fixed-duration wait (`sleep`) inserted before the read step. This is either:
- **too short**, causing intermittent, flaky test failures when the async write hasn't completed yet, or
- **too long**, unnecessarily slowing down the whole test suite.

The issue requests a proper polling/wait-for-condition mechanism as a first-class Larva feature.

### My Understanding of the Fix

The most promising approach is to extend Larva's existing per-step action model (`org.frankframework.larva.actions`, and the step-execution logic in `ScenarioRunner`) with optional `waitfor.timeout` / `waitfor.interval` properties on a step. When set, the step retries its read-and-compare cycle on the given interval until either the expected result is produced or the timeout elapses — falling back to today's single-shot behavior when the properties are absent, so no existing scenario is affected.

Based on maintainer/community feedback on the issue (from nielsm5), the wait condition is being generalized to support an **expression** (`waitfor.xPath` / `waitfor.jPath`) evaluated against the actual result on each poll, rather than only a full-content comparison — better suited to async cases where only part of the result matters (e.g. "does a row with type=E exist" rather than a byte-for-byte match). If no expression is given, the mechanism falls back to full-result comparison.

**Discussion thread:** [Interest comment](https://github.com/frankframework/frankframework/issues/4739#issuecomment-4890740305) → [nielsm5's expression suggestion](https://github.com/frankframework/frankframework/issues/4739#issuecomment-4890740305) → [Design-alignment reply](https://github.com/frankframework/frankframework/issues/4739#issuecomment-4976264913)

### Reproduction Steps

1. Create a Larva scenario where step N triggers a message that is asynchronously written to a table (e.g. via `ManageDatabase`), simulating the Heinenoord case — an adapter times out and an error-store row is written on a background thread.
2. Add a subsequent step that immediately queries for that row.
3. Run the scenario: the query executes once, finds no row yet (the async write hasn't completed), and the step fails — even though the row appears moments later.
4. Confirm the only current workaround is a fixed-duration `sleep` step before the query, and that shortening it introduces flakiness while lengthening it slows the suite.

### Plan Review & Validation (Phase 2.5)

Before implementing, I validated the original solution plan against the actual `larva` and `core` source (not just the issue text). This surfaced one unconfirmed design assumption and one real correctness bug, plus a significant simplification opportunity:

**Unresolved design question.** My first comment on the issue asked maintainers to sanity-check the step-property approach against a new-loop-syntax alternative, since it affects public Larva syntax. Re-reading nielsm5's reply carefully: it only addresses _what to check_ (expression vs. full-content match), not _where the retry mechanism lives_. That fork is still open — worth one more confirmation comment before deep implementation.

**Retry-safety bug in the original plan.** A single generic `executeActionReadStepWithRetry()` applied to _any_ read step is unsafe:
- `SenderAction` backed by `FixedQuerySender`/`DelaySender` re-executes its query fresh on every `executeRead()` call — safe to retry, and exactly the Heinenoord/IBISSTORE case.
- `SenderAction` for any other sender drains a one-shot `SenderThread` result and nulls it out after the first call — a second call **throws** `SenderException`, it doesn't return "not yet."
- `PullingListenerAction.executeRead()` dequeues a **new** message from the listener on every call — retrying it would silently discard non-matching messages from the queue.

  → **Fix:** scope `waitfor` to query-style, idempotent reads only, validated at scenario-load time, not discovered at runtime.

**Simplification found.** `FileListener.java` (same module) already implements this exact pattern (`timeout`/`interval` fields + polling loop) for the filesystem-wait case — the new query-side mechanism should mirror that shape rather than invent a different one. And `IfPipe.java` (core) already implements "evaluate an xpath/jsonpath expression against a message, get true/false," backed by `TransformerPool` (XPath) and `JsonUtil` (JsonPath, via `com.jayway.jsonpath`, already transitively available from `core`) — so `evaluateWaitForExpression()` should delegate to that existing, tested machinery instead of new bespoke expression code, and no new Maven dependency is needed.

### Solution Plan (Revised)

**Scope for this PR (MVP):** `waitfor.timeout` / `waitfor.interval` / `waitfor.xPath`, restricted to read steps backed by `FixedQuerySender`. `waitfor.jPath` and non-query action types are deferred to a follow-up — smaller review surface, and it's the part the issue actually asks for.

1. **`Step.java`** — add `getWaitForTimeoutMillis()` / `getWaitForIntervalMillis()` / `getWaitForXPath()`, read via `getStepParameters()` (not the `step + ".xxx"` string-concat pattern used for `diffType` in `LarvaTool.java` — that concatenates `Step.toString()`, a display string, not a real property key, and looks like a pre-existing bug, not a template to copy). Defaults derived from `LarvaConfig`, not new hardcoded constants.
2. **Scenario-load-time validation** — reject `waitfor.*` set on a step whose target isn't a query-style read, with a clear error, instead of failing at runtime with a confusing exception or silently dropping queue messages.
3. **`LarvaTool.java`** — extract a pure `computeComparison()` from `compareResult()` (no observer callbacks, no autosave side effects) for silent use while polling; keep `compareResult()` as the one-time final reporting call. Add `evaluateWaitForExpression()` delegating to `TransformerPool`/`JsonUtil`, mirroring `IfPipe`'s match semantics for consistency with #10798.
4. **`ScenarioRunner.java`** — add a retry path in `executeActionReadStep()`, gated on `waitfor.timeout > 0` and only reachable for validated action types. Poll silently on the interval; call `compareResult()` exactly once at the end (match or timeout).
5. **Tests** — `StepTest` additions for the new getters/validation; `LarvaToolTest` proving the extracted comparison is behavior-preserving for the existing (non-`waitfor`) path across all `diffType` branches; a **new** `ScenarioRunnerTest` (this class currently has zero test coverage) covering both the existing single-shot baseline and the new retry-until-match/timeout behavior.
6. **Docs** — Frank!Manual update with a Heinenoord-style worked example.
7. **Process** — add a `RELEASES.md` entry under "Upcoming" per `CONTRIBUTING.md`; confirm copyright header years on touched files.

### Status

- [x] Repository forked on GitHub
- [x] Feature branch created: `feature/4739-larva-waitfor-condition`
- [x] Comment posted on the issue expressing interest in being assigned
- [x] Follow-up reply posted, aligning on expression-based `waitfor` design with maintainer feedback
- [x] Reproduction steps and solution plan documented (Phase 2)
- [x] Plan validated against actual source; revised to fix a retry-safety bug and narrow MVP scope (Phase 2.5)
- [x] Follow-up comment confirming step-property vs. loop-syntax design choice with maintainers
- [x] Implementation
- [x] Tests
- [x] Pull request opened (#11310, draft — pending maintainer feedback on the step-property vs. loop-syntax question)

---

_Part of the CodePath AI301 open-source contribution assignment._
