# Risk, Mitigation, and MCP/Tool Protocols (Free-Tier Stack)

## Global Guardrails
- Single source of truth: Supabase Postgres; state transitions idempotent and recorded in `audit_log`.
- All agents must be deterministic given (tenant_id, task_id, payload, rules_version); include hash of rules+payload in every event.
- Default timeouts: Worker 8s soft, 10ms CPU per invocation target; queue backpressure triggers circuit-breaker to shed non-critical tasks.
- Every external call has: timeout, retry with jitter (3 attempts), and a DLQ path with human escalation.

## Failure Matrix (What Can Go Wrong → Mitigation → Execution Protocol)
- Supabase free tier paused for inactivity → Keep health cron hit every 12h; on 503, auto-failover to read-only KV cache; page ops to resume project.
- Supabase connection cap exceeded → Use pgbouncer-like pooling via PostgREST; queue tasks batch writes; shed analytics first.
- RLS misconfiguration exposes cross-tenant data → Pre-deploy RLS tests; runtime guard checks `tenant_id` on every query; canary deploy with synthetic tenants; rollback script ready.
- Worker cold-start/CPU overrun → Keep functions small; split tasks; set queue concurrency to 1 per partition; measure CPU via Cf-Trace; auto-split big payloads to subtasks.
- Queue backlog/poison messages → Max retry 3; DLQ topic `tasks-dead`; daily sweeper lists stuck tasks; manual replay tool `pnpm tools:replay --task <id>`.
- Clock skew across services → Use monotonic timestamps from DB `NOW()`; reject client timestamps; TTLs computed server-side.
- Idempotency loss → Every task has `idempotency_key` stored in DB; writes conditioned on key absence; replays are safe.
- PDF generation fails/oversize → Fallback to plain PDFKit template; cap size 10MB; store render logs; DLQ after 2 failures.
- Mail API (EasyPost/Lob) outage → Switch to offline label template + USPS CLI postage file; queue for later submission; flag tasks `mail_pending`.
- ELT API down/rate limited → Circuit-breaker after 429/5xx; backoff exponentially; if >24h, switch to manual packet path; notify ops.
- Storage object corruption/missing → Store SHA256 per EvidenceBlob; verify on upload; nightly integrity scan; re-request upload from user if mismatch.
- Upload abuse (huge files) → Enforce signed URL with size limit 10MB; reject mime types not whitelisted; virus scan via Workers AV hook (optional).
- Auth compromise/token theft → Short-lived Supabase JWTs; refresh rotation; device logout; anomaly detection on IP/UA mismatch; admin lockout switch.
- Secrets leak in repo/logs → Pre-commit secret scan; CI secret scan; logs scrubber; rotate keys via script `pnpm tools:rotate-secrets`.
- Data loss/migration gone wrong → Migrations gated; backup before apply via `supabase db dump`; revert plan documented; shadow table validation after migrate.
- Staging data contaminates prod → Separate projects/keys; hard check on `SUPABASE_URL` hostname; guardrails refusing prod writes from non-prod builds.
- Task ordering bugs → Use per-title partition key; enforce sequential processing in queue for same partition; optimistic locking on Title row version.
- Timezone errors on statutory wait → All dates stored UTC; rules specify durations; UI shows state-local offset computed from address.
- Evidence tampering → Hash-chain in `audit_log`; expose proofs to customers; signed URLs short-lived.
- DDoS/abuse of public endpoints → Turn on Cloudflare DDoS free shield; rate-limit per IP; require auth for mutations; captcha on public forms if opened.
- Cost overrun (free tier bust) → Alert at 70% quota; switch to degraded mode (no previews, reduced cron); cache-heavy reads; backoff non-critical tasks.
- Agent hallucination/unsafe action → Agents can only call whitelisted MCP tools; dry-run flag default true in staging; require rule hash validation; human approval for irreversible ops.
- Compliance (PII/consent) gaps → Data map by table/column; purge job for right-to-delete; log lawful basis; no production data in logs.
- Pilot customer SLA breach → SLA clock per title; alert when >80% of SLA; automatic concession credit creation procedure.
- Version drift between rules/templates → Rules and templates versioned together; task stores `rules_version`; deployments validate template exists for that version.
- Dependency supply-chain risk → Lockfiles committed; weekly `pnpm audit`; deny-list known bad packages; reproducible builds in CI.
- Incident communication failure → Status page (simple Vercel static); template comms; 1h update cadence during Sev1.

## Execution Protocols (Step-by-Step)
- Incident (Sev1/Sev2)
  - Declare in #incident channel; assign IC + comms lead.
  - Freeze deploys; capture timeline in `ops/runbooks/incidents.md`.
  - Mitigate (failover, circuit-breaker, rollback); validate; communicate ETA.
  - Postmortem within 48h with action items.
- Queue DLQ Handling
  - `pnpm tools:list-dlq --limit 50`
  - For each task: inspect payload, root-cause tag, patch rules/templates if needed, `--replay` or `--cancel`.
- Secret Rotation
  - Run `pnpm tools:rotate-secrets --target supabase|vercel|cloudflare`.
  - Update GitHub Actions secrets; bump version tag; notify ops.
- Migration Run
  - `supabase db diff` → review → `supabase db push --dry-run` staging → run integration tests → backup prod → `supabase db push` prod → shadow validation query.
- Rules/Template Release
  - Add/modify YAML in `packages/rules/states`; run validator; bump `rules_version`; ensure matching templates; publish NPM package; deploy Worker with new hash.
- Manual-State Fallback Activation
  - Flip feature flag `manual_mode=true` per state.
  - Force tasks into manual queue; generate mail packets locally; store proofs; resume auto when API stable.
- Data Deletion Request
  - Verify requester; enqueue `data_purge` task with scope; purge Supabase rows + storage; log in audit; confirm completion.

## MCP Tooling Protocol
- Message contract: `{task_id, tenant_id, rules_version, payload_hash, dry_run, created_at}`.
- Allowed tool calls per agent type are whitelist-enforced; all calls logged to AuditLog with result + duration.
- Retries: max 3; classify errors (transient vs fatal); only transient go to retry; fatal go to DLQ.
- Circuit-breakers: per-integration rolling window; open on >20% failure over 5 min; auto half-open after cooldown.
- Observability: metrics `queue.lag`, `tasks.fail`, `tasks.dlq`, `elt.latency`, `mail.latency`, `db.qps`; traces sampled at 10%.

## Monitoring & Alerting
- Alerts: queue lag > 2x SLA; DB errors >1% of requests; Worker 5xx >1%; Supabase quota >70%; storage integrity mismatch detected.
- Dashboards: per-state SLA, rejection reasons top-10, mail delivery success, template/render failure rate.

## Disaster Recovery
- Daily Supabase backups; weekly verified restore into staging clone.
- KV cache can warm from DB snapshots; if Supabase down, system serves read-only status from KV and blocks new writes.
- Vercel/Worker redeploy scripts stored locally and in CI; no single point of failure in Secrets (duplicated in vault + provider).

## Onboarding Checklist (Ops Readiness)
- Run chaos drills: kill ELT endpoint, exhaust queue, break RLS, corrupt template; verify detection and recovery.
- Validate RLS tests, backup/restore test, DLQ replay test, and manual-mode switch before go-live.
