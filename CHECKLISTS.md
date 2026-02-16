# Checklists

## Pre-Commit
- Lint + unit tests pass.
- No secrets in diff.
- Rules/templates unchanged? If changed, ran `rules:validate`.

## Pre-PR
- Added/updated docs for new features.
- Added tests for new logic.
- DB schema changes accompanied by migration + rollback notes.

## Pre-Deploy (staging)
- CI green (unit, contract).
- Migrations applied to staging.
- Playwright smoke run.
- Feature flags set for staging.

## Pre-Deploy (prod)
- Backup taken (Supabase dump).
- Migrations reviewed/approved.
- Feature flags staged; manual-mode ready.
- Incident channel on-call aware for change window.

## Post-Deploy
- Smoke tests run (health, task create, webhook ingest).
- Metrics baseline recorded.
- Log scan for errors first 30 minutes.

## New State Onboarding
- Rules YAML authored + validated.
- Templates hashed and linked.
- Manual fallback steps documented.
- Chaos test for that state (ELT down) executed.

## Pilot Onboarding
- Tenant created; users invited; roles assigned.
- Import 5 sample homes.
- Run end-to-end flow with mocks; confirm SLA timers visible.
- Support contacts shared; status page link shared.

## Incident Wrap-Up
- Timeline captured; root cause and actions recorded.
- Action items assigned with dates.
- Customers notified (if applicable).
