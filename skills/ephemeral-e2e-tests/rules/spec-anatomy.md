# Spec anatomy

How an ephemeral spec is put together, and the runtime semantics that make the difference
between a spec that reports the truth and one that times out for reasons unrelated to the app.

## Imports — the guard replaces `@playwright/test`

```ts
import { test, expect } from "../../helpers/runtime-guard";
import { TestHelpers } from "../../helpers/test-helpers";
import { config } from "../../config/env";
import { openExamLecture } from "../../helpers/incamera";
import { startExam, completeExam, courseCompleted } from "../../helpers/exam";
```

`helpers/runtime-guard.ts` re-exports `expect` unchanged and extends `test` with a
`{ auto: true }` fixture. Importing `test` from `@playwright/test` in an ephemeral spec silently
drops the guard — the spec still runs, and console errors and 5xx responses stop failing it.

### The guard is context-scoped, deliberately

It attaches its listeners to `context`, watching `context.pages()` plus every future
`context.on('page')`. The exam submits its form with `target: '_blank'` unless the org opens it
in the same tab, so the exam arrives as a **popup**. A page-scoped guard would watch the tab the
user leaves behind and report "no runtime errors" for a page it never observed.

### Its ignore lists are blind spots

`IGNORED_CONSOLE` and `IGNORED_REQUESTS` in `helpers/runtime-guard.ts` suppress CSP Report-Only
output, GA/GTM noise, Stripe telemetry (`m.` / `r.stripe.com` only — Stripe's API and JS are NOT
ignored), favicon 404s, and the Zendesk widget's own 30s watchdog message. Every entry is a place
the guard cannot see. Adding one requires a justification comment naming what you observed and
where; do not add an entry to make your own spec go green.

## Structure

`test.describe` with a single focused test is the norm. Pin the fixture identifiers as
module-level constants with a comment saying where they come from, so a seeder change has an
obvious blast radius:

```ts
const COURSE_URL = "free-exam-course";
// A member, not config.testData.defaultUserEmail (an org admin) — this exercises the learner
// path. Seeded by SampleDataSeeder#create_org_member_users! for org-paid.
const LEARNER_EMAIL = "paid_member_04@example.com";
```

### Claim your own learner account

`org-paid` is seeded with `paid_member_01@example.com` … `paid_member_05@example.com` (students,
password `config.testData.defaultUserPassword`). `02` and `03` are already claimed by
`exam-incamera.spec.ts` and `exam-take.spec.ts` respectively — **deliberately different**,
because both drive the same seeded exam and taking it mutates that user's state, so a shared
account makes the specs interfere whenever they run together. Pick an unclaimed index and say so
in a comment. Need a sixth? That is a seeder change (@rules/fixtures-in-app-repo.md).

Do not use `config.testData.defaultUserEmail` for learner flows — on the ephemeral target that
resolves to `paid_org_admin@example.com`, an org admin.

### Log in explicitly, per test

Playwright creates a fresh context per test, so there is no shared session:

```ts
await page.goto("/users/sign_in");
await TestHelpers.login(page, LEARNER_EMAIL, config.testData.defaultUserPassword);
```

Relative paths work: `playwright.config.ts` sets `baseURL` from `config.app.baseUrl`, which
follows `E2E_TARGET`. Never hardcode a host — that is what decoupled the browser from the
seeding path once already.

## Locators: `:visible` is load-bearing

The exam renders **both** desktop and mobile partials into the same DOM
(`user_exams/answer_statuses/_side_menu`, `_mobile_info_and_btn`,
`_mobile_incamera_info_and_btn` all emit `.btn-submit-button`). A bare `.first()` takes whichever
comes first in source order — usually the hidden mobile one — and the click then waits out its
full 60s action timeout against an element no user can see. This reads as a hang, not a failure.

```ts
const finish = exam.locator("a.btn-complete-section:visible, a.btn-submit-button:visible");
```

## Popups and closing tabs

- `startExam(page, context)` returns the page the exam actually runs on: it races
  `context.waitForEvent("page", { timeout: 15000 })` against the click and falls back to the
  current page when no popup appears, because same-tab is an org setting rather than a constant.
- **Submitting an exam closes its tab.** Assert the outcome on the **opener**
  (`courseCompleted(page, COURSE_URL)`), or wait on `exam.waitForEvent("close")`. Any call on a
  closed page throws `Target page, context or browser has been closed`.
- Finishing is a **two-stage UI**: both `終了` (`.btn-complete-section`) and `テストを終了する`
  (`.btn-submit-button`) merely *open* a remodal confirm — neither submits. While the remodal is
  up its overlay covers the page, so clicking the other control just waits out its timeout
  against a covered element. `completeExam` already handles this; do not reimplement it.

## The incamera gate stands in front of every exam

The seeded exam is incamera-enabled on both the org and the exam, so
`LecturesController#show` redirects every visit to the lecture through
`/lectures/<id>/incamera_check`. `openExamLecture(page, courseUrl)` walks the course page, finds
the lecture link, navigates, and clears the gate if it appears.

`passIncameraGate` asserts on the **cookie**, not on leaving the page, and that choice is
load-bearing: the gate **advances on failure too** — after 10 attempts at 500ms it sets
`authenticatedIncamera_<lectureId>` to `"false"` and shows a denied notice. "We reached the exam"
would pass with a broken fixture; `cookie === "true"` would not. If you write a new gate-adjacent
spec, keep that property.

Passing the gate does not move the browser — the page stays put and relabels its button into a
link back to the lecture. Navigate explicitly.

## Assert outcomes

The failure a release gate exists to catch is user-visible. Prefer:

- the course page reporting `<done>/<total>` with `done === total` (`courseCompleted` — locale
  independent, which is why it reads the ratio rather than Japanese copy)
- a response status on a real endpoint (`< 400` on `POST /users/incamera_upload_image`)
- a record visible in the UI afterwards

over "the element became visible" or "the click happened".

Correctness of *content* is usually not the point: `answerVisibleQuizzes` picks the first option
in every radio group on purpose, so the spec does not become a second fixture to maintain in
lockstep with the seeded answer key.

## Timeouts

Inherit them. `playwright.config.ts` sets test 240s / expect 90s / action 60s / navigation 90s,
and `config.waits.*` holds the tuned values (`saveResponseMs` 45s, `tabPanelMs` 30s, …). Bound
your own loops so a UI change becomes a clear failure rather than a hang — `completeExam` caps at
`maxSections = 6` and throws with the URL when a page exposes neither an advance nor a submit
control.

Note the seeded exam has a **2 minute whole-exam time limit** (`EXAM_TIME_LIMIT_MINUTES` in
`CustomSeeds::E2e`, pinned to `whole_exam` mode). A spec that dawdles inside the exam will hit
expiry, which is a real behaviour and short enough to be waited out on purpose — but do not
mistake it for a flake.
