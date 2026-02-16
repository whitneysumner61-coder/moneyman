# API Spec (Worker Edge)

Auth: Supabase JWT in `Authorization: Bearer <token>`; `tenant_id` claim required. All responses JSON.

## Endpoints
- `POST /api/tasks`
  - body: `{ type: "intake|status_check|duplicate_title|payoff|mail_notice", home_id?, payload }`
  - returns: `{ task_id, status }`
- `GET /api/tasks/:id`
  - returns: `{ id, status, attempts, result?, events[] }`
- `POST /api/homes`
  - body: `{ state, vin, hud_plate?, address, photos[] }`
  - returns: `{ home_id }`
- `GET /api/homes/:id`
  - returns home + latest title projection.
- `POST /api/hooks/easypost`
  - headers: `X-Signature`
  - body: provider payload → enqueued `hook_event`.
- `POST /api/hooks/elt/:provider`
  - signature verified; body stored to `task_event`.
- `POST /api/admin/replay`
  - body: `{ task_id, dry_run?: true }` (admin role only).

## Error Envelope
```json
{ "error": { "code": "INVALID_INPUT|UNAUTHORIZED|NOT_FOUND|RATE_LIMIT", "message": "...", "details": {} } }
```

## Rate Limits
- Default 60 req/min per tenant; stricter on admin endpoints.

## Idempotency
- Client may send `Idempotency-Key` header; Worker stores in `idempotency_log`; identical payloads return existing result.

## Versioning
- Header `X-Rules-Version` optional; if absent uses current stable. Response echoes version used.

## Validation
- All payloads pass Zod schemas; numeric currency fields expected in cents; files uploaded via pre-signed URLs from Supabase Storage.
