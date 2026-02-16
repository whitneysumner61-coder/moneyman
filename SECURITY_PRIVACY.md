# Security & Privacy

## Data Classification
- PII: names, emails, addresses, IDs, VIN/HUD photos.
- Confidential: DMV/ELT credentials, payment tokens.
- Public: marketing content only.

## Controls
- Auth: Supabase Auth JWT; enforce `aud` + `tenant_id`.
- RLS: mandatory on all tenant tables; tests in CI.
- Encryption: Supabase at rest; TLS in transit; optional client-side encryption for Evidence blobs.
- Secrets: stored in Vercel/Cloudflare/Supabase; rotated via tooling; no secrets in repo.
- Logging: PII scrubbing middleware; only hashed identifiers in logs.
- Least privilege: per-provider keys; restrict to needed scopes; per-env isolation.
- Access: admins via SSO (when enabled); principle of least privilege for roles.

## Privacy
- Data minimization: collect only required documents per state rule.
- Retention: configurable per tenant; purge task; right-to-delete runbook.
- Consent: explicit consent screen for uploads; log consent in `audit_log`.

## Compliance Roadmap
- SOC2-lite: policies for access control, change management, incident response.
- Backups: daily Supabase backups; weekly restore test.
- Vendor review: ELT/mail/e-sign vendors tracked with DPAs; data map maintained.

## Incident Response
- Defined in PROTOCOLS: declare, contain, eradicate, recover, notify.
- Notify tenants if PII involved; timeline in audit.

## Secure Coding
- Input validation via Zod everywhere.
- Disable eval/dynamic code in templates; whitelist helpers.
- CSP on frontend; same-site cookies (if used); HTTPS only.
- Dependency scans weekly; lockfiles pinned.
