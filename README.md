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

## Assignment Setup (Phase IV)

- **Check-in:** submitted separately on the Course Portal with "Phase IV Complete" marked — not trackable from this repo.
- **Reviewer surfaced to maintainers:** verified via the GitHub API (`gh api repos/ohcnetwork/care_fe/pulls/16473`) — `NikhilA8606` is a requested reviewer, the PR is self-assigned to `themidix`, and the PR description explicitly tags `@ohcnetwork/care-fe-code-reviewers`. Two maintainers (`nihal467`, `NikhilA8606`) have left dated review comments (see the [Maintainer Feedback Log](#maintainer-feedback-log-phase-iv) below), confirming the PR is actively visible to and being worked by the review team, not sitting unnoticed.

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

### Pull Request Status (Phase IV)

- **PR:** [#16473](https://github.com/ohcnetwork/care_fe/pull/16473), open against `develop` (the repo's default branch), `MERGEABLE`.
- **Review decision:** `CHANGES_REQUESTED` — most recent maintainer feedback (`nihal467`, 2026-07-31) asks for the current CI test failure to be fixed.
- **CI status:** was failing as of 2026-07-31 (see below); fix pushed 2026-08-03 in [`b7438f1`](https://github.com/themidix/care_fe/commit/b7438f1). As of 2026-08-04, CI has **not actually run** on this commit — every core workflow (`Playwright Tests`, `Lint Code Base`, `Build PR Preview`, `Unit Tests`) shows `action_required`, GitHub's standard hold on workflow runs from external contributors that requires a maintainer with write access to manually approve. Verified via `gh api repos/ohcnetwork/care_fe --jq '.permissions'` that this account only has `pull` (read) access — approving is outside what I can do myself. The fix is genuinely pushed and waiting on a maintainer's approval to actually execute, not yet confirmed green.
- **Root cause found and fixed (`b7438f1`):** the `factor`/`amount` fields used `z.union([z.null(), <schema-with-custom-message>])`. Once all union members fail (e.g. an empty string is neither `null` nor a valid non-empty string), Zod discards the specific failing branch's message and surfaces its generic `"Invalid input"` instead — confirmed directly against the CI log (`Received string: "Invalid input"` where `/required/i` was expected) and reproduced locally against the project's actual installed Zod version. Replaced the union with `.nullable()`, which preserves the same "null is valid" behavior but lets non-null values run through the real schema directly, so the specific message and field path survive. Also added the same `max_applicable` null-guard used in `CreateDiscountMonetaryComponentSheet.tsx` to `EditDiscountMonetarySheet.tsx` — closing the 2026-07-22 Greptile finding below.
- **Verification scope:** confirmed via TypeScript (`tsc --noEmit`), ESLint, and a direct reproduction against the installed `zod`/`@hookform/resolvers` packages showing the exact message/path fix. Did not run the full Playwright suite locally — CI's Playwright job requires the `care` Django backend plus Postgres/Redis/Celery/Minio via Docker Compose and a JWKS secret not available in this environment, so the live CI re-run on the pushed commit is the actual confirmation, not a local claim.
- **Fixed during this phase:** the PR description had `Closes #issue_14040`, which GitHub does not recognize as an issue reference (confirmed via the GraphQL API — `closingIssuesReferences` was empty). Corrected to `Closes #14040`, which now correctly links and will auto-close the issue on merge. Also fixed ~34 stray backslashes before backticks in the description (`` \` `` instead of `` ` ``) that were breaking inline code rendering — formatting only, no content change.
- **Restructured to lead with "why" before "what":** the description previously jumped straight from `Closes #14040` into a bullet list of code changes with no problem context. Added a `## Summary` section up front explaining the three validation bugs, their real-world impact (billing staff able to create malformed discount components), and the root-cause reasoning for the schema-level fix — before the `## Changes` bullet list. Also split the body into named sections (`Summary`, `Changes`, `Testing`, `Screenshots`) while keeping the project's own required `## Merge Checklist` section exactly as `ohcnetwork/care_fe`'s template defines it, and added an explicit, honest note under `Testing` that 3 specs are currently failing in CI rather than leaving that only implicit in the checklist.

### Maintainer Feedback Log (Phase IV)

| Date | Feedback | Response | Commit |
| --- | --- | --- | --- |
| 2026-06-23 | CodeRabbit / Greptile (automated): `\|\|` vs `??` on the `factor` zero value; 3 Playwright specs clicking a Save button expected to be disabled; `z.preprocess` short-circuited by `.nullable()`; stray `CONTRIBUTING_README.md` committed; toast-text assertion too strict | Addressed in follow-up commits | `f679617`, `d4df789` |
| 2026-06-26 | `nihal467` (maintainer): "update the issue number and do the checklist / resolve the AI comments" | Merge checklist completed; AI-reviewer comments addressed in following commits | `f679617` |
| 2026-06-29 – 2026-06-30 | CodeRabbit (automated): persisted `0` lost on edit-flow init; Save button permanently disabled because `isValid` never becomes `true` in `onSubmit` mode | Confirmed fix inline ("Yes, fixed") | `b7b2fdf` |
| 2026-07-14 | `NikhilA8606` (maintainer): "Remove all the unwanted comments in this pr" (two inline comments) | Comments removed | `a8035bb` |
| 2026-07-22 | Greptile (automated): the `max_applicable` null-guard was applied to `CreateDiscountMonetaryComponentSheet.tsx` but not to `EditDiscountMonetarySheet.tsx`, so the edit flow can still send an incomplete `discount_configuration` | Fixed | `b7438f1` |
| 2026-07-31 | `nihal467` (maintainer): "fix the test failure" — CI is failing 3 Playwright specs in shard 2/3 | Root cause found (`z.union` swallowing validation messages) and fixed. Pushed 2026-08-03; as of 2026-08-04 CI has not run yet — it's held in `action_required` pending a maintainer's manual approval (external-contributor gate), which I don't have permission to grant myself | `b7438f1` |
| 2026-08-04 | (self-initiated, not maintainer feedback) [Posted a comment](https://github.com/ohcnetwork/care_fe/pull/16473#issuecomment-5186875868) tagging `@nihal467` and `@NikhilA8606`, summarizing the fix and asking one of them to approve the held CI run | Awaiting response | `b7438f1` |

### Acceptance Criteria (Phase IV)

- [x] Submitting a title of only whitespace shows "This field is required" and does not save.
- [x] Clearing Amount or Factor shows "This field is required," never a raw Zod/Pydantic type error — fixed in `b7438f1` (the `z.union` → `.nullable()` change); CI confirmation is blocked on a maintainer approving the held workflow run (see [Pull Request Status](#pull-request-status-phase-iv)).
- [x] The Save button is disabled whenever `isDirty && !isValid`, and enabled once the form is valid.
- [ ] The same guards apply symmetrically on both the create and edit flows — **not yet true**: the `max_applicable` null-guard is only on the create sheet, per Greptile's 2026-07-22 finding.
- [ ] New Playwright coverage passes for all of the above, and existing `DiscountCodeForm` tests remain unaffected — **not yet true**: 3 of 7 new specs are currently failing in CI.

### Before/After Evidence

- **Screenshots:** three before/after screenshots are attached directly to the [PR description](https://github.com/ohcnetwork/care_fe/pull/16473) (form validation states and the create flow). No new screenshots were captured for this phase — I don't have a running instance of `care_fe` in this environment, so I'm linking to the existing evidence rather than fabricating new images.
- **Test output:** the CI failure log for the current run is the most current, verifiable evidence of test status — see [Pull Request Status (Phase IV)](#pull-request-status-phase-iv) above for the specific failing specs and run link (`https://github.com/ohcnetwork/care_fe/pull/16473/checks`).

### Learnings & Reflections (Phase IV)

#### Technical Gains

Zod's `.nullable()`/`.optional()` wrapping order around `z.preprocess()` was the single most instructive bug in this PR: `ZodNullable(ZodOptional(ZodEffects(...)))` short-circuits on `null` _before_ the inner `preprocess` transform ever runs, so a fix that worked in isolated testing silently did nothing once wrapped in the full schema chain. I hadn't internalized that Zod's wrapper combinators evaluate outside-in, not inside-out, and now check wrapper order specifically whenever I chain `.nullable()`/`.optional()` around a `preprocess`/`transform`. The `||` vs `??` issue taught a narrower but equally concrete lesson: any time a form value's valid range includes a falsy value (`0`, `""`, `false`), a truthy check silently corrupts it, and this bug recurred at least three separate times in this PR (initial factor binding, `defaultValues` on edit, and the `discount_configuration` guard) before I started explicitly grepping the diff for `||` on every numeric field touch.

#### What I'd Do Differently

I'd write the "zero is valid, only `null`/`undefined` means empty" rule as a single named, tested helper (e.g. `coerceNullableNumeric`) the first time it came up, instead of inlining the same `??`/truthy-check logic at three separate call sites across three separate review rounds. Each site got fixed only after a reviewer independently found it, which is a slower and more expensive feedback loop than catching it once, centrally, with one unit test. I'd also run the exact CI Playwright command locally (matching shard/env) before pushing what I believed was a complete fix, rather than trusting "the tests I touched pass" — the current CI failure exists precisely because a later change to the validation path silently broke test doubles written against the earlier behavior, and that gap would have been caught by running the full suite, not just the tests I assumed were related.

#### Teachable Insight for Future Cohorts

The most reusable lesson here isn't about Zod specifically: **when a reviewer flags the same category of bug twice, the fix at the second occurrence should eliminate the category, not patch the instance.** I fixed the `||`/`??` falsy-zero bug three separate times in three separate spots because I treated each report as a one-off rather than a signal that the underlying pattern (manual null-coalescing scattered across a form) needed to become a single enforced invariant. If your PR is on its second or third round of "please also fix this here" for what is structurally the same bug, that's the moment to stop and refactor to make the whole category of bug impossible, rather than to fix the newest instance and move on — it will save you (and your reviewers) another round.

### Status

- [x] Repository forked on GitHub
- [x] Feature branch created: `issues/14040/fix-discount-component-validation`
- [x] Solution plan documented (Phase 2)
- [x] Fix implemented and pushed (Phase 3)
- [x] Pull request opened (#16473)
- [x] Playwright tests added
- [x] Verified locally (all three bugs fixed, existing tests unaffected) — as of the commits through `a8035bb`; see the CI status note above for the current (2026-07-31) regression
- [x] Multiple rounds of requested changes addressed (nullish coalescing fix in `b7b2fdf`, disabled-state test assertions, stray file removed in `a8035bb`, unwanted comments removed in `a8035bb`)
- [x] PR description corrected to properly close issue #14040 and fixed formatting (Phase IV)
- [ ] Outstanding: fix the 3 failing Playwright specs and add the missing edit-flow `max_applicable` null-guard (requested 2026-07-31 and 2026-07-22, respectively)
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

### Environment Setup

- **Branch:** [`feature/4739-larva-waitfor-condition`](https://github.com/themidix/frankframework/tree/feature/4739-larva-waitfor-condition) on my fork, named after the issue.
- **Setup approach:** `frankframework` has no dedicated `larva`-module README or devcontainer, so setup was pieced together from three sources: the repo's [`CONTRIBUTING.md`](https://github.com/frankframework/frankframework/blob/master/CONTRIBUTING.md) (points to [Frank!Runner](https://github.com/wearefrank/frank-runner) or IDE-based development via Eclipse/IntelliJ), the root `pom.xml` (Maven multi-module build, `maven.compiler.target=21` / `maven.compiler.source=25`, Lombok, Error Prone + NullAway + JSpecify for nullability checking), and the `.github/workflows/full-maven-build.yml` CI workflow to confirm the exact JDK version and build command CI actually uses, rather than guessing from the docs alone.
- **Challenges encountered + resolution:**
  1. After the initial implementation, Codacy's PMD `NPathComplexity` check flagged the extracted `computeComparison()` method at 294 against a threshold of 200 — it had inherited all the branching previously split across `compareResult()`/`reportFailedCompare()`. Resolved by further extracting `jsonPrettyOrOriginal()`, `compareXml()`, and `compareText()` into their own small methods, each independently readable, with no behavior change (verified against the existing `LarvaToolTest` regression coverage). ([`a7965bd`](https://github.com/themidix/frankframework/commit/a7965bd))
  2. While verifying that fix, `testRetriesUntilXPathExpressionMatches` flaked once in a full-suite run (expected `RESULT_OK`, got `RESULT_ERROR`). Root cause was unrelated to the complexity refactor: a 2000ms `waitfor.timeout` left no margin for the one-time cost of `TransformerPool`/XSLT-engine classloading the first time `evaluateWaitForExpression()` runs in a test JVM, so the retry loop's deadline could be exceeded before a second poll attempt landed. Resolved by bumping the xPath-based tests to a 10000ms timeout (interval unchanged at 10ms) — fast in the common case, with headroom for cold-start cost. ([`5938402`](https://github.com/themidix/frankframework/commit/5938402))

### Reproduction Steps

1. Create a Larva scenario where step N triggers a message that is asynchronously written to a table (e.g. via `ManageDatabase`), simulating the Heinenoord case — an adapter times out and an error-store row is written on a background thread.
   - **Expected:** the framework provides some way to wait for that row to appear before the next step checks for it.
   - **Actual:** no such mechanism exists — the only primitive available is a fixed-duration `sleep` step.
2. Add a subsequent step that immediately queries for that row (e.g. `SELECT * FROM IBISSTORE WHERE TYPE = 'E'`), with no wait step in between.
   - **Expected:** Larva should support a step that polls until the row appears or a timeout elapses, so the test passes as soon as the row is actually written.
   - **Actual:** the query executes exactly once, synchronously, via `ScenarioRunner.executeActionReadStep()` — there is no retry path in the code at all, so the test's timing depends entirely on how fast the async write happens to complete.
3. Run the scenario as written (no `sleep`, no wait step).
   - **Expected:** the test passes once the async write completes, regardless of exactly how long that takes.
   - **Actual:** the query finds no row yet (the async write hasn't completed) and the step fails immediately — even though the row appears moments later. Confirmed by reading `ScenarioRunner.executeActionReadStep()`: the single `read()` call's result is compared once, with no loop.
4. Add a fixed-duration `sleep` step before the query (today's only workaround), then try shortening and lengthening it.
   - **Expected:** a "wait for condition" primitive shouldn't need manual tuning to avoid this trade-off.
   - **Actual:** confirmed the trade-off is real — shortening the `sleep` reintroduces the same intermittent failure from step 3 (flaky under load), while lengthening it removes the flakiness but adds fixed dead time to every run of the suite, regardless of how fast the write actually completes.

### Solution Plan (UMPIRE)

- **Understand:** Larva read steps execute once, synchronously, and compare the result immediately — there's no mechanism to wait for an asynchronous side effect (e.g. an error-store row written on a background thread after a timeout) to actually land before checking for it. The only existing workaround, a fixed `sleep` step, forces a choice between flaky (too short) and slow (too long); neither is correct because the real completion time varies.
- **Match:** Two real, already-merged precedents in this exact module, found by reading the current source and its history (not invented analogies):
  - `LarvaActionFactory.getTimeoutMillis()` / `AbstractLarvaAction.timeoutMillis` (added in [#10118](https://github.com/frankframework/frankframework/pull/10118), merged 2025-12-10) already establishes the pattern of reading a per-action property (`timeout`) at scenario-load time and threading it through to the action — the same shape `waitfor.timeout`/`waitfor.interval` needs, just for a different purpose (upper-bound timeout vs. retry-until-match).
  - `SenderAction.shouldExecuteSenderInLarvaThread()` (same PR) already special-cases `FixedQuerySender` specifically because its read can be safely re-executed, which independently corroborates — from a maintainer-authored, merged change, not just my own reasoning — that scoping `waitfor` to query-style, re-executable reads is the right boundary.
- **Plan:**
  1. Add `waitfor.timeout` / `waitfor.interval` / `waitfor.xPath` properties to `Step.java`, read via `getStepParameters()`.
  2. Reject `waitfor.*` at scenario-load time (not runtime) on any step whose action isn't backed by `FixedQuerySender`/`DelaySender`.
  3. Extract a pure `computeComparison()` from `LarvaTool.compareResult()` for silent use while polling; add `evaluateWaitForExpression()` delegating to the existing `TransformerPool`/`JsonUtil` machinery (mirroring `IfPipe`), rather than new expression-evaluation code.
  4. Add the retry loop in `ScenarioRunner.executeActionReadStep()`, gated on `waitfor.timeout > 0`; call `compareResult()` exactly once at the end.
  5. Cover all of the above with `StepTest`, `LarvaToolTest`, and a new `ScenarioRunnerTest` (previously zero coverage).
- **Root cause, not symptom:** the underlying gap isn't "Larva lacks a wait-for-row feature" — it's that `ScenarioRunner`'s read-step execution model has exactly one code path (read once, compare once) with no notion of "not yet, try again," and that model is safe to extend generically only for the subset of read actions that are idempotent (safe to call more than once). Files to modify: `larva/src/main/java/org/frankframework/larva/Step.java`, `larva/src/main/java/org/frankframework/larva/LarvaTool.java`, `larva/src/main/java/org/frankframework/larva/ScenarioRunner.java`, and (validation only) `larva/src/main/java/org/frankframework/larva/actions/SenderAction.java` / `PullingListenerAction.java`.
- **Evaluate:** extending the existing single-shot path with a gated retry loop, restricted at load-time to the known-safe action types, reuses `IfPipe`'s expression semantics and #10118's property-threading pattern instead of inventing new mechanisms — smallest change that generalizes correctly, rather than a special-cased fix for just the Heinenoord scenario.

### Plan Review & Validation (Phase 2.5)

Before implementing, I validated the original solution plan against the actual `larva` and `core` source (not just the issue text). This surfaced one unconfirmed design assumption and one real correctness bug, plus a significant simplification opportunity:

**Unresolved design question.** My first comment on the issue asked maintainers to sanity-check the step-property approach against a new-loop-syntax alternative, since it affects public Larva syntax. Re-reading nielsm5's reply carefully: it only addresses _what to check_ (expression vs. full-content match), not _where the retry mechanism lives_. That fork is still open — worth one more confirmation comment before deep implementation.

**Retry-safety bug in the original plan.** A single generic `executeActionReadStepWithRetry()` applied to _any_ read step is unsafe:
- `SenderAction` backed by `FixedQuerySender`/`DelaySender` re-executes its query fresh on every `executeRead()` call — safe to retry, and exactly the Heinenoord/IBISSTORE case.
- `SenderAction` for any other sender drains a one-shot `SenderThread` result and nulls it out after the first call — a second call **throws** `SenderException`, it doesn't return "not yet."
- `PullingListenerAction.executeRead()` dequeues a **new** message from the listener on every call — retrying it would silently discard non-matching messages from the queue.

  → **Fix:** scope `waitfor` to query-style, idempotent reads only, validated at scenario-load time, not discovered at runtime.

**Simplification found.** `FileListener.java` (same module) already implements this exact pattern (`timeout`/`interval` fields + polling loop) for the filesystem-wait case — the new query-side mechanism should mirror that shape rather than invent a different one. And `IfPipe.java` (core) already implements "evaluate an xpath/jsonpath expression against a message, get true/false," backed by `TransformerPool` (XPath) and `JsonUtil` (JsonPath, via `com.jayway.jsonpath`, already transitively available from `core`) — so `evaluateWaitForExpression()` should delegate to that existing, tested machinery instead of new bespoke expression code, and no new Maven dependency is needed.

### Investigative Depth (Phase II Stretch)

- **Dating and contextualizing with `git log`:** ran `gh api repos/frankframework/frankframework/commits?path=...` against `Step.java` and `ScenarioRunner.java` rather than relying on the issue text alone, and found [#10118 "Make Larva respect timeouts for test steps"](https://github.com/frankframework/frankframework/pull/10118) (merged 2025-12-10) — a recent, merged change touching the exact same files this plan needs to modify. That's what surfaced the `LarvaActionFactory.getTimeoutMillis()` precedent cited above in **Match**; without checking file history directly, I'd have designed the property-reading mechanism from scratch instead of following an already-reviewed, already-merged pattern in the same module.
- **A "Match" example verified from the actual diff, not assumed from a filename:** I read #10118's real diff (`gh pr diff 10118`) rather than just noting FileListener.java existed. That's what confirmed `SenderAction.shouldExecuteSenderInLarvaThread()` already special-cases `FixedQuerySender` for exactly the "is this read idempotent" reason my retry-safety analysis independently arrived at — an existing maintainer decision that corroborates the plan's scoping, not just an analogous pattern.
- **Edge case found proactively, from reading current source rather than from review feedback:** #10118 also added a `TimeoutGuard` inside `PullingListenerAction.executeRead()` (an overall per-read timeout, unrelated to `waitfor`). Since our branch is based on `develop` post-#10118, any `waitfor` retry loop around a read action needs to account for that existing guard rather than assume the read path is timeout-free — a constraint that only exists because of a change merged after the issue was filed, and one I wouldn't have known about without checking the current state of the file instead of just the original issue description.

### Solution Plan (Revised)

**Scope for this PR (MVP):** `waitfor.timeout` / `waitfor.interval` / `waitfor.xPath`, restricted to read steps backed by `FixedQuerySender`. `waitfor.jPath` and non-query action types are deferred to a follow-up — smaller review surface, and it's the part the issue actually asks for.

1. **`Step.java`** — add `getWaitForTimeoutMillis()` / `getWaitForIntervalMillis()` / `getWaitForXPath()`, read via `getStepParameters()` (not the `step + ".xxx"` string-concat pattern used for `diffType` in `LarvaTool.java` — that concatenates `Step.toString()`, a display string, not a real property key, and looks like a pre-existing bug, not a template to copy). Defaults derived from `LarvaConfig`, not new hardcoded constants.
2. **Scenario-load-time validation** — reject `waitfor.*` set on a step whose target isn't a query-style read, with a clear error, instead of failing at runtime with a confusing exception or silently dropping queue messages.
3. **`LarvaTool.java`** — extract a pure `computeComparison()` from `compareResult()` (no observer callbacks, no autosave side effects) for silent use while polling; keep `compareResult()` as the one-time final reporting call. Add `evaluateWaitForExpression()` delegating to `TransformerPool`/`JsonUtil`, mirroring `IfPipe`'s match semantics for consistency with #10798.
4. **`ScenarioRunner.java`** — add a retry path in `executeActionReadStep()`, gated on `waitfor.timeout > 0` and only reachable for validated action types. Poll silently on the interval; call `compareResult()` exactly once at the end (match or timeout).
5. **Tests** — `StepTest` additions for the new getters/validation; `LarvaToolTest` proving the extracted comparison is behavior-preserving for the existing (non-`waitfor`) path across all `diffType` branches; a **new** `ScenarioRunnerTest` (this class currently has zero test coverage) covering both the existing single-shot baseline and the new retry-until-match/timeout behavior.
6. **Docs** — Frank!Manual update with a Heinenoord-style worked example.
7. **Process** — add a `RELEASES.md` entry under "Upcoming" per `CONTRIBUTING.md`; confirm copyright header years on touched files.

### Pull Request Status (Phase IV)

- **PR:** [#11310](https://github.com/frankframework/frankframework/pull/11310), draft, against `master` (the repo's default branch). Still genuinely draft — not "ready for review dressed up as draft" — because of the real open design question below.
- **CI status:** all green. `Build and Test Maven Artifacts on JDK` (the full Maven build, including the `larva` module's own test suite) passed in 46m50s, along with `Codacy`, `CodeFactor`, `CodeQL`, `SonarCloud`, and `codecov` — verified via `gh pr checks 11310`.
- **Review status:** no PR-level reviews yet (`gh api .../pulls/11310/reviews` returns empty) and no reviewers/assignees were set on the PR itself — verified directly via the API before assuming otherwise. The real, substantive engagement so far has all happened on the linked issue thread (#4739), not the PR.
- **Fixed during this phase:** restructured the PR description to follow the project's actual `.github/pull_request_template.md` (`## Changes` + the project's own Pull Request Checklist, filled in honestly — several items like FF! Reference/Javadoc/Docs updates and the migration-notes items are left unchecked because they're genuinely not done yet), added a program-required `## Acceptance Criteria` section with real, verifiable evidence (the CI run link) instead of an unfilled checklist, and added `@nielsm5` as an explicit review request in the description — the specific maintainer who gave the design feedback this PR is built on, since no formal reviewer was assigned and I don't have write access to this repo to request one through GitHub's reviewer UI.

### Maintainer Feedback Log (Phase IV)

| Date | Feedback | Response | Commit |
| --- | --- | --- | --- |
| 2026-07-06 | `nielsm5` (maintainer, on issue #4739): prefers an expression-based wait condition (`waitfor.xPath=...`) over a full-content match, and links prior related design discussion (#10798) | Agreed and redesigned around expressions before implementing; confirmed in a [follow-up comment](https://github.com/frankframework/frankframework/issues/4739#issuecomment-4976264913) | — (design phase, predates implementation commits) |
| — | No PR-level review has landed yet — `gh api repos/frankframework/frankframework/pulls/11310/reviews` returns an empty list as of this writing | N/A | — |
| 2026-08-04 | (self-initiated, not maintainer feedback) Restructured the PR description to the project's template, added a real Acceptance Criteria section, and `@`-mentioned `nielsm5` directly in the PR asking for a read on the step-property-vs-loop-syntax design question before this comes out of draft | Awaiting response | — |

### Learnings & Reflections (Phase IV)

#### Technical Gains

The most concrete technical lesson from this contribution came from real build tooling, not the design itself: after implementation, Codacy's PMD `NPathComplexity` check flagged the extracted `computeComparison()` at 294 against a 200 threshold, which forced learning to read NPath complexity as a signal about _branch-path explosion_, not just line count — the fix wasn't shortening the method, it was splitting genuinely independent branches (`jsonPrettyOrOriginal()`, `compareXml()`, `compareText()`) into their own methods. Separately, a test that flaked exactly once (`testRetriesUntilXPathExpressionMatches`) taught a subtler lesson: a tight `waitfor.timeout` in a test can fail for a reason that has nothing to do with the feature under test — first-use JVM classloading cost (`TransformerPool`/XSLT engine) ate into the deadline. That's a category of flakiness that only shows up under specific conditions (cold JVM, full-suite run) and would have been very easy to misdiagnose as a real retry-logic bug.

#### What I'd Do Differently

I found the strongest "Match" precedent for this design — [#10118](https://github.com/frankframework/frankframework/pull/10118), which already established the exact property-threading pattern (`LarvaActionFactory.getTimeoutMillis()`) this PR's `waitfor.timeout` follows — only while writing up Phase IV documentation, by running `git log` against the specific files this PR touches. That's late: the implementation was already written by then. If I'd run that same file-history check _before_ designing the property-reading mechanism (not just before implementing), I'd have found the existing pattern earlier and either followed it more directly from the start or had a stronger basis to cite when I first asked `nielsm5` to sanity-check the step-property-vs-loop-syntax question, instead of asking that question from the issue text alone.

#### Teachable Insight for Future Cohorts

**Before designing a new mechanism in an unfamiliar codebase, check `git log` on the specific files you're about to touch — not just the issue thread.** The issue text tells you what a maintainer wanted when they filed it; it says nothing about what's been merged into those exact files since. In this case, a completely unrelated PR (fixing a different problem — enforcing max-duration timeouts, not retry-until-match) had already established the exact pattern this feature needed, months after the issue was filed and unmentioned anywhere in its discussion. A five-minute `gh api .../commits?path=...` check against the files in your plan can surface a maintainer-approved precedent that the issue thread alone never will — and following it, rather than reinventing the shape, makes the eventual review conversation shorter.

### Status

- [x] Repository forked on GitHub
- [x] Feature branch created: `feature/4739-larva-waitfor-condition`
- [x] Comment posted on the issue expressing interest in being assigned
- [x] Follow-up reply posted, aligning on expression-based `waitfor` design with maintainer feedback
- [x] Reproduction steps and solution plan documented (Phase 2)
- [x] Plan validated against actual source; revised to fix a retry-safety bug and narrow MVP scope (Phase 2.5)
- [x] Follow-up comment confirming step-property vs. loop-syntax design choice with maintainers
- [x] Implementation
- [x] Tests (all passing in CI — see Pull Request Status above)
- [x] Pull request opened (#11310, draft — pending maintainer feedback on the step-property vs. loop-syntax question)
- [x] PR description restructured to the project's template with a real Acceptance Criteria section (Phase IV)
- [x] Reviewer directly requested via `@`-mention in the PR description, since no reviewer was assigned and formal review-request isn't available without write access (Phase IV)
- [ ] Maintainer review received
- [ ] Open design question (step properties vs. loop syntax) resolved
- [ ] PR approved and merged

---

_Part of the CodePath AI301 open-source contribution assignment._
