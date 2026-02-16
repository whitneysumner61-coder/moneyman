# Task Dependency Graphs (Reference)

## Duplicate Title (API State) DAG
```mermaid
flowchart TD
  intake --> validate
  validate --> status
  status --> branch{path}
  branch --> dup[duplicate_packet]
  branch --> payoff[payoff_packet]
  branch --> abandon[abandon_notice]
  dup --> mail[mail_notice?]
  payoff --> mail
  abandon --> mail
  mail --> submit[submit_duplicate/affidavit]
  submit --> track[track_status_webhook/cron]
  track --> done[deliver_title]
```

## Manual State DAG
```mermaid
flowchart TD
  intake --> validate
  validate --> gather[gather_required_docs]
  gather --> packet[render_packet]
  packet --> labels[create_certified_labels]
  labels --> wait[statutory_wait]
  wait --> file[file_affidavit/duplicate]
  file --> track[track_mail_delivery]
  track --> done
```

## Dependency Rules
- Partition key: `tenant_id:home_id`; tasks for a home run sequentially.
- Parallelism allowed inside independent steps (e.g., generating multiple notices) but results must merge into a single `task_event`.
- Each step outputs `{status, data, events[]}`; reducers update projections.
- Any step may emit `needs_human` to pause DAG; UI shows required actions.

## DAG Validation
- CLI lints DAGs to ensure: no orphan steps, branch coverage complete, terminal node present, max depth under configurable limit, and no forbidden cycles.
