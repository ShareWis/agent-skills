---
name: ephemeral-e2e-tests
description: Write, run and prove Playwright E2E specs that execute against the persistent `e2e` ephemeral environment behind the WisdomBase "E2E · release gate" (SWWB-24455). Use when adding or debugging a spec under tests/ephemeral/ in ShareWis/wisdombase-playwright-tests, when a scenario needs new fixture data in db/seeds/custom/e2e.rb in ShareWis/sharewis-act, or when the release gate goes red. Triggers on: ephemeral E2E, release gate, tests/ephemeral, chromium-ephemeral, chromium-camera, E2E_TARGET, incamera gate, exam spec, CustomSeeds::E2e.
---

# Ephemeral E2E Tests (release gate)

Specs that run against `e2e` — a **persistent, named** ephemeral environment — as part of a
**manually-triggered, non-blocking** pre-production release gate. A human runs the gate from
GitHub Actions before approving a production release; it reports the SHA it actually tested and
never blocks the pipeline.

This is a **different world** from the Staging suite in the same repo. Staging specs are fixtured
by two AWS seeding/cleaning Lambdas and registered in `TEST_SCENARIO_MAP`. Ephemeral specs are
fixtured by a Rails seeder in the app repo and are selected **by their directory**, not by
registration. Do not carry Staging habits across; see @rules/spec-anatomy.md.

## The three repos

| Purpose | Repo |
| --- | --- |
| Playwright suite (the spec) | `ShareWis/wisdombase-playwright-tests` |
| Rails app, fixture seeder, gate workflow | `ShareWis/sharewis-act` |
| Specs and plans (docs) | `ShareWis/wisdombase-documents` |

Adding a spec that needs new fixture data is a **two-repo job with two separate PRs**. The suite
repo is self-contained (its own `package.json` / `playwright.config.ts`); the app repo is
Ruby 2.7 / Rails 6.0 / Aurora MySQL 8.0 / HAML, where **all** commands go through
`./scripts/worktree_compose.sh` — read its `AGENTS.md` before touching it.

The canonical long-form guide lives in the suite repo at `docs/EPHEMERAL_GUIDE.md`. Read it when
you need the full walkthrough; this skill is the operating contract.

## Non-negotiables

1. **`e2e` is PERSISTENT.** State accumulates across gate runs. If your spec completes a lecture,
   takes an exam, or mutates any record, the seeder must reset it — or the *second* run fails.
   @rules/persistent-env-state.md
2. **Prove it passes twice consecutively.** A single green run is not evidence on a persistent
   env. This is the real bar for "done".
3. **Never paper over an app bug in test code.** Two real defects were found exactly this way
   (SWWB-24509, SWWB-24510). If you must work around one, say so explicitly in a code comment
   and file a Jira ticket in project SWWB. A green suite hiding a real defect is worse than a
   red one.
4. **Import `test` from `helpers/runtime-guard`, never from `@playwright/test`.** The guard is a
   `{ auto: true }` fixture that fails a spec on unexpected console errors and failed requests,
   and it is **context-scoped** so it also observes the exam popup. A page-scoped guard silently
   watches the wrong page.
5. **Assert a user-visible OUTCOME**, not intermediate UI state. "The course reports every
   lecture complete" — not "the button was clicked".
6. **Use `:visible` locators.** Desktop and mobile partials both render into the same DOM
   (e.g. `.btn-submit-button`), so `.first()` picks the hidden mobile one and the click waits out
   its full timeout against an element no user can see.
7. **Never call the Staging seeding Lambdas from an ephemeral run.** `config/env.ts` couples the
   app URL and the seeding path to a single `E2E_TARGET` entry deliberately, and `global-setup.ts`
   has two guards that abort rather than let you mutate Staging. Do not loosen them.

## How a spec gets picked up — location, not registration

From `playwright.config.ts`:

```ts
const EPHEMERAL_ONLY  = /tests\/ephemeral\//;
const INCAMERA_SPECS  = /tests\/ephemeral\/.*incamera.*\.spec\.ts/;
```

- Put the spec in **`tests/ephemeral/`** → runs under the `chromium-ephemeral` project.
- Name it **`*incamera*.spec.ts`** → runs under `chromium-camera` instead, which adds the
  fake-camera launch args plus explicit `camera`/`microphone` permissions.

