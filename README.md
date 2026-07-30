# Open Source Contribution Log

Tracking issues tackled, implementation plans, and pull requests across multiple open-source projects — CodePath AI301.

## Phase I Submission

### Setup Verification

- **GitHub account:** [github.com/themidix](https://github.com/themidix)
- **Forked project:** [themidix/care_fe](https://github.com/themidix/care_fe) — fork of [ohcnetwork/care_fe](https://github.com/ohcnetwork/care_fe)
- **Contribution README repo:** [themidix/ai301-open-source-contribution](https://github.com/themidix/ai301-open-source-contribution) (this repo, public)

### Issue Selection

- **Issue:** [ohcnetwork/care_fe#14040 — Discount Monetary Component form validation bugs](https://github.com/ohcnetwork/care_fe/issues/14040)
  - Status at time of selection: **open**, unassigned, labeled `good first issue`, no linked PR from another contributor.
  - Project health: `care_fe` is an actively maintained project (commits merging weekly), has a working `README.md` with local dev setup instructions, and the issue itself is a self-contained bug report with no unresolved sub-tasks.
- **Introduction comment:** _pending — see note below._

> **Note:** I have not yet posted an introduction comment on the issue thread itself (the issue currently has zero comments). Draft comment, ready to post from my own GitHub account:
>
> > Hi! I'm working through this issue as part of a course open-source contribution assignment. I've reproduced all three validation bugs described above (whitespace-only title, raw Zod error on cleared Amount/Factor, Save enabled on an invalid form) in `DiscountMonetaryComponentForm.tsx` and have a fix + Playwright coverage ready. Opening a PR shortly — happy to adjust based on any maintainer feedback.

### Why I Chose This Issue

_Draft — personalize before submitting:_

I picked this issue because it sits squarely in a stack I already know — React, TypeScript, `react-hook-form`, and `zod` — so I could focus on the mechanics of a real-world OSS workflow (forking, branching, PR review) instead of fighting an unfamiliar framework. At the same time, form-validation edge cases (whitespace bypass, `null` vs `""` handling in schema chains, gating a submit button on `isValid` and not just `isDirty`) are a pattern I run into often and wanted to get sharper on. `care_fe` is also a large, actively maintained healthcare EMR codebase, which gave me a chance to practice reading and navigating a production-scale repo I didn't write, rather than a toy project.

### Problem Summary (Issue #14040)

The Create Discount Component form (`DiscountMonetaryComponentForm.tsx`) in `care_fe` has three related validation bugs: a whitespace-only title (`"   "`) passes the `min(1)` check and gets saved as if it were valid, clearing the Amount or Factor field surfaces Zod's raw internal type error instead of a friendly "This field is required" message, and the Save button stays clickable even when the form is in a known-invalid state. This matters because it lets hospital billing staff create malformed discount components (blank titles, missing amounts) and exposes a confusing, non-localized error message instead of guided validation feedback. I chose it because the root cause was traceable to a single file, the fix pattern (`.trim()`, `z.preprocess`, an `isValid` gate) was already precedented elsewhere in the codebase (`DiscountCodeForm.tsx`), and it let me ship a complete, testable fix within the scope of Phase I.

### Check-in

- [ ] Check-in form submitted on the Course Portal with "Phase I Complete" marked — _submit separately on the course portal; not trackable from this repo._

### Stretch / Bonus

- **Files/modules involved:** `src/pages/Facility/settings/billing/discount/discount-components/DiscountMonetaryComponentForm.tsx` (schema + Save button logic), with the working `.trim()` pattern already precedented in the sibling `src/pages/Facility/settings/billing/discount/discount-codes/DiscountCodeForm.tsx`, and a new Playwright spec at `tests/facility/settings/billing/discount/discountMonetaryComponentCreate.spec.ts`.
- **Maintainer/related links:** [PR #16473](https://github.com/ohcnetwork/care_fe/pull/16473) review feedback from `nihal467` and `NikhilA8606`; automated review from CodeRabbit/Greptile flagging the `??` vs `||` nullish-coalescing bug and a `max_applicable` null-guard gap on the edit flow.
- **What "fixed" looks like (acceptance criteria):**
  1. Submitting a title of only whitespace shows "This field is required" and does not save.
  2. Clearing Amount or Factor shows "This field is required," never a raw Zod/Pydantic type error.
  3. The Save button is disabled whenever `isDirty && !isValid`, and enabled once the form is valid.
  4. The same guards apply symmetrically on both the create and edit flows (currently the edit flow still needs the `max_applicable` null-guard per Greptile's review).
  5. New Playwright coverage passes for all of the above, and existing `DiscountCodeForm` tests remain unaffected.

---

## Contribution Index

| Project | Issue / PR | Status | Link |
|---|---|---|---|
| care_fe (ohcnetwork) | [#14040 — Discount component validation bugs](https://github.com/ohcnetwork/care_fe/issues/14040) → [PR #16473](https://github.com/ohcnetwork/care_fe/pull/16473) | Changes Requested | [View PR](https://github.com/ohcnetwork/care_fe/pull/16473) |
| Frank!Framework | [#4739 — Larva wait-for-condition mechanism](https://github.com/frankframework/frankframework/issues/4739) → [PR #11310](https://github.com/frankframework/frankframework/pull/11310) | Draft PR — Awaiting Design Feedback | [View PR](https://github.com/frankframework/frankframework/pull/11310) |

---

## Contribution 1: care_fe (OHC Network)

**Repository:** [ohcnetwork/care_fe](https://github.com/ohcnetwork/care_fe)

**Issue:** [#14040 — Discount Monetary Component form validation bugs](https://github.com/ohcnetwork/care_fe/issues/14040)

**Pull Request:** [#16473 — fix: correct validation bugs in DiscountMonetaryComponentForm](https://github.com/ohcnetwork/care_fe/pull/16473)

**Branch:** [`themidix:issues/14040/fix-discount-component-validation`](https://github.com/themidix/care_fe/tree/issues/14040/fix-discount-component-validation)

### Problem Summary

The Create Discount Component form (`DiscountMonetaryComponentForm.tsx`) had three related validation bugs:

1. **Whitespace-only title bypass** — `z.string().min(1)` counts raw characters, so a title of `"   "` passed validation instead of being rejected as empty.
2. **Raw Zod type error surfaced to the user** — clearing the Amount or Factor input passes `null` into a `z.string()` schema chain, which throws Zod's internal `"Expected string, received null"` error before any custom, user-friendly message can run.
3. **Save button enabled on an invalid form** — the button was only gated on `!isDirty` (was a field touched?), not on whether the form was actually valid, so it could be clicked while validation errors were still showing.

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

### Implementation Notes (Phase 3)

What was implemented this phase:
- ✅ Added `.trim()` to the `title` field schema — whitespace-only input now correctly triggers "This field is required"
- ✅ Wrapped numeric fields with `z.preprocess(val => val ?? "")` — empty Amount/Factor now shows "This field is required" instead of the raw Zod type error
- ✅ Fixed the Save button to be disabled when the form is invalid (`!isDirty || !isValid`)
- ✅ Added a new Playwright test spec covering all validation scenarios

Testing notes:
- 🧪 All three bugs verified fixed locally
- 🧪 Existing `DiscountCodeForm` tests unaffected
- 🧪 New Playwright spec covers whitespace title, empty numeric fields, and Save button state

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
- [ ] Requested changes addressed (nullish coalescing fix, disabled-state test assertions, remove stray file)
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
