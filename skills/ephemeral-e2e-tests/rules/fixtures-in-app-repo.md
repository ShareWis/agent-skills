# Fixture data lives in the app repo

The suite repo seeds nothing on the ephemeral target. `config/env.ts` sets
`seedingEnabled: false` for `ephemeral`, and `global-setup.ts` logs
`data is seeded by the ephemeral deploy; skipping seeding Lambda` and returns. The two Staging
Lambdas point at Staging's database and **must not** be called.

Ephemeral fixtures come from `ShareWis/sharewis-act`:

```
db/seeds/ephemeral.rb                 # base dataset — 3 orgs, users, sample courses
  └─ db/seeds/custom/e2e.rb           # CustomSeeds::E2e — the gate's own fixtures
spec/seeds/e2e_custom_seed_spec.rb    # its specs
```

`CustomSeedRunner` selects **exactly one** custom file by the env id (`EphemeralHost.id`), so on
the `e2e` stack it runs `e2e.rb` and nothing else. It runs on **every boot and every deploy**,
and the gate also re-runs it before every suite run.

Read the app repo's `AGENTS.md` first. All commands go through `./scripts/worktree_compose.sh` —
never raw `docker compose`.

## What `CustomSeeds::E2e` does today

```ruby
def self.run(context:)
  enable_incamera_on_tenant!(context)   # org-paid: incamera_enable = true
  exam = configure_sample_exam!(context) # pin the free-exam-course exam's configuration
  reset_exam_attempts!(exam)            # delete previous attempts — see @rules/persistent-env-state.md
  :ok
end
```

Constants worth knowing: `EXAM_COURSE_URL = 'free-exam-course'`,
`TENANT_SUBDOMAIN = 'org-paid'`, `EXAM_TIME_LIMIT_MINUTES = 2`.

## Adding fixture data

1. **Build on `context`, do not recreate.** `context` is the hash the base seeder returns:
   `:organizations` (the 3 orgs), `:super_admin`, `:users`, `:master_courses`, `:categories`,
   `:labels`, `:tags`. On a redeploy the identical hash is rebuilt read-only by
   `SampleDataSeeder#existing_context`, so the handles are live rows either way. Never create a
   new org.
2. **Idempotent, always.** It runs on every boot and every deploy. Deterministic keys plus
   `find_or_initialize_by` / `upsert_all`; never a blind `create!` in a loop. For
   `acts_as_paranoid` models without a soft-delete-aware unique index, look up `with_deleted` and
   `restore` — otherwise a soft-deleted fixture is duplicated on re-run.
3. **Pin every column you depend on.** Do not inherit a default: a previously deployed ref may
   have left the row in another shape. That is why the exam's `time_limit_mode_cd` is set
   explicitly even though the fixture only cares about the minutes.
4. **Add the cleanup.** Whatever your spec mutates, reset it here — in reverse dependency order.
   `reset_exam_attempts!` is the worked example: `lecture_statuses` carries `user_exam_id` so it
   goes first; `delete_all` for the partitioned composite-PK table; `really_destroy!` for
   `UserExam` because it owns `user_exam_details`, `user_exam_sections` and
   `organization_store_custom_fields` via `dependent: :destroy` and `delete_all` would orphan
   every one of them.
5. **Add a spec** in `spec/seeds/e2e_custom_seed_spec.rb`. Cover idempotency (running twice
   changes nothing) and the reset.
6. **Sample data only — never real users or PII.** Prefix ticket-scoped records so they are
   obvious to query and remove.

`update_columns` over `update!` is used throughout and is intentional: the `organizations` row is
at MySQL's row-size limit and full validation pulls in unrelated required settings. Keep that,
but note it skips callbacks — if your fixture needs a callback to fire, say why in a comment.

## Applying a change without waiting for a deploy

The **custom** seeder, with real exceptions instead of the best-effort swallow:

```bash
docker exec e2e-web-1 bundle exec rails ephemeral:seed_custom
```

The **base** seeder is skipped once the baseline exists, so edits to `db/seeds/ephemeral.rb` are
**not** picked up by a redeploy. Force it:

```bash
docker exec -e FORCE_BASE_SEED=1 e2e-web-1 bundle exec rails db:seed
```

A custom seeder that raises during a deploy is **best-effort**: `CustomSeedRunner` logs
`[ERROR]` plus a backtrace to the deploy/SSM logs and continues, so the env still comes up on the
base data. That swallow makes a half-applied seed look green — if fixtures are missing, use the
hand-run path above, which passes `raise_on_error: true`.

## Existing fixtures you can use without an app-repo PR

| Fixture | Value |
| --- | --- |
| Tenant | `org-paid` → `https://e2e-org-paid.ephemeral.share-wis.com/` |
| Org admin | `paid_org_admin@example.com` (`config.testData.defaultUserEmail`) |
| Learners | `paid_member_01`…`paid_member_05@example.com` — `02`/`03` claimed |
| Password | `config.testData.defaultUserPassword` |
| Exam course | `free-exam-course`, incamera-enabled, 2 minute whole-exam limit |

Other tenants (`org-free`, and the third base org) exist in the base dataset but have no
ephemeral spec yet — driving one is a fine reason to add fixtures, not a reason to skip the PR.
