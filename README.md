# mh-title-saas (free-tier first)

Vertical SaaS to automate mobile home title recovery using free-tier cloud, agent swarms, and state-specific rule files.

## High-Level Architecture
```mermaid
flowchart LR
  UI[Next.js (Vercel)] --> API[Workers API Edge]
  API --> Queue[Cloudflare Queues]
  Queue -->|tasks| Agents{{Agent Swarm}}
  Agents --> Supabase[(Supabase Postgres/Storage)]
  Agents --> KV[Cloudflare KV cache]
  Agents --> Mail[EasyPost/Lob]
  Agents --> ELT[ELT provider API/mock]
  Supabase --> BI[Metrics/Logs]
```

## Components
- `apps/web`: Next.js (TS) on Vercel Hobby; Supabase Auth; Zod forms; VIN/HUD photo capture.
- `apps/worker`: Cloudflare Worker (TS) exposing REST hooks + consuming Queues; scheduled cron.
- `packages/shared`: DTOs, validation, error codes.
- `packages/rules`: YAML state rules, JSON schema validator, packet template registry.
- `infra/`: IaC stubs for Vercel/Cloudflare; Supabase CLI config.

## Core Domain Model (Supabase)
- Tenant, User (RBAC, RLS by tenant)
- Home (VIN, HUD plate, address, photos)
- Title (state, status, path: duplicate | payoff | abandoned | conversion)
- Lien (holder, amount, payoff status)
- Task (queue-driven, retries, correlation_id)
- EvidenceBlob (storage refs + hashes)
- MailEvent (label id, tracking, delivery)
- StateRule (versioned ruleset per state)
- AuditLog (append-only)

## Agent Swarm Responsibilities
- `intake-validator`: VIN checksum, HUD plate format, photo presence.
- `state-router`: picks ruleset; resolves notarization + fee tables.
- `status-check`: ELT API/mock; emits Title status.
- `packet-builder`: merges rules + data into PDFs; signs/notarizes when required.
- `mail-submit`: Certified Mail labels; stores delivery confirmations.
- `notary-esign`: integrates Notarize/Stavvy when enabled.
- `audit-logger`: durable events to AuditLog; emits metrics.

## State Rules (YAML) Shape
```yaml
state: FL
version: "0.1.0"
forms:
  duplicate_title: ["HSMV-82101", "HSMV-82139"]
notarization: jurat
fees:
  duplicate_title: 85.00
submission:
  channel: "elt|mail|in-person"
  endpoints:
    elt: "https://sandbox.ddi/..."
mail:
  notice_required: true
  certified: true
timeouts:
  notice_wait_days: 30
rejection_codes:
  - code: VIN_MISMATCH
    remedy: "Reverify VIN; resubmit with photo of HUD plate."
```

## Environments
- `local`: optional Supabase local; Cloudflare dev.
- `staging`: Supabase project + Vercel preview + Worker dev; sandbox integrations.
- `prod`: Supabase prod + Vercel prod + Worker prod.
- `pilot-{client}`: optional isolated schema.

## CI/CD (GitHub Actions)
- Jobs: lint/test → build web → build worker → supabase db diff → Playwright smoke (staging) → manual approve → deploy worker + promote Vercel.
- Secrets: managed via GitHub + Vercel + Cloudflare; no .env in repo.

## Security Baseline
- Supabase RLS per tenant; JWT claims enforced at DB.
- Storage buckets scoped per tenant; signed URLs for evidence.
- AuditLog append-only; hash chain for tamper evidence.
- Least-privileged API keys for ELT/mail; rotate via Secrets Manager.

## Happy-Path Workflow (FL/GA)
1) Intake form → validate VIN/HUD → store to Supabase.
2) Queue `status-check` → ELT mock/API returns lien/title info.
3) `state-router` selects path (duplicate/payoff/abandoned).
4) `packet-builder` renders PDFs from templates; attaches fees.
5) `mail-submit` issues Certified Mail labels; stores tracking.
6) `notary-esign` (optional) collects signatures.
7) Status updates via webhook/cron → Task updates → UI timeline.

## Manual-State Workflow
1) Collect required IDs/forms per rules.
2) Render packet; create Certified Mail labels.
3) Log delivery; wait statutory days.
4) File duplicate/affidavit; record outcome.

## Metrics
- time_to_title, rejection_rate_by_state, mail_delivery_confirmation_pct, task_queue_lag, cost_per_title, pilot_NPS.

## Run Locally (soon)
```bash
pnpm install
pnpm --filter apps/web dev          # Next.js with Supabase env
pnpm --filter apps/worker dev       # Miniflare for Workers/Queues
supabase start                      # optional local Supabase
```

## Roadmap (free-tier aware)
- M1: FL/GA rules + mock ELT + Certified Mail sandbox.
- M2: First real ELT sandbox + abandoned-home notice flow.
- M3: Add payments (virtual card/ACH) + SOC2-lite controls.

## Handoff to Claude
- Fill `packages/rules/states/fl.yaml`, `ga.yaml` with real forms/fees.
- Add templates in `packages/rules/templates/{state}/`.
- Supply ELT status mock fixtures and manual-state runbook.
