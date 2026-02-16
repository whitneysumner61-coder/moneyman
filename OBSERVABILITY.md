# Observability

## Signals
- Metrics: queue.lag, queue.fail, tasks.dlq, worker.cpu_ms, elt.latency, mail.latency, db.qps, db.errors, rejection_rate_by_state, time_to_title.
- Logs: structured JSON with `tenant_id`, `home_id`, `task_id`, `step`, `rules_version`.
- Traces: W3C traceparent propagated through queue messages; sampled 10%.

## Sinks
- Cloudflare Logpush → R2 → BigQuery external table (cost-light).
- Supabase logs (Postgres) retained per plan; export daily if needed.

## Dashboards (min)
- SLA dashboard: time-to-title, SLA breaches, per-state latency.
- Queue health: lag, DLQ volume, retries by reason.
- Integrations: ELT success/429/5xx, mail success/fail.
- Errors: top error codes, template/rule mismatches.

## Alerts (pager)
- queue.lag > 2x SLA for 10m.
- tasks.dlq > 20 in 10m.
- worker 5xx >1% over 5m.
- Supabase quota >70%.
- rejection_rate_by_state z-score >2 over 24h.

## Instrumentation
- Telemetry helper in `packages/shared` wraps logs, metrics, trace context.
- Agent steps emit `step_completed` events with latency + result size.
- Sample size tunable via feature flag to manage free-tier limits.
