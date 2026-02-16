# Implementation Strategy (Deep Cut)

## Architecture Patterns
- Event-sourced tasks: treat every Title workflow as an append-only stream (`task_event` table) with reducers that materialize current state; enables replay, auditing, and time-travel debugging.
- Deterministic task VM: implement a minimal workflow interpreter (JSON DSL) that runs inside Workers; steps are pure functions with side-effect adapters (mail/ELT). This avoids heavyweight orchestration (e.g., Temporal) while keeping free-tier budgets.
- Partial evaluation of rules: compile YAML rules into TS modules at build-time; cache in KV with `version_hash`; worker loads precompiled functions for fee calc/validation → zero cold start penalty for parsing.
- CQRS: commands mutate via queues; queries hit Supabase views/materialized projections for UI speed; projections updated by reducers.
- Tenant partitioning: per-tenant row-level security + per-tenant partition keys in queues; optional schema-per-tenant for high-risk pilots (toggle).

## Workflow DSL (sketch)
```json
{
  "name": "duplicate_title_FL_v1",
  "steps": [
    {"id": "validate", "type": "fn", "fn": "validate_intake"},
    {"id": "status", "type": "call", "integration": "elt.ddi.status"},
    {"id": "branch", "type": "switch", "on": "status.path", "cases": {
      "duplicate": [{"id": "packet", "type": "fn", "fn": "build_packet"}],
      "payoff": [{"id": "payoff", "type": "fn", "fn": "gen_payoff_letter"}]
    }},
    {"id": "mail", "type": "call", "integration": "mail.easypost.label"},
    {"id": "wait", "type": "sleep", "duration_days": 30},
    {"id": "final", "type": "call", "integration": "elt.ddi.submit_duplicate"}
  ]
}
```
- Interpreter guarantees: step idempotency, deterministic randomness via seed, bounded CPU per step, resumable via checkpoints in Supabase.

## Rules & Templates Engine
- Precompile YAML → TS via codegen; emit Zod schemas + fee calculators + evidence checklists.
- Template registry: Handlebars/MD templates mapped to rule versions; `packet-builder` asserts template hash matches rule hash to prevent drift.
- Static analysis: enforce that every `rejection_code` has a remediation; every `submission.channel` has a fallback; lints fail otherwise.

## Integrations Layer
- Adapter interface per provider: `call(request) -> {status, data, retryable}`; standard error envelope; circuit-breaker module shared.
- Sandbox-first: fixtures stored in repo; contract tests (Pact) ensure we never regress mock schemas; real endpoints wrapped with feature flags.
- RPA fallback: headless Playwright runner invoked via `wrangler queues consumer --ephemeral` with chromium download cached; only for “manual” states and gated by feature flag to control free-tier CPU.

## Data Quality & Safety
- Property-based tests (fast-check) for workflow interpreter and rules: generate random VIN/homes and assert invariants (e.g., no negative fees, SLA timelines monotone).
- Synthetic tenants: seed data for chaos drills; mark synthetic via `tenant.kind = synthetic`; ensures monitoring excludes test noise.
- Evidence integrity: SHA256 for each upload; optional double-hash (payload + metadata) to detect swapped files.
- Deterministic IDs: ULID for ordering; idempotency key on all external calls; side effects behind “exactly-once” guard stored in `idempotency_log`.

## Advanced Observability
- Structured events per step: `{tenant, home, step, status, latency_ms, rules_version, payload_hash}`.
- Trace context propagated in queue messages (W3C traceparent); spans stitched post-hoc in analysis using Logpush + BigQuery external table.
- Anomaly detection: simple z-score alert on rejection_rate_by_state and queue_lag; no paid APM required.

## Performance Tactics (free-tier)
- KV-prefetch rules and templates; cache bust on `version_hash`.
- Chunked uploads with client-side compression (JPEG/webp) to cut Supabase Storage.
- Batch DB writes per agent tick; use RPC for hot paths to reduce round trips.
- Gate preview deploys to reduce Vercel CPU-hour usage; run Playwright only on `main` and release branches.

## Security & Compliance Extras
- Formal policy-as-code: Open Policy Agent (rego) snippets for RLS tests (run in CI).
- Deterministic redaction for logs; regex-based secret scrubbing middleware in Workers.
- Least-privilege API keys scoped per-tenant where provider supports sub-keys; otherwise per-environment with key-rotation cadence.

## Tooling Targets
- `pnpm rules:validate` – schema + hash check.
- `pnpm tasks:replay --id <task>` – DLQ replay with dry-run option.
- `pnpm tasks:plan --state FL --path duplicate_title` – renders workflow DSL for inspection.
- `pnpm chaos:toggle --scenario elt-outage` – injects faults in staging via feature flags.
- `pnpm evidence:verify --home <id>` – recompute hashes, detect missing blobs.

## Stretch Ideas
- Vision assist: optional on-device model (TFLite) to pre-validate HUD/VIN photo quality before upload.
- Auto-form filler: compile rules → PDF form field mappings; deterministic filler ensures reproducible packets for audits.
- Smart SLA tuner: learn per-state latency from telemetry; adjust `wait` steps and customer expectations automatically.
- Partner-facing API: expose webhook + API for bulk title checks; rate-limited; uses same DSL under the hood.
