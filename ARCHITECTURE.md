# Architecture Blueprint (Free-Tier First)

## Topology
- Frontend: Next.js (TS) on Vercel Hobby; uses Supabase Auth; calls Worker API over HTTPS; real-time via Supabase Realtime (optional, watch quotas).
- Edge API: Cloudflare Worker (REST + signed webhooks) fronting Queues; validates auth (Supabase JWT) and tenant scope; thin compute only.
- Queues: Cloudflare Queues for async tasks; partitions by `tenant_id:home_id` to guarantee order.
- Agents: Stateless functions consuming Queues; each agent constrained <10ms CPU; long steps split into micro-tasks.
- Data: Supabase Postgres (row-level security) + Storage; Cloudflare KV as hot cache for rules and task cursors.
- Templates/Rules: Versioned in repo and published as NPM package; cached in KV; hash pinned per task.
- Observability: Cloudflare Logs → Logpush to R2 (free-tier friendly) → BigQuery/Supabase external table for analysis.

## Service Boundaries
- `apps/web`: UI-only; no business logic beyond validation.
- `apps/worker`: API endpoints + queue producers; task router; webhook receiver; cron jobs.
- Agents (conceptual): `intake`, `state-router`, `status-check`, `packet-build`, `mail-submit`, `notary-esign`, `audit`.
- `packages/rules`: schema + loaders + fee calculators; pure functions; no IO.
- `packages/shared`: types, errors, telemetry helpers, idempotency util.

## Key Flows
- Command flow: UI → Worker `/api/tasks` → enqueue → agent chain → Supabase writes → UI polls or subscribes.
- Webhook flow: External (mail/ELT) → Worker `/api/hooks/{source}` → signature verify → enqueue `hook_event` → state reducer updates Title/Task.
- Manual fallback: Feature flag per state; router diverts to `manual` queue; packet + mail produced; ops notified.

## Data Contracts (IDs)
- `tenant_id` UUID v4
- `home_id` ULID (time-sortable)
- `task_id` ULID
- `rules_version` semver string
- `idempotency_key` sha256(payload + rules_version)

## Database Tables (superset)
- `tenant(id, name, status, created_at)`
- `user(id, tenant_id, email, role, created_at)`
- `home(id, tenant_id, state, vin, hud_plate, address, created_at)`
- `title(id, home_id, status, path, rules_version, rejection_code, updated_at)`
- `lien(id, title_id, holder, amount_cents, status, payoff_reference)`
- `task(id, home_id, type, status, attempts, payload, result, idempotency_key, created_at, updated_at)`
- `evidence_blob(id, home_id, storage_path, sha256, kind, created_at)`
- `mail_event(id, home_id, carrier, label_id, tracking, status, delivered_at)`
- `state_rule_version(state, version, sha256, released_at)`
- `audit_log(id, tenant_id, actor, action, payload_hash, created_at)`

## APIs (Worker)
- `POST /api/tasks` create task (types: intake, status_check, duplicate_title, payoff, mail_notice)
- `GET /api/tasks/:id` fetch status
- `POST /api/hooks/easypost` webhook
- `POST /api/hooks/elt/:provider` webhook
- `POST /api/admin/replay` (authenticated ops) replay DLQ item

## Feature Flags
- `state.FL.enabled`, `state.GA.enabled`
- `integration.mail.enabled`
- `integration.elt.{provider}.enabled`
- `manual_mode.{state}`

## Performance Budgets (free-tier friendly)
- Worker CPU target <10ms per invocation; payload <100KB.
- Queue concurrency per partition = 1; global concurrency capped to stay under 100k req/day.
- Supabase: keep <500MB DB; compress blobs client-side; thumbnails for images.

## Security Controls
- Supabase RLS on every table keyed by `tenant_id`.
- JWT verification in Worker with audience check; reject if `role` not permitted.
- Signed webhooks (HMAC) for mail/ELT.
- Input validation via Zod on all boundaries.
- Secrets live in provider secret stores; never logged.

## Deployment
- Vercel Hobby for `apps/web` (preview per PR).
- Cloudflare `wrangler deploy` for Worker + Queues.
- Supabase migrations via `supabase db push` from CI with manual approval to prod.

## Recovery Paths
- Supabase pause/outage: read-only mode using KV + cached snapshots; block mutations; queue tasks suspended.
- Mail/ELT outage: circuit-breaker to manual path; backlog retained in queue.
- Worker rollback: `wrangler deploy --env prod --version <prev>`; Vercel rollback via UI/CLI.
