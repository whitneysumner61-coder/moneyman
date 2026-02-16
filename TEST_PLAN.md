# Test Plan

## Pyramid
- Unit: pure functions (rules, validators, reducers, DSL interpreter).
- Contract: provider adapters (ELT, mail) against mocks/fixtures.
- Integration: Worker + Supabase (local) for task lifecycle.
- E2E: Next.js ↔ Worker ↔ Supabase ↔ mocks via Playwright.
- Chaos: fault injection in staging (feature-flag driven).

## Tooling
- Jest + ts-jest for unit.
- Pact/Prism for contract mocks.
- Playwright for E2E smoke (happy path FL/GA, manual state).
- k6 smoke (low RPS) to stay inside free tiers.
- fast-check for property-based rules tests.

## Required Scenarios
- Intake validation: VIN checksum fail, HUD missing, oversized photo rejected.
- Idempotent task creation: repeat `POST /api/tasks` with same key.
- Status check: ELT success, ELT 429, ELT 500 → circuit-breaker, DLQ.
- Packet build: missing template hash mismatch → fails fast.
- Mail: provider 503 → retry; after 3 fails → DLQ and alert.
- Manual fallback: flag on → DAG switches path; resumes when off.
- RLS: cross-tenant access denied on every table.
- Backup/restore: dump + restore; compare checksums for `evidence_blob` rows.
- Migration safety: shadow table diff matches expected.
- Chaos: drop ELT, saturate queue, Supabase pause, KV wipe; system enters safe degraded mode.

## Gates per PR
- Unit + contract + lint required.
- Playwright smoke on main/release only.
- Rules lint/validate if rules changed.
- DB migrations dry-run in CI with test DB.

## Metrics to Watch in Tests
- Queue processing time p95 < 2s (staging mocks).
- Worker CPU time <10ms median.
- Rejection rate in synthetic runs <1%.
