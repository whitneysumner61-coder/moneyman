# State Rules Schema (YAML)

Use semver and keep rules + templates in lockstep. Validate via JSON Schema before publish.

```yaml
state: FL                  # postal code
version: "0.1.0"           # semver
updated_at: 2026-02-16     # ISO date
forms:
  duplicate_title:
    codes: ["HSMV-82101", "HSMV-82139"]
    notarization: "jurat|ack|none"
    fee_usd: 85.00
    requires_id_copy: true
  lien_payoff:
    codes: ["HSMV-82139"]
    notarization: "ack"
    fee_usd: 0
submission:
  channel_priority: ["elt", "mail", "in-person"]
  elt:
    provider: "ddi|vitu|pdp|sta"
    endpoint: "https://sandbox.ddi.example/v1/..."
    auth: "api_key|oauth|sftp"
  mail:
    address: "DMV PO Box ..."
    certified_required: true
  in_person:
    allowed: true
notices:
  required: true
  types:
    - name: "lienholder_notice"
      method: "certified_mail"
      wait_days: 30
      proof: ["tracking", "delivery_scan"]
timelines:
  sla_days:
    duplicate_title: 14
    payoff: 10
  statutory_wait_days: 30
rejection_codes:
  - code: "VIN_MISMATCH"
    remedy: "Reverify VIN, include HUD plate photo."
  - code: "NO_PROOF_OF_ADDRESS"
    remedy: "Attach utility bill or lease."
evidence_requirements:
  photos: ["VIN_plate", "HUD_plate", "home_front"]
  ids: ["gov_id_front", "gov_id_back"]
  docs: ["bill_of_sale?", "affidavit_of_abandonment?"]
fees:
  duplicate_title_usd: 85.00
  mail_certified_usd: 7.50
  notarization_usd: 10.00
flags:
  supports_electronic_title: true
  requires_wet_signature: false
  abandoned_home_path: true
version_hash: "sha256:..."
```

## JSON Schema (abridged)
```json
{
  "type": "object",
  "required": ["state", "version", "forms", "submission", "fees", "flags"],
  "properties": {
    "state": { "type": "string", "pattern": "^[A-Z]{2}$" },
    "version": { "type": "string" },
    "forms": { "type": "object" },
    "submission": { "type": "object" },
    "notices": { "type": "object" },
    "timelines": { "type": "object" },
    "rejection_codes": { "type": "array" },
    "evidence_requirements": { "type": "object" },
    "fees": { "type": "object" },
    "flags": { "type": "object" },
    "version_hash": { "type": "string" }
  }
}
```

## Validation Rules
- `version_hash` must equal sha256 of file content sans hash line.
- `forms.*.fee_usd` matches `fees` totals; validator checks consistency.
- `channel_priority` only includes enabled submission types.
- `wait_days` integers; no negative values.
- All monetary values as numbers (USD); UI converts to cents.

## Release Process
- Edit YAML → run `pnpm rules:validate` → bump `version` → commit.
- Publish `@mh/rules` package with embedded YAML + generated JSON for KV cache.
- Worker deploy pins `rules_version` and `version_hash`; tasks record both.
