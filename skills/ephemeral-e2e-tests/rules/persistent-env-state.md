# The env is persistent — design for run N, not run 1

`e2e` is a **named, long-lived** ephemeral stack, not a per-run throwaway. The release gate
re-runs against it on every candidate build, and QA uses it between runs. Nothing resets it
between gate runs except the seeder.

Every consequence below follows from that one fact.

## The trap that has already bitten

Once a **finished `LectureStatus`** exists for a lecture, `Course#lecture_to_study_for_user`
excludes that lecture, and the course page **stops rendering its link entirely**. There is no UI
path back. So:

- Run 1: spec finds the lecture link, takes the exam, passes.
- Run 2: no lecture link on the course page. The spec fails with something that looks like a
  selector problem and is not.

`openExamLecture` throws with that explanation baked into the message, so you get told rather
than left guessing. `CustomSeeds::E2e.reset_exam_attempts!` is what makes run 2 equal run 1 — it
hard-deletes the exam's `LectureStatus` and `UserExam` rows on every deploy and before every
suite run.

## The rule

**If your spec mutates anything, the seeder must reset it.** Ask, before writing a line of spec
code: *what does this spec leave behind, and what does the next run see?* Completing a lecture,
submitting an exam, purchasing, changing a setting, creating a record with a unique constraint —
all of these need a reset in `db/seeds/custom/e2e.rb`. See @rules/fixtures-in-app-repo.md for how to
write it.

Run 1 and run 2 exercising different pages is exactly what a release gate must not do.

## Prove it twice

**A single green run is not evidence.** Run the spec twice in a row against `e2e` and require
both to pass:

```bash
E2E_TARGET=ephemeral npx playwright test --project=chromium-ephemeral tests/ephemeral/your-spec.spec.ts
E2E_TARGET=ephemeral npx playwright test --project=chromium-ephemeral tests/ephemeral/your-spec.spec.ts
```

The second run is the one that catches missing cleanup. If it needs a manual reseed between runs
to pass, the seeder is wrong — the gate will not reseed on your behalf beyond what
`CustomSeeds::E2e` does.

Note the gate runs with `retries: 2` in CI, so a first-attempt failure that a retry papers over
is a flake you still own. Check the report, not just the exit code.

## Do not paper over app bugs

Two open defects were found precisely by refusing to:

- **SWWB-24509** — the gate page fires `faceapi.nets.tinyFaceDetector.loadFromUri(...)` **without
  awaiting it**, so the weights can still be in flight when the button becomes clickable.
  Clicking then throws `TinyYolov2 - load model before inference` and the detection silently does
  not run. A human is usually slow enough to miss this; a script is not, and a real user on a
  slow link can hit it.
- **SWWB-24510** — the incamera gate is a **client-set cookie**, i.e. trivially forgeable. A
  security issue, not a test problem.

`helpers/incamera.ts` currently polls `faceapi.nets.tinyFaceDetector.isLoaded` to work around
24509, and says so in a comment naming the ticket. That is the required shape for any workaround:

1. The workaround is **narrow** and lives next to the thing it works around.
2. A comment states what the app does wrong, cites the file, and names the Jira ticket.
3. The ticket exists in project **SWWB**.

What is not acceptable: widening a timeout, retrying until it sticks, adding a
`runtime-guard` ignore entry, or relaxing an assertion, in order to hide app misbehaviour. A
green suite hiding a real defect is worse than a red one — the gate feeds a human's production
approval decision.

## Other persistence consequences worth remembering

- **Concurrency.** The gate workflow uses `concurrency: { group: e2e-release-gate,
  cancel-in-progress: false }`, so gate runs queue rather than overlap. Your **local** run has no
  such protection: a local run during a gate run drives the same database. Coordinate.
- **Shared fixtures across specs.** Two specs driving the same seeded record will interfere
  whenever they run together. Give each spec its own learner account (see @rules/spec-anatomy.md) and
  keep that separation deliberate and commented.
- **A previously deployed ref may have left rows in an unexpected shape.** This is why
  `CustomSeeds::E2e` pins *every* column its fixture depends on — including
  `time_limit_mode_cd` — rather than assuming a default. Seed everything you depend on; inherit
  nothing.
