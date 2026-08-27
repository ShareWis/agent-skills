# Running specs and proving them

## Locally, against `e2e`

From the root of your `wisdombase-playwright-tests` clone:

```bash
E2E_TARGET=ephemeral npx playwright test --project=chromium-ephemeral
E2E_TARGET=ephemeral npx playwright test --project=chromium-ephemeral tests/ephemeral/exam-take.spec.ts
```

`E2E_TARGET=ephemeral` is the **only** switch you need — `config/env.ts` derives both the app URL
(`https://e2e-org-paid.ephemeral.share-wis.com/`) and the seeding path from one entry, so a caller
can never drive one environment while seeding another. **Do not set `BASE_URL`** to reach `e2e`;
`BASE_URL` overrides the URL only and re-decouples the two.

`--project` is mandatory here, and it must be an ephemeral-native one.

## Camera specs need Linux

`chromium-camera` cannot be developed on macOS: any `getUserMedia` call touching audio never
settles under `--use-file-for-fake-video-capture`, and the gate page's first call is the priming
`{video: true, audio: true}`. It hangs rather than fails. Run in a container instead — from the
root of your clone:

```bash
docker run --rm -v "$PWD":/work -w /work -e E2E_TARGET=ephemeral \
  mcr.microsoft.com/playwright:v1.57.0-jammy \
  sh -c "npm ci && npx playwright test --project=chromium-camera"
```

Keep the image tag in step with `@playwright/test` in `package.json` (currently `^1.57.0`).

Camera support is Chromium-only and permanently so: Firefox's `media.navigator.streams.fake`
yields a synthetic pattern with **no face**, and WebKit has no fake-device support at all.

### The face fixture

`fixtures/face.y4m` feeds the fake capture device. A real, detectable face is required — not for
the in-exam capture loop (which draws to a canvas and uploads blind, checking nothing) but for the
**pre-exam gate**, which runs `faceapi.detectAllFaces` and only lets you through on a hit. A
colour-bar source passes the loop and fails the gate. If you replace it, run
`node scripts/verify-face-fixture.js fixtures/face.y4m` first, and keep the clip ~1 second — Y4M
is uncompressed (~4.4 MB/s at 640x480) and Chrome loops the file. It is a synthetic StyleGAN2
portrait on purpose; never substitute a photo of a colleague.

## Network access

The `e2e` env sits behind a security-group and Traefik allowlist. The gate opens a temporary
`/32` for its runner and revokes it afterwards (on failure and cancellation too). A **local** run
needs your IP already on the allowlist — otherwise the health check fails first with
`[HEALTH CHECK] ephemeral is unreachable at …`, which is the intended fail-fast, not a bug.

## Guard messages you will meet, and what they mean

`global-setup.ts` refuses ambiguous runs rather than letting them mutate the wrong database.

| Message | Cause | Fix |
| --- | --- | --- |
| `Refusing to run: BASE_URL points at an ephemeral env … but Lambda seeding is enabled` | `BASE_URL` set to `e2e` without `E2E_TARGET=ephemeral` | drop `BASE_URL`, set `E2E_TARGET=ephemeral` |
| `E2E_TARGET=ephemeral requires an explicit ephemeral-only --project` | no `--project`, so every project would run | add `--project=chromium-ephemeral` (or `chromium-camera`) |
| `E2E_TARGET=ephemeral cannot run Staging-capable projects: …` | `chromium` / `firefox` / `webkit` selected | those specs need `SEED_RESULT`; run them on the default (staging) target |
| `E2E_TARGET=ephemeral runs only tests/ephemeral/** specs` | a Staging spec named explicitly | run it against Staging, or port its fixtures into `db/seeds/custom/e2e.rb` and move it under `tests/ephemeral/` |
| `Seed result not available. Global setup may have failed.` | a Staging spec ran on the ephemeral target | as above |

The gate rejects `chromium` / `firefox` / `webkit` in its own run step for the same reason.

## Debugging a failure

- **Runtime guard violations** are attached to the test as `runtime-violations`
  (`[console] …`, `[response] 500 GET …`) and listed in the thrown error. Read them before
  assuming a selector problem.
- **Artifacts**: `trace: "on-first-retry"`, `screenshot: "on"`, `video: "retain-on-failure"`.
  `npx playwright show-report` locally; the gate uploads `playwright-report-<sha>` as an artifact.
- **`helpers/timeout-reporter.ts`** prints a summary sorting failures into TIMEOUT (infra),
  ASSERTION (bug) or OTHER. A TIMEOUT on a control you expect to be visible usually means a
  missing `:visible` (see @rules/spec-anatomy.md) or a covered element behind a remodal overlay.
- **No lecture link on the course page** → stale state from a previous run, not a selector
  change. See @rules/persistent-env-state.md.
- **Gate cookie never became `"true"`** → the fixture was not detected. `"false"` means the gate
  ran and found no face (verify the fixture); `undefined` means the check never completed
  (usually SWWB-24509, the un-awaited model load).

## Prove it twice

Two consecutive passes against `e2e` is the bar. The second run is what catches missing seeder
cleanup. See @rules/persistent-env-state.md.

The gate runs with `retries: 2` and `workers: 1` under `CI`, so a first-attempt failure a retry
papers over is still a flake you own — read the report, not just the exit code.

## Triggering the gate

In `ShareWis/sharewis-act`: **Actions → "E2E · release gate" → Run workflow**. The workflow
branch must be `master`.

| Input | `deploy_and_test` | `test_current` |
| --- | --- | --- |
| `ref` | **required** (e.g. `master`) — deploys it to `e2e`, then tests it | **must be blank** — re-runs against whatever is already deployed |
| `projects` | `chromium-ephemeral,chromium-camera` (default) | same |

The mode/ref combination is validated up front and rejected with a clear error, so a blank ref on
`deploy_and_test` or a filled ref on `test_current` fails fast. The requested ref is pinned to an
immutable SHA **once**, before anything is built, and the report names the SHA actually deployed —
never the requested ref. Exit status is informational: the gate is non-blocking, and a human still
approves the release in the AWS console.
