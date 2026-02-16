# Data Model (Supabase)

## Conventions
- All tables include `id`, `created_at`, `updated_at`; UUID v4 unless noted.
- `tenant_id` on every tenant-scoped table; enforced via RLS.
- Monetary values in cents (int); UTC timestamps.

## Tables
- `tenant(id, name, status, plan, created_at)`
- `user(id, tenant_id, email, role, last_login_at, created_at)`
- `home(id ULID, tenant_id, state, vin, hud_plate, address_json, photos JSONB, created_at)`
- `title(id, home_id, status, path, rules_version, rejection_code, sla_due_at, updated_at)`
- `lien(id, title_id, holder, amount_cents, status, payoff_reference, payoff_due_at)`
- `task(id ULID, home_id, type, status, attempts, payload JSONB, result JSONB, idempotency_key, correlation_id, created_at, updated_at)`
- `task_event(id, task_id, step, status, data JSONB, error JSONB, created_at)` — event-sourcing spine.
- `mail_event(id, home_id, carrier, label_id, tracking, status, delivered_at, raw JSONB)`
- `evidence_blob(id, home_id, storage_path, sha256, kind, content_type, size_bytes, created_at)`
- `state_rule_version(state, version, sha256, released_at)`
- `audit_log(id, tenant_id, actor, action, payload_hash, meta JSONB, created_at)`
- `feature_flag(key, value JSONB, environment, created_at)` — read-only at runtime.
- `billing_event(id, tenant_id, type, amount_cents, meta JSONB, created_at)` — optional if billing enabled.

## Indexes (minimal set)
- `home(tenant_id, state, vin)` unique on `(tenant_id, vin)`.
- `task(home_id, status)`; `task(idempotency_key)` unique.
- `task_event(task_id, created_at)` for replay ordering.
- `mail_event(tracking)` unique.
- `state_rule_version(state, version)` unique.
- `audit_log(tenant_id, created_at)` for paging.

## RLS Outline
- `tenant_id = current_setting('request.jwt.claims.tenant_id')::uuid`.
- Roles: `admin`, `operator`, `viewer`; restrict write scopes.
- `feature_flag` selectable by env only (no tenant).

## Stored Procedures (optional)
- `fn_resolve_title_state(home_id)` → projection for UI.
- `fn_upsert_task(task_json)` with idempotency guard.
- `fn_record_audit(actor, action, hash, meta)` small wrapper.
