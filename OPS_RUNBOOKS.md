# Ops Runbooks

## DLQ Replay
1) List items: `pnpm tools:list-dlq --limit 50`
2) Inspect payload + last error.
3) If fixable (e.g., rule typo), patch and redeploy.
4) Replay: `pnpm tools:replay --task <id> [--dry-run]`
5) Verify task reaches terminal state; close incident note.

## Supabase Paused/Down
1) Detect via health check.
2) Flip `feature_flag` → `read_only=true`; block new mutations.
3) Serve cached status from KV; show banner in UI.
4) Resume project in Supabase dashboard; rerun backlog tasks from queue.

## ELT Outage
1) Circuit-breaker opens after threshold; alert ops.
2) Toggle `manual_mode.<state>=true`.
3) Generate packets + mail; log manual submission tracking.
4) When stable, close breaker; resume auto path; reconcile duplicates.

## Mail Provider Outage
1) Switch to offline label template; queue `mail_pending`.
2) On recovery, submit queued labels; dedupe by `label_id`.

## Migration Rollback
1) If failure post-apply: restore latest backup to new DB; switch connection env var to restored DB.
2) Apply hotfix migration if forward-fix is smaller.

## Secret Rotation
1) Run `pnpm tools:rotate-secrets --target <provider>`.
2) Update CI + provider consoles.
3) Redeploy Worker + Vercel with new secrets; smoke test.

## Data Deletion Request
1) Verify identity.
2) Enqueue `data_purge` task scoped to tenant/user.
3) Remove Supabase rows + storage objects.
4) Log in audit; send confirmation.

## Chaos Drill Playbook
- Scenarios: ELT down, Supabase paused, KV wiped, queue backlog.
- Steps: toggle fault flag → run synthetic tasks → verify alerts, degraded mode, recovery time.
- Record outcomes in `ops/drills/YYYY-MM-DD.md`.