No workflow or config edit is needed. Merge to the suite repo's `master` and the next gate run
includes it — the gate checks that repo out fresh every run. Do **not** add the spec to
`TEST_SCENARIO_MAP`: an unknown scenario aborts the entire Staging suite in `global-setup.ts`,
which is precisely why these specs are isolated by directory.

Both ephemeral projects carry the fake camera, because the seeded exam is incamera-enabled on the
org and the exam, so the pre-exam gate stands in front of it regardless. `chromium-ephemeral` is
therefore **not** a camera-free project and cannot prove camera-free behaviour.

## Workflow

**Step 1 — read the two existing specs.** They are the pattern, and they encode hard-won
detail in their comments:

- `tests/ephemeral/exam-take.spec.ts` — login → clear the incamera gate → open the exam popup →
  answer every section → submit → assert the course reports all lectures complete.
- `tests/ephemeral/exam-incamera.spec.ts` — same up to the exam, then asserts the in-exam loop
  POSTs a frame to `/users/incamera_upload_image` and the response is `< 400`.

**Step 2 — decide whether you need new fixture data.**

- *Existing fixtures suffice* (course `free-exam-course`; learners `paid_member_01`, `04`, `05`
  `@example.com` are unclaimed — `02` and `03` are already taken by the two specs above) → suite
  repo PR only.
- *You need more* → the fixture goes in `db/seeds/custom/e2e.rb` (`CustomSeeds::E2e`) **with
  cleanup**, plus a matching case in `spec/seeds/e2e_custom_seed_spec.rb`, as a **separate app
  repo PR**. @rules/fixtures-in-app-repo.md

**Step 3 — write the spec.** Start from `templates/ephemeral-test-template.spec.ts` in the suite
repo. Prefer the existing helpers over re-implementing:

| Helper | Exports |
| --- | --- |
| `helpers/runtime-guard.ts` | `test`, `expect` (auto guard fixture), `IGNORED_CONSOLE`, `IGNORED_REQUESTS` |
| `helpers/incamera.ts` | `passIncameraGate`, `openExamLecture`, `onIncameraGate`, `gateLectureId` |
| `helpers/exam.ts` | `startExam`, `answerVisibleQuizzes`, `completeExam`, `courseCompleted` |
| `helpers/test-helpers.ts` | `TestHelpers.login` (and the rest of the Staging toolkit) |
| `config/env.ts` | `config.app.baseUrl`, `config.testData.*`, `config.waits.*` |

Details and an annotated skeleton: @rules/spec-anatomy.md

**Step 4 — run it, twice.** @rules/running-and-proving.md

```bash
# from the root of your wisdombase-playwright-tests clone
E2E_TARGET=ephemeral npx playwright test --project=chromium-ephemeral
```

Camera specs **cannot** be developed on macOS — any `getUserMedia` touching audio never settles
under `--use-file-for-fake-video-capture`. Use the Linux container recipe in
@rules/running-and-proving.md. A local run also needs your IP already on the `e2e` allowlist.

**Step 5 — ship.** Separate PRs per repo. Commit with **no git identity flags** — let the repo
config apply.

## Definition of done

- [ ] Spec lives in `tests/ephemeral/`, uses the runtime-guard `test` fixture and the existing
      helpers.
- [ ] Asserts a user-visible outcome, not intermediate UI state.
- [ ] **Passes twice consecutively** against `e2e`.
- [ ] Any new fixture data has a `db/seeds/custom/e2e.rb` change WITH cleanup, plus a
      `spec/seeds/e2e_custom_seed_spec.rb` case, as a separate app-repo PR.
- [ ] Any app bug discovered is filed in Jira (SWWB), not silently worked around.
- [ ] Separate PRs per repo, no git identity flags on the commits.

## Rules

- @rules/spec-anatomy.md — structure, imports, helpers, locator and popup semantics
- @rules/persistent-env-state.md — what a persistent env breaks, and the prove-twice bar
- @rules/fixtures-in-app-repo.md — `CustomSeeds::E2e`, idempotency, cleanup, hand-reseeding
- @rules/running-and-proving.md — local runs, the macOS camera limitation, triggering the gate
