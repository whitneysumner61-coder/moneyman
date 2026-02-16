# Delivery Plan (Expanded)

## Milestones
- M0 (Week 0-1): Repo scaffolding, Supabase schema, Worker + Queue skeleton, FL/GA rule placeholders, CI baseline.
- M1 (Week 2-3): Intake UX, VIN/HUD validation, evidence upload, task enqueue; mock ELT + mail sandbox; DLQ tooling.
- M2 (Week 4-5): FL/GA happy path (duplicate title + mail notices); manual-state fallback; audit log; metrics pipeline.
- M3 (Week 6-7): First ELT sandbox integration; abandoned-home notice flow; SLA timers; incident tooling.
- M4 (Week 8-9): Pilot with 2 tenants; harden RLS; chaos drills; prepare launch assets.
- M5 (Week 10-12): Add 2 more states; payments (virtual card/ACH); SOC2-lite policy set; public beta listings.

## Workstreams & Owners (suggested)
- Platform: CI/CD, secrets, migrations, observability.
- Frontend: Intake UI, timeline, tenant admin, file uploads.
- Integrations: ELT, mail, e-sign/notary.
- Rules/Content: State YAML, templates, runbooks.
- Ops/Compliance: RLS tests, backup/restore, incident comms, data purges.

## Backlog (Prioritized)
- P0: Auth (Supabase), RLS, task router, idempotency, audit log, DLQ tooling.
- P1: Intake form, VIN checksum, HUD photo upload, packet builder (PDF), mail sandbox.
- P2: ELT mock fixtures, manual-mode path, SLA timers, rejection code mapping.
- P3: Abandoned-home notice workflow, notification emails, dashboard metrics.
- P4: Payments integration, CSV import/export, role-based UI, multi-tenant billing.
- P5: Mobile offline-friendly uploads, SSO, custom webhooks for tenants.

## Acceptance Criteria (per milestone)
- M1: Create title task → status transitions visible; DLQ replay works; RLS tests pass.
- M2: FL/GA duplicate title flow completes end-to-end with mock ELT + mail label stored; audit entries for every transition.
- M3: Live ELT sandbox call returns status; manual fallback toggle works; incident runbook exercised.
- M4: Pilot tenants complete ≥5 titles each; time-to-title improved vs baseline; chaos drills green.
- M5: Payments live; SOC2-lite doc set stored; beta listings published.

## Testing Gates
- Unit coverage on shared/rules >80%.
- Contract tests for ELT/mail mocks required in CI.
- Playwright smoke per PR against staging.
- Migration test + backup before prod migrate.

## Success Metrics
- <5% rejection rate in pilot states.
- ≥50% cycle-time reduction vs baseline for duplicate titles.
- 99% task success (non-DLQ) per week.
- Supabase/Worker spend stays inside free tier for pilot volume.
