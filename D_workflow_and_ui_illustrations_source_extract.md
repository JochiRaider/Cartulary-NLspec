# Appendix D: Workflow and UI Illustrations Source Extract

This appendix is **non-normative**.

It preserves the workflow sequence diagrams, UI mockups, and explanatory interaction notes from the exploratory source artifact.

## 8. Record lifecycle and IR workflow model

### Lifecycle

The workflow phrase below is explanatory only. The authoritative contract is Core 03 §6.

**rough capture → enriched → linked → reviewed → superseded / rolled back**

In the current profile, `linked` is a derived milestone rather than a persisted `capture_state`, and `rolled back` is a reviewer-history outcome rather than a persisted lifecycle state. The important point is that the rough capture remains recoverable. Normalization adds structure; it does not erase the original analyst input.

### 1. Rapid creation of a timeline event

```mermaid
sequenceDiagram
    participant A as Analyst A
    participant UI as Browser grid
    participant App as App API
    participant PG as Postgres

    A->>UI: Type "Possible VPN logon by jdoe? on WS-023?" in blank row
    UI->>App: create timeline event + unresolved mention tokens
    App->>PG: insert change_set
    App->>PG: insert records + timeline_events + entity_mentions
    App->>PG: insert change_set_mutations + record_revisions
    App->>PG: rebuild timeline_grid_projection(row)
    PG-->>App: commit
    App-->>UI: saved row + row_version
    App-->>UI: websocket update to other viewers
```

Concrete scenario: Analyst A creates a row with a nullable `occurred_at`, summary text, and raw mention tokens `jdoe?` and `WS-023?`. The system does **not** block on missing canonical identity/host.

### 2. Attachment of a screenshot or other evidence object

```mermaid
sequenceDiagram
    participant A as Analyst A
    participant UI as Browser grid
    participant App as App API
    participant OBJ as Object store
    participant PG as Postgres

    A->>UI: Paste or drag screenshot onto selected row
    UI->>App: create blob slot {incident_id, client_txn_id, byte_size, optional hints}
    App->>PG: insert pending object_blobs row
    App-->>UI: upload_target + object_blob_id + accepted_contract
    UI->>OBJ: upload binary
    UI->>App: POST /api/v1/evidence-records/{record_id}/attach-blob {object_blob_id, base_row_version, client_txn_id}
    App->>PG: insert records + evidence_records + record_links + change_set_mutations + record_revisions
    App->>PG: rebuild timeline_grid_projection(row)
    PG-->>App: commit
    App-->>UI: evidence_count=1 and preview affordance available when previewable
```

Important design choice: upload is **two-step** so incomplete uploads do not leave fake evidence attached. The create request carries the incident anchor, an idempotency key, one declared size contract, and optional advisory or integrity hints, and the response echoes `accepted_contract` so later finalization can compare against one server-accepted contract. When finalizing onto an existing evidence record, the public step-2 route is `POST /api/v1/evidence-records/{record_id}/attach-blob`, and that step uses record-scoped optimistic concurrency via `base_row_version` plus route-scoped `client_txn_id` idempotency. Replaying the same create request across a lost response returns the same slot; replay does not refresh an expired target; terminal size or expected-hash mismatch at finalization fails the slot rather than attaching evidence anyway. Abandoned pending blobs are cleaned up later. The same evidence model also supports requested-but-not-yet-received evidence: an evidence record can be created in `requested` or `pending_receipt` state with no blob, then later advanced to `received` or `available` as custody events and uploads occur.

### 2a. Evidence preview and download access

```mermaid
sequenceDiagram
    participant UI as Browser grid / inspector
    participant App as App API
    participant PG as Postgres
    participant OBJ as Object store

    UI->>App: POST /preview-handle {} or POST /download-handle {}
    App->>PG: validate session, incident membership, evidence/blob state
    App-->>UI: opaque same-origin href + expires_at + single_use + preview/media metadata
    UI->>App: GET /api/v1/evidence-handles/{handle_token}
    App->>PG: re-check session, membership, and current evidence/blob state
    alt redeem allowed
        App->>OBJ: fetch source bytes or derived safe preview bytes
        OBJ-->>App: bytes
        App-->>UI: inline preview bytes or validated download response
    else blocked or unsupported
        App-->>UI: explicit error; no silent fallback
    end
```

The handle returned to the browser is same-origin and opaque. The browser redeems it through the application, not by constructing raw object-store URLs, and blocked preview states surface explicitly instead of silently collapsing into download. Handle issuance intentionally uses `{}` only, does not take `client_txn_id`, and mints a fresh handle on each success rather than using issuance idempotency.

### 3. Later linkage of the event to canonical host and identity records

```mermaid
sequenceDiagram
    participant B as Analyst B
    participant UI as Browser inspector
    participant App as App API
    participant PG as Postgres

    B->>UI: Open row inspector and click unresolved "jdoe?" / "WS-023?"
    UI->>App: search aliases and candidate records
    App->>PG: query identities, hosts, aliases, mentions
    PG-->>App: candidate matches
    UI->>App: resolve identity to john.doe@corp.example
    UI->>App: create or resolve host to WS-023.corp.example
    App->>PG: insert/update host/identity records
    App->>PG: update entity_mentions(resolved_record_id, status)
    App->>PG: insert record_links(event->identity, event->host)
    App->>PG: insert change_set_mutations + record_revisions and rebuild projection row
    PG-->>App: commit
    App-->>UI: row now shows canonical chips instead of unresolved tokens
```

This is the core progressive-structuring workflow. The timeline row stays fast to create, but later becomes relationally useful.

### 3.0 Bootstrap first deployment admin during startup

```mermaid
sequenceDiagram
    participant Proc as Process start
    participant App as App startup
    participant PG as Postgres
    participant FS as Bootstrap manifest file

    Proc->>App: start
    App->>App: validate deployment config
    App->>PG: query active deployment admins
    App->>PG: query bootstrap-completion state
    alt zero active admins and no completion marker
        App->>FS: read configured manifest path
        App->>App: validate cartulary.bootstrap_admin.v1
        App->>PG: create user + bootstrap marker + admin audit event in one transaction
        PG-->>App: commit
    else active admin exists
        App->>App: skip bootstrap consumption
    else completion marker already exists and no active admin remains
        App->>App: fail closed
    end
    App->>App: start HTTP / WebSocket / background-job listeners only after successful preflight
```

The bootstrap-created user's first login then follows the existing TOTP setup flow in **3A. First login requiring TOTP setup** unchanged.

### 3A. First login requiring TOTP setup

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Login screen
    participant App as App API
    participant PG as Postgres

    U->>UI: Submit valid email and password
    UI->>App: POST /api/v1/auth/login
    App->>PG: verify local password and inspect MFA enrollment state
    PG-->>App: local account valid, mfa_required=true, no active TOTP
    App-->>UI: 401 mfa_setup_required + bootstrap_token + bootstrap_expires_at
    U->>UI: Continue TOTP setup
    UI->>App: POST /api/v1/auth/mfa/totp/begin {bootstrap_token}
    App->>PG: create pending enrollment + issue secret material
    PG-->>App: pending enrollment
    App-->>UI: enrollment_id + secret_base32 + otpauth_uri
    U->>UI: Enter six-digit code from authenticator
    UI->>App: POST /api/v1/auth/mfa/totp/complete {bootstrap_token, enrollment_id, code}
    App->>PG: activate TOTP and clear pending state
    PG-->>App: commit
    App-->>UI: success, no session issued
    U->>UI: Submit email and password again
    UI->>App: POST /api/v1/auth/login + second_factor
    App-->>UI: authenticated session
```

This keeps the base profile on an administrator-assisted recovery model while still making first-time MFA enrollment deterministic and interoperable.

### 3B. Lost-device recovery through administrator TOTP reset

```mermaid
sequenceDiagram
    participant A as Deployment admin
    participant UI as Admin console
    participant App as App API
    participant PG as Postgres
    participant U as User

    A->>UI: Reset user's TOTP credential
    UI->>App: POST /api/v1/users/{user_id}/mfa/totp/reset
    App->>PG: clear active and pending TOTP state + revoke all sessions
    PG-->>App: commit
    App-->>UI: safe user resource returned
    U->>UI: Submit valid email and password later
    UI->>App: POST /api/v1/auth/login
    App->>PG: verify password and inspect MFA enrollment state
    PG-->>App: mfa_required=true, no active TOTP
    App-->>UI: 401 mfa_setup_required + bootstrap_token + bootstrap_expires_at
    U->>UI: complete begin/complete TOTP setup flow
    UI->>App: POST /api/v1/auth/mfa/totp/begin then /complete
    App->>PG: activate replacement factor
    PG-->>App: commit
    U->>UI: log in normally with new factor
```

This recovery path stays outside the workbook hot path and keeps deployment-local credential recovery distinct from incident-scoped authorization.

### 3a. Create party from requester or source text

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Browser inspector
    participant App as App API
    participant PG as Postgres

    A->>UI: Click "Create party from text" on requester/source text
    UI->>App: submit source-preserving text + optional explicit fields
    App->>PG: search same-incident parties by normalized primary_email / external_ref
    alt unique exact match exists
        PG-->>App: existing party record
        App->>PG: update referencing record *_party_id only
    else no unique exact match
        App->>PG: insert records + parties
        App->>PG: update referencing record *_party_id only
    end
    App->>PG: insert change_set_mutations + record_revisions and rebuild projection row
    PG-->>App: commit
    App-->>UI: row keeps original text and now shows linked party chip or equivalent link state
```

This flow preserves the raw requester or source wording while adding a stable same-incident `party_id`.

### 3b. Link existing party without blocking row capture

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Browser inspector
    participant App as App API
    participant PG as Postgres

    A->>UI: Open requester/collector/source field on an already-saved row
    UI->>App: query existing incident-scoped parties
    App->>PG: search `party_grid_projection` + canonical party records
    PG-->>App: candidate parties
    A->>UI: Select existing party
    UI->>App: patch only the hidden *_party_id field
    App->>PG: update referencing record without rewriting preserved text
    App->>PG: insert change_set_mutations + record_revisions and rebuild projection row
    PG-->>App: commit
    App-->>UI: row remains in place; text stays visible; linked party is now available for pivots or queues
```

This flow layers canonical linkage over already-captured text without requiring the analyst to leave the workbook surface or re-enter the row.

### 3c. Clear requester, collector, or source party link with `value: null`

The examples below are illustrative only. The authoritative wire contract remains Core 01 §18B, Core 01 §19, and the shared mutation rules in Core 01 §3.3.5.

#### Example: clear `task.requester_party_id`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.task_requests.v1",
  "base_row_version": 22,
  "client_txn_id": "txn_task_requester_party_clear_01",
  "changes": [
    {
      "field_key": "task.requester_party_id",
      "value": null
    }
  ]
}
```

#### Example: clear `evidence.collector_party_id`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.evidence.v1",
  "base_row_version": 9,
  "client_txn_id": "txn_evidence_collector_party_clear_01",
  "changes": [
    {
      "field_key": "evidence.collector_party_id",
      "value": null
    }
  ]
}
```

#### Example: clear `evidence.source_party_id`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.evidence.v1",
  "base_row_version": 10,
  "client_txn_id": "txn_evidence_source_party_clear_01",
  "changes": [
    {
      "field_key": "evidence.source_party_id",
      "value": null
    }
  ]
}
```

### 3d. Clear both preserved party text and linked party ref in one patch

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.task_requests.v1",
  "base_row_version": 23,
  "client_txn_id": "txn_task_requester_clear_both_01",
  "changes": [
    {
      "field_key": "task.requester_party_text",
      "value": null
    },
    {
      "field_key": "task.requester_party_id",
      "value": null
    }
  ]
}
```

This example shows the ordinary `Clear both` shape: one record patch with two direct-write field changes and no bespoke action route.

### 3e. Clear `task.decision_record_id` with `value: null`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.task_requests.v1",
  "base_row_version": 24,
  "client_txn_id": "txn_task_decision_clear_01",
  "changes": [
    {
      "field_key": "task.decision_record_id",
      "value": null
    }
  ]
}
```

### 3f. Coordination collection patch examples

The examples below are illustrative only. The authoritative wire contract remains Core 01 §19 plus the shared mutation rules in Core 01 §3.3.5.

#### Example: add one audience party ref on `comm_log.audience_party_ids`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.comm_log.v1",
  "base_row_version": 12,
  "client_txn_id": "txn_comm_log_party_01",
  "changes": [
    {
      "field_key": "comm_log.audience_party_ids",
      "action_payload": {
        "kind": "collection_actions_v1",
        "actions": [
          { "op": "add_party_ref", "party_id": "pty_01" }
        ]
      }
    }
  ]
}
```

Matching read-side fragment:

```json
{
  "field_key": "comm_log.audience_party_ids",
  "value": {
    "kind": "collection_value_v1",
    "ordered": false,
    "items": [
      {
        "item_ref": "party_ref:pty_01",
        "item_kind": "party_ref",
        "display_text": "Email Distribution Team",
        "party_id": "pty_01"
      }
    ]
  }
}
```

#### Example: add one pending evidence ref on `status_review.pending_evidence_ids`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.status_review.v1",
  "base_row_version": 7,
  "client_txn_id": "txn_status_review_evidence_01",
  "changes": [
    {
      "field_key": "status_review.pending_evidence_ids",
      "action_payload": {
        "kind": "collection_actions_v1",
        "actions": [
          { "op": "add_record_ref", "linked_record_id": "rec_evidence_01" }
        ]
      }
    }
  ]
}
```

Matching read-side fragment:

```json
{
  "field_key": "status_review.pending_evidence_ids",
  "value": {
    "kind": "collection_value_v1",
    "ordered": false,
    "items": [
      {
        "item_ref": "record_ref:rec_evidence_01",
        "item_kind": "record_ref",
        "display_text": "EDR package for WS-023",
        "linked_record_id": "rec_evidence_01"
      }
    ]
  }
}
```

#### Example: add then remove one open risk ref on `handoff.open_risk_refs`

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.handoff.v1",
  "base_row_version": 19,
  "client_txn_id": "txn_handoff_risk_add_01",
  "changes": [
    {
      "field_key": "handoff.open_risk_refs",
      "action_payload": {
        "kind": "collection_actions_v1",
        "actions": [
          { "op": "add_risk_ref", "risk_ref_text": "Pending confirmation of outbound data access scope" }
        ]
      }
    }
  ]
}
```

```http
PATCH /api/v1/records/{record_id}
```

```json
{
  "view_schema_id": "cartulary.view.handoff.v1",
  "base_row_version": 20,
  "client_txn_id": "txn_handoff_risk_remove_01",
  "changes": [
    {
      "field_key": "handoff.open_risk_refs",
      "action_payload": {
        "kind": "collection_actions_v1",
        "actions": [
          { "op": "remove_risk_ref", "item_ref": "risk_ref:rsk_01" }
        ]
      }
    }
  ]
}
```

Matching read-side fragment:

```json
{
  "field_key": "handoff.open_risk_refs",
  "value": {
    "kind": "collection_value_v1",
    "ordered": false,
    "items": [
      {
        "item_ref": "risk_ref:rsk_01",
        "item_kind": "risk_ref",
        "display_text": "Pending confirmation of outbound data access scope",
        "risk_ref_id": "rsk_01",
        "risk_ref_text": "Pending confirmation of outbound data access scope"
      }
    ]
  }
}
```

### 4. Review, version inspection, and rollback of a mistaken edit

```mermaid
sequenceDiagram
    participant R as Reviewer
    participant UI as History panel
    participant App as App API
    participant PG as Postgres

    R->>UI: Open history for event row
    UI->>App: fetch row-centric history for record and linked mutations
    App->>PG: query change_set_mutations + record_revisions + change_sets
    PG-->>App: ordered row history with actor/time and reversible diffs
    R->>UI: select mistaken host-link change and click rollback
    UI->>App: rollback selected history entry or whole change set
    App->>PG: insert new change_set(source='rollback')
    App->>PG: reverse selected mutation / restore prior row snapshot / soft-delete mistaken link
    App->>PG: insert new change_set_mutations + record_revisions and rebuild projection
    PG-->>App: commit
    App-->>UI: current row updated, history preserved
```

#### Reviewer UI rollback granularity

The reviewer UI MUST remain row-centric, but it MUST NOT be limited to whole-row restore. In MVP, the history panel MUST show actor, timestamp, operation, and a diff summary expanded to changed field/link/mention/tag/evidence-entry units.

The reviewer UI MUST allow rollback of a single logical history entry when that entry maps to one reversible mutation target, including one scalar field edit, one link add/remove, one tag add/remove, one mention resolve/dismiss/restore, one auto-resolution or auto-match, or one evidence attach/detach association. The UI MUST also expose whole-row restore and whole-change-set rollback as secondary actions for multi-target or destructive changes.

Arbitrary user-selected subsets of fields from historical snapshots are not required in MVP. Rollback remains a new attributed action by the reviewer, not a hidden database revert.

#### Destructive-operation contention precedence

```mermaid
sequenceDiagram
    participant A as Reviewer A
    participant B as Reviewer B
    participant UI as Browser / inspector
    participant App as App API
    participant PG as Postgres

    A->>UI: Start restore, rollback, or merge
    UI->>App: destructive-operation request A
    App->>PG: acquire protected-set locks in canonical record_id order
    PG-->>App: locks acquired
    B->>UI: Submit overlapping destructive-operation request B
    UI->>App: destructive-operation request B
    App->>PG: try acquire same protected-set locks
    PG-->>App: lock unavailable
    App-->>UI: 409 record_locked + retryable=true
    A->>App: continue operation A
    App->>PG: commit and release locks
    PG-->>App: complete
    B->>UI: retry request B without refreshed inputs
    UI->>App: destructive-operation request B retry
    App->>PG: acquire locks, then re-read authoritative state
    PG-->>App: current state shows stale version or route precondition failure
    App-->>UI: 409 row_version_conflict or route-specific precondition error
```

This sequence is explanatory only. The owner contract makes `record_locked` the fail-fast outcome while overlapping destructive work is still in flight on the same protected set. After those locks are released, the same request falls through to the ordinary stale-version or route-precondition path.

### 4a. Timeline supersede with a direct replacement relation

```mermaid
sequenceDiagram
    participant R as Reviewer
    participant UI as Inspector / history surface
    participant App as App API
    participant PG as Postgres

    R->>UI: Choose Supersede and optionally select replacement row
    UI->>App: POST /api/v1/records/{record_id}/supersede {base_row_version, client_txn_id, reason, replacement_record_id?}
    App->>PG: validate reviewer role, lifecycle state, and replacement-target guards
    App->>PG: insert change_set
    alt replacement selected
        App->>PG: insert record_links(replacement -> superseded, link_type='supersedes')
    end
    App->>PG: update timeline_events.capture_state='superseded'
    App->>PG: insert change_set_mutations + record_revisions and rebuild projection row
    PG-->>App: commit
    App-->>UI: success includes committed replacement_record_id
    App-->>UI: row history shows capture_state change plus replacement-link unit
```

This illustration keeps the UI obligation lightweight: the reviewer invokes supersede from the inspector, history surface, or another reviewer-only non-grid surface; an optional replacement row can be selected there; and when the action succeeds the selected replacement remains recoverable nearby without adding a default visible Timeline column. Correction is rollback and re-supersede rather than direct editing of the hidden replacement field on a superseded row.

### Bulk paste/import from existing spreadsheet or clipboard

- **Clipboard paste is day-one functionality.**\
  Pasting TSV/CSV into the timeline sheet should create/update multiple rows starting from the selected cell.
- Known columns map directly.
- Unknown columns are stored into `raw_capture.import_columns`.
- Host/identity text from pasted cells follows the same `entity_binding_mode` contract as interactive edits: `mention_origin` fields create unresolved `entity_mentions`; `entity_origin` fields create or upsert host/identity records.
- Repeated identical mention values across different source rows remain separate mention rows with distinct source locators.
- The entire paste is one visible `change_set`, with ordered mutation entries and one row revision per affected record.

Clipboard paste validates the hot-path grid experience. By itself it does not prove brownfield workbook migration readiness.

For file-based onboarding, keep workbook interaction on the grid surface and isolate workbook parsing inside a dedicated imports module. Clipboard-driven ingest and file-based import should still share the same mapping engine and a canonical tabular source model so behavior does not drift across intake paths.

For file-based import, start with a bounded contract rather than promising full spreadsheet fidelity. The first assistant should support CSV file import plus selected-sheet or selected-region XLSX import, with preview, header mapping, provenance capture, and unknown-column preservation. It should prioritize sheets or regions that map to timeline, systems/hosts, accounts/identities, indicators, evidence tracker, and VERIS-like summaries when present. Mapping contracts, not sheet labels alone, should decide whether a source column is `mention_origin` or `entity_origin`.

Formulas, macros, workbook automation, external links, comments, pivot tables, charts, workbook protection, and merged-cell layout semantics should not be treated as live workbook logic. Formula cells should be imported as inert values or raw source metadata only, with visible warnings or explicit rejection when a feature is unsupported. File-based import should not auto-resolve host/account aliases; imported tokens should remain unresolved mentions until an analyst resolves them explicitly.

#### Example preview payload

```json
{
  "data": {
    "import_session_id": "imp_sess_01",
    "import_unit_id": "imp_unit_03",
    "locator_kind": "xlsx_region",
    "locator": {
      "sheet_name": "Timeline",
      "rect_a1": "A1:C4"
    },
    "source_rect_a1": "A1:C4",
    "header_row_ref": 1,
    "data_start_row_ref": 2,
    "inferred_row_count": 3,
    "inferred_column_count": 3,
    "warning_codes": [],
    "unit_status": "selected",
    "columns": [
      { "source_column_ordinal": 1, "source_header_text": "Occurred At" },
      { "source_column_ordinal": 2, "source_header_text": "Summary" },
      { "source_column_ordinal": 3, "source_header_text": "User" }
    ],
    "preview_rows": [
      {
        "source_row_ref": 2,
        "cells": [
          { "source_column_ordinal": 1, "display_text": "2026-03-14T09:14:00Z", "cell_kind": "datetime" },
          { "source_column_ordinal": 2, "display_text": "VPN login", "cell_kind": "string" },
          { "source_column_ordinal": 3, "display_text": "jdoe", "cell_kind": "string" }
        ]
      },
      {
        "source_row_ref": 3,
        "cells": [
          { "source_column_ordinal": 1, "display_text": "", "cell_kind": "blank" },
          { "source_column_ordinal": 2, "display_text": "Possible follow-up action", "cell_kind": "string" },
          { "source_column_ordinal": 3, "display_text": "asmith", "cell_kind": "string" }
        ]
      }
    ],
    "truncated": false
  }
}
```

#### Example import_session resource

```json
{
  "data": {
    "import_session_id": "imp_sess_01",
    "incident_id": "inc_2026_017",
    "created_by_user_id": "usr_analyst_01",
    "created_at": "2026-03-14T09:10:00Z",
    "source_file_kind": "xlsx",
    "original_filename": "timeline.xlsx",
    "source_content_sha256": "1111111111111111111111111111111111111111111111111111111111111111",
    "parser_profile_id": "example_parser_profile_id",
    "parser_version": "2026-04-02",
    "assistant_profile": "phase2_workbook_import_v1",
    "session_status": "mapped",
    "selected_unit_ids": ["imp_unit_03"],
    "blocking_diagnostics": [
      {
        "code": "import_apply_blocked",
        "reason_code": "unit_not_ready",
        "message": "Selected unit is not ready to apply.",
        "import_unit_id": "imp_unit_04"
      }
    ],
    "nonblocking_warning_codes": []
  }
}
```

#### Example import_unit resource before mapping approval

```json
{
  "data": {
    "import_unit_id": "imp_unit_03",
    "import_session_id": "imp_sess_01",
    "locator_kind": "xlsx_region",
    "locator": {
      "sheet_name": "Timeline",
      "rect_a1": "A1:C4"
    },
    "source_rect_a1": "A1:C4",
    "header_row_ref": 1,
    "data_start_row_ref": 2,
    "inferred_row_count": 3,
    "inferred_column_count": 3,
    "warning_codes": [],
    "unit_status": "selected"
  }
}
```

#### Example import_unit resource after mapping approval

```json
{
  "data": {
    "import_unit_id": "imp_unit_03",
    "import_session_id": "imp_sess_01",
    "locator_kind": "xlsx_region",
    "locator": {
      "sheet_name": "Timeline",
      "rect_a1": "A1:C4"
    },
    "source_rect_a1": "A1:C4",
    "header_row_ref": 1,
    "data_start_row_ref": 2,
    "inferred_row_count": 3,
    "inferred_column_count": 3,
    "warning_codes": [],
    "unit_status": "ready",
    "mapping_fingerprint": "2222222222222222222222222222222222222222222222222222222222222222",
    "approved_mapping": {
      "target_view_schema_id": "cartulary.view.timeline.v1",
      "unknown_column_policy": "preserve_raw_capture",
      "source_columns": [
        {
          "source_column_ordinal": 1,
          "source_header_text": "Occurred At",
          "field_key": "timeline.occurred_at",
          "entity_binding_mode": null,
          "transform_id": null,
          "transform_options": {},
          "empty_value_policy": "write_null"
        },
        {
          "source_column_ordinal": 2,
          "source_header_text": "Summary",
          "field_key": "timeline.summary",
          "entity_binding_mode": null,
          "transform_id": "trim_v1",
          "transform_options": {},
          "empty_value_policy": "omit_field"
        },
        {
          "source_column_ordinal": 3,
          "source_header_text": "User",
          "field_key": "timeline.identity_refs",
          "entity_binding_mode": "mention_origin",
          "transform_id": "trim_v1",
          "transform_options": {},
          "empty_value_policy": "omit_field"
        }
      ]
    }
  }
}
```

#### Example select/skip flow before apply

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Browser grid
    participant App as App API
    participant PG as Postgres

    A->>UI: Create import session and review discovered units
    A->>UI: Preview unit A
    UI->>App: GET /api/v1/import-sessions/{import_session_id}/units/{import_unit_id}/preview
    App-->>UI: preview payload with columns[] and preview_rows[]
    A->>UI: Approve mapping for unit A
    UI->>App: PUT /api/v1/import-sessions/{import_session_id}/units/{import_unit_id}/mapping
    App->>PG: persist approved mapping and recompute unit status
    PG-->>App: commit
    App-->>UI: updated durable unit resource
    A->>UI: Select unit A
    UI->>App: POST /api/v1/import-sessions/{import_session_id}/units/{import_unit_id}/select {client_txn_id}
    App->>PG: update selected_unit_ids[] and recompute session/unit status
    PG-->>App: commit
    App-->>UI: selected_unit_ids[] + updated unit
    A->>UI: Skip unit B
    UI->>App: POST /api/v1/import-sessions/{import_session_id}/units/{other_import_unit_id}/skip {client_txn_id, reason}
    App->>PG: persist skipped state without deleting prior mapping
    PG-->>App: commit
    App-->>UI: selected_unit_ids[] + updated unit
    A->>UI: Apply selected units
    UI->>App: POST /api/v1/import-sessions/{import_session_id}/apply {client_txn_id}
    App->>PG: read persisted selected_unit_ids[]
    App-->>UI: 202 Accepted + common job resource
```

### Illustrative terminal job-result summaries

The fragments below are illustrative only. The authoritative owner is Core 01 §3.3.9.1 and §17.

#### `snapshot_created`

```json
{
  "job_id": "job_snapshot_01",
  "status": "succeeded",
  "result_summary": {
    "code": "snapshot_created",
    "message": "Snapshot created.",
    "resource_refs": [
      {
        "kind": "snapshot",
        "id": "snap_01",
        "route": "/api/v1/snapshots/snap_01"
      }
    ]
  }
}
```

#### `import_session_partially_applied`

```json
{
  "job_id": "job_import_apply_01",
  "status": "succeeded",
  "result_summary": {
    "code": "import_session_partially_applied",
    "message": "Selected units applied with partial success.",
    "resource_refs": [
      {
        "kind": "import_session",
        "id": "imp_01",
        "route": "/api/v1/import-sessions/imp_01"
      }
    ]
  }
}
```

#### `incident_bundle_imported`

```json
{
  "job_id": "job_bundle_import_01",
  "status": "succeeded",
  "result_summary": {
    "code": "incident_bundle_imported",
    "message": "Incident bundle imported.",
    "resource_refs": [
      {
        "kind": "incident",
        "id": "inc_01",
        "route": "/api/v1/incidents/inc_01"
      }
    ]
  }
}
```

#### `job_canceled`

```json
{
  "job_id": "job_any_01",
  "status": "canceled",
  "result_summary": {
    "code": "job_canceled",
    "message": "Job canceled before completion."
  }
}
```

UI note: on receipt of a terminal result, the client keeps the current workbook surface in place, surfaces known refs as non-modal completion chips or links, degrades unknown refs to `message`-only, and never auto-navigates from the incoming `route`.

### Non-Timeline create-policy examples

- **Linked note from context.** Selecting a Timeline, Host, Identity, or Evidence row may preseed a linked-record reference for `add linked note`, but the note does not commit until `note.title` or `note.body` is non-empty after the bound `single_line_title_v1` or `multiline_body_v1` normalization. A linked but otherwise empty note remains an unsaved draft, not a saved record.
- **Evidence request without a blob.** A blank-row Evidence create with no explicit lifecycle choice commits only when at least one writable evidence field remains present after its bound contract normalization. In the current profile, `evidence.title` uses `single_line_title_v1`, `evidence.storage_ref` uses `locator_text_v1`, and `evidence.collector_party_text` plus `evidence.source_party_text` use `party_text_v1`. Optional hidden `evidence.collector_party_id` and `evidence.source_party_id` links may supplement that text through same-surface enrichment, but they do not by themselves satisfy the minimum create signal and they do not rewrite the preserved text. The committed row defaults `evidence.lifecycle_state` to `requested`, and omitted `requested_at` defaults to the commit timestamp. A create lacking both a qualifying field signal and a finalized blob leaves no evidence row.
- **Canonical indicator create gate.** A blank-row Indicators create is refused until canonical identity is determinable from `indicator.indicator_type`, `indicator.value_kind`, `indicator.display_value`, and `indicator.normalized_value` when required. `indicator.hash_algorithm` and `indicator.hash_value` are pairwise; `defanged_value` and `stix_pattern` may be present but do not satisfy the identity gate.
- **Assessment from selected subject.** Creating an assessment from a selected Host or Identity row may preseed `assessment.subject_ref` and `assessment.subject_type`, but the row commits only when `assessment.assessment_state` and non-empty `assessment.rationale` are also present. Omitted `assessed_at` and `assessor` default at commit time.
- **Task request from selected records.** A task-request create flow may preseed `task.linked_record_ids` or `task.decision_record_id`, but the row commits only when `task.title` remains present after `single_line_title_v1` normalization and `task.task_kind` is present. `task.requester_party_text`, `task.blocked_reason`, `task.external_ticket_ref`, and `task.closure_summary` follow `party_text_v1`, `reason_note_v1`, `locator_text_v1`, and `multiline_body_v1` when written. An optional hidden `task.requester_party_id` may supplement `task.requester_party_text`, but it does not by itself satisfy the minimum create signal and it does not rewrite the preserved text.
- **Decision from selected support.** A decision create flow may preseed `decision.support_refs`, but the row commits only when `decision.decision_type`, `decision.summary`, and `decision.rationale` remain present after the bound `single_line_title_v1` and `multiline_body_v1` normalization. Omitted `decision.status`, `decision.owner_user_id`, and `decision.decided_at` default to `proposed`, the current actor, and the commit timestamp.

### Auto-resolution policy for typed host/account strings

This revision resolves the MVP policy for alias auto-resolution.

The system MAY auto-resolve a typed host or identity token to an existing alias only in interactive mention-capture flows, and only when `auto_resolution_confidence = 100`.

`auto_resolution_confidence = 100` applies only when all of the following are true:

- the edited cell determines the expected entity type (`Hosts` => `host`, `Identities` => `identity`) and candidate matching is limited to that type within the same incident;
- the token matches exactly one existing alias after deterministic normalization limited to case-folding plus whitespace collapse;
- the raw token contains no explicit uncertainty marker such as `?`, `~`, `maybe`, or `prob`;
- the target record is not soft-deleted, merged, retired, or disabled;
- no competing candidate record remains after normalization.

Anything below `100` MAY drive ranking or suggestions, but MUST NOT create or update `record_links`, set `entity_mentions.resolved_record_id`, or otherwise mutate resolution state without explicit analyst selection.

To preserve raw analyst input, auto-resolution MUST still insert an `entity_mentions` row for the typed token with `resolution_status='resolved'` and `resolved_record_id` set to the chosen record. The corresponding `record_links` row MUST use `provenance='auto_match'` and `confidence=100`.

Auto-resolution MAY occur only in:

- inline commit of a Timeline `Hosts` or `Identities` cell;
- interactive clipboard paste into those same relationship cells, where the resulting auto-resolutions are part of the same visible `change_set`.

Auto-resolution MUST NOT occur in:

- the inspector's explicit resolve flow;
- Hosts/Identities alias-edit cells;
- merge/dedupe workflows;
- file-based import through the Import Extension Profile;
- background jobs or async enrichment/cleanup;
- any workflow that would create a new canonical host/identity or edit alias rows without explicit analyst confirmation.

The UI MUST NOT silently auto-resolve. When auto-resolution occurs, the current sheet MUST show an immediate non-modal disclosure on the same surface that includes the raw token, the canonical target, the matched alias text, and direct `Undo` and `Review` actions. For batch paste, the disclosure MUST also include the number of tokens auto-resolved in that visible change set. The resolved chip or cell MUST remain inspectably marked as auto-resolved, and row history MUST preserve the raw token, matched alias text, `confidence=100`, and mutation source.

`Undo` from the immediate disclosure MUST restore the raw unresolved token, remove the auto-created link, and preserve focus and scroll position. After the immediate disclosure expires, the user MUST still be able to choose `Revert to unresolved` from the chip context or row history in no more than two actions. That later correction is a new attributed revision; it MUST NOT rewrite history.

### Unknown or ambiguous fields

Unknown values must remain valid:

- `occurred_at` may be null.
- summary may be null if another field or attachment exists.
- host/account text may remain unresolved.
- confidence can be left unset.
- details may be plain text without structure.

- Timestamp entry should also preserve visible interpretation state: invalid timestamp text remains unsaved local state until corrected or discarded, and clearing a timestamp cell serializes as explicit JSON `null` only when the bound field declares `clearable=true`. When the bound field is not clearable, the attempted save fails closed and the visible local state remains unsaved rather than being silently coerced.

### End-to-end attribution

Every step above writes the actor’s `user_id` into:

- current record envelope,
- `change_sets.actor_user_id`,
- `record_revisions`,
- link and tag creation metadata,
- object blob and evidence metadata.


### Reference-pack lifecycle sketch

The diagram below is explanatory only. The authoritative contract is Core 01 §11.3 and §11.4.

```mermaid
stateDiagram-v2
    [*] --> staged
    staged --> verified_available: verification passed
    staged --> failed: verification failed
    staged --> missing: payload unavailable
    verified_available --> active: explicit activation
    active --> verified_available: activate different verified version
    verified_available --> disabled: admin disable
    active --> disabled: admin disable
    disabled --> verified_available: re-enable / re-verify
    verified_available --> failed: later integrity failure
    active --> failed: later integrity failure
    disabled --> failed: later integrity failure
    verified_available --> missing: payload unavailable
    active --> missing: payload unavailable
    disabled --> missing: payload unavailable
```

### Snapshot artifact lifecycle sketch

The diagram below is explanatory only. The authoritative contract is Core 01 §10.2 through §10.5 and Core 04 §2.1.

```mermaid
stateDiagram-v2
    [*] --> pending_approval: render complete
    pending_approval --> approved: approvals satisfied
    approved --> published: explicit publish
    approved --> invalidated: superseded or approval no longer applies
    published --> invalidated: superseded or approval no longer applies
    invalidated --> pending_approval: new render candidate
```

#### Example snapshot resource

```json
{
  "data": {
    "snapshot_id": "snap_2026_017_01",
    "incident_id": "inc_2026_017",
    "created_by_user_id": "usr_reviewer_01",
    "created_at": "2026-03-14T12:00:00Z",
    "snapshot_at": "2026-03-14T12:00:00Z",
    "source_change_set_high_watermark": "cs_000184",
    "derivation_version": "derivation_v1",
    "export_model_sha256": "3333333333333333333333333333333333333333333333333333333333333333"
  }
}
```

#### Example release resource

```json
{
  "data": {
    "release_id": "rel_2026_017_01",
    "incident_id": "inc_2026_017",
    "snapshot_id": "snap_2026_017_01",
    "snapshot_at": "2026-03-14T12:00:00Z",
    "source_change_set_high_watermark": "cs_000184",
    "derivation_version": "derivation_v1",
    "export_model_sha256": "3333333333333333333333333333333333333333333333333333333333333333",
    "template_id": "tmpl_exec_html",
    "template_version": "3",
    "redaction_profile_id": "redact_external_a",
    "redaction_profile_version": "5",
    "output_kind": "html",
    "release_scope": "internal_review",
    "output_sha256": "4444444444444444444444444444444444444444444444444444444444444444",
    "release_state": "pending_approval",
    "created_by_user_id": "usr_reviewer_01",
    "created_at": "2026-03-14T12:03:00Z",
    "approved_at": null,
    "invalidated_at": null,
    "published_at": null,
    "invalidation_reason": null
  }
}
```

#### Example approve success response

```json
{
  "data": {
    "release": {
      "release_id": "rel_2026_017_01",
      "incident_id": "inc_2026_017",
      "snapshot_id": "snap_2026_017_01",
      "snapshot_at": "2026-03-14T12:00:00Z",
      "source_change_set_high_watermark": "cs_000184",
      "derivation_version": "derivation_v1",
      "export_model_sha256": "3333333333333333333333333333333333333333333333333333333333333333",
      "template_id": "tmpl_exec_html",
      "template_version": "3",
      "redaction_profile_id": "redact_external_a",
      "redaction_profile_version": "5",
      "output_kind": "html",
      "release_scope": "internal_review",
      "output_sha256": "4444444444444444444444444444444444444444444444444444444444444444",
      "release_state": "approved",
      "created_by_user_id": "usr_reviewer_01",
      "created_at": "2026-03-14T12:03:00Z",
      "approved_at": "2026-03-14T12:05:00Z",
      "invalidated_at": null,
      "published_at": null,
      "invalidation_reason": null
    },
    "approval_progress": {
      "approval_recorded": true,
      "approval_requirements_satisfied": true,
      "resulting_release_state": "approved"
    }
  }
}
```

### Blob-upload and evidence lifecycle sketch

The diagram below is explanatory only. The authoritative contract is Core 02 §13 and Core 03 §8.

```mermaid
flowchart LR
    subgraph B[Blob upload machine]
        b1[pending]
        b2[available]
        b3[failed]
        b4[quarantined]
        b1 --> b2
        b1 --> b3
        b1 --> b4
    end

    subgraph E[Evidence lifecycle machine]
        e1[requested]
        e2[pending_receipt]
        e3[received]
        e4[available]
        e5[quarantined]
        e6[released]
        e1 --> e2
        e2 --> e3
        e3 --> e4
        e3 --> e5
        e4 --> e6
    end

    b2 -. finalize attach .-> e3
    b4 -. finalize attach as quarantined .-> e5
    b1 -. no attached evidence .-> x[not finalized]
    b3 -. no attached evidence .-> x
```


### Benchmark timing-state illustrations

This subsection is illustrative only. Core 04 §9 remains the normative owner of the claim-bearing benchmark profile, measurement-predicate registry, and timed or fixture-sensitive pass/fail semantics.

#### Blank-row creation timing boundary

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Timeline grid
    participant App as App API
    participant PG as Postgres

    A->>UI: Type one qualifying value into a blank Timeline row
    A->>UI: Press Enter
    Note over UI: Start timing at commit acceptance
    UI->>App: create row
    App->>PG: insert records + timeline row + revisions
    PG-->>App: commit
    App-->>UI: row payload with record_id + row_version
    Note over UI: Stop when committed row is visible with stable binding
```

Illustrative start state: authenticated session complete, incident already open, Timeline surface already loaded, and default sort, filter, and grouping state active.

Illustrative stop predicate: the committed row is visible, the entered value is rendered in the target field, and the new row is bound to stable `record_id` plus `row_version`.

#### View-change first-useful versus stable viewport

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Workbook surface
    participant App as App API
    participant PG as Postgres

    A->>UI: Submit sort, filter, or grouping change
    Note over UI: Start timing at submit
    UI->>App: query updated view
    App->>PG: read projection rows
    PG-->>App: first visible block
    App-->>UI: first visible block
    Note over UI: first useful viewport
    App->>PG: finish remaining ordered read
    PG-->>App: final ordered block
    App-->>UI: final ordered viewport state
    Note over UI: stable viewport
```

Illustrative first-useful predicate: the first visible row window for the requested state is rendered with stable `record_id` binding and working keyboard navigation.

Illustrative stable predicate: the visible row window and result ordering now match the final deterministic order, and no further reorder occurs without new user or server input.

#### Evidence-inspector metadata-shell boundary

```mermaid
sequenceDiagram
    participant A as Analyst
    participant UI as Inspector
    participant App as App API
    participant PG as Postgres

    A->>UI: Open inspector on a Timeline row linked to 100 evidence records
    Note over UI: Start timing at open action
    UI->>App: request inspector data
    App->>PG: fetch row summary + evidence metadata window
    PG-->>App: selected-row summary + evidence count + first list window
    App-->>UI: render inspector metadata shell
    Note over UI: Stop when metadata shell is visible
```

Illustrative metadata-shell predicate: the selected-row summary, total linked-evidence count, and first rendered evidence-list window are visible, and each evidence item in that first rendered window shows filename or media-type label, attachment state, and preview-handle availability. Binary preview bytes are not required for this stop condition.

## 9. UI concepts focused on preserving the spreadsheet feel

The UI should feel like a **workbook**, but the sheets are **saved views over projections**, not separate storage silos. The built-in tabs are intentionally few: Timeline, Hosts, Identities, Evidence, and Notes. Indicators and Assessments should remain contract-backed system views over related projections even when canonical indicators or assessments are first-class records underneath. Framework overlays such as ATT&CK or VERIS should start as contract-backed system views and only become dedicated tabs if usage justifies it. Required base coordination surfaces `cartulary.view.comm_log.v1`, `cartulary.view.handoff.v1`, `cartulary.view.status_review.v1`, and `cartulary.view.lesson.v1` are opened and addressed by their standardized `view_schema_id`; a saved view over one of those schemas is a separate preset or layout surface rather than the required base surface itself.

### UI concept 1: Primary workbook-style timeline view

```text
+------------------------------------------------------------------------------------------------------------------+
| Incident IR-2026-017 | Timeline* | Hosts | Identities | Evidence | Notes | [Search / filter] | A  B  R        |
+------------------------------------------------------------------------------------------------------------------+
| View: [Capture order v]  Sort: [Time v]  Group: [None v]  Filters: [Unresolved] [Has evidence] [Tag: rough]    |
+----+----------+-------------------------------+------------------+------------------+------+-----------+---------+
| #  | Time     | Summary                       | Hosts            | Identities       | Evd. | Tags      | Edited  |
+----+----------+-------------------------------+------------------+------------------+------+-----------+---------+
| 81 | 09:14?   | Possible VPN logon ...        | WS-023?          | jdoe?            | 1    | rough     | B 2m    |
|    |          | screenshot attached           |                  |                  |      |           |         |
| 82 |          | [type here…]                  |                  |                  |      |           |         |
+----+----------+-------------------------------+------------------+------------------+------+-----------+---------+
| Status: Saved | Analyst B is on row 81 | Enter=new row | Tab=next cell | Ctrl+V=paste | Space=preview |
+------------------------------------------------------------------------------------------------------------------+
```

#### Screen regions

- **Top bar**: incident identity, workbook tabs, search, presence avatars.
- **View bar**: saved view selector, sort, group, filter chips.
- **Grid**: primary work surface.
- **Status bar**: save/conflict state and keyboard hints.
- **Inspector drawer**: collapsible on the right, not shown above.

#### Inline editing behavior

- Selecting a cell and typing edits it immediately.
- Enter commits and moves vertically; Tab commits and moves horizontally.
- Typing in the blank row creates a real record as soon as there is one non-empty value.
- Cells with relationship semantics still accept raw typing; they do not force picker-first interaction.

#### Keyboard-first interactions

- Arrow keys move selection.
- Enter/Shift+Enter navigate rows.
- `Ctrl+V` pastes multi-cell blocks.
- `Ctrl+K` opens quick link/resolve for the current cell.
- `Space` previews linked evidence for the selected row.
- `Alt+H` opens history for the selected row.

#### Copy/paste and bulk editing

- Paste TSV/CSV directly from Excel into the visible grid.
- If the paste range exceeds existing rows, new rows are created automatically.
- Fill-down and multi-row tag assignment are supported from the selection model.
- Bulk edits are mutation batches, not hidden macros.

#### Sorting / filtering / grouping

- Column header click sorts.
- Filter chips apply without leaving the sheet.
- Timeline grouping is a presentation-only transform over the current filtered result set. It MUST NOT create, delete, or mutate source records, projection rows, links, or tags.
- Timeline sheets MUST support `Group: None` plus exactly one active grouping key in MVP. The active key MUST be stored as a stable contract value in `saved_views.query_json.group_by`, not as a visible label, and `Group: None` is represented by omission rather than by `null`.
- Allowed grouping keys for timeline sheets are:
  - `timeline.occurred_day` derived from `occurred_at` at day granularity
  - `timeline.recorded_day` derived from `recorded_at` at day granularity
  - `timeline.capture_state`
  - `timeline.has_evidence` where `evidence_count > 0`
  - `timeline.has_unresolved_mentions` where at least one `entity_mentions` row for the source record has `resolution_status='unresolved'`
- dismissed mentions do not contribute to this derived flag, and a row whose remaining mentions are all dismissed groups under `false`
- Grouping keys MUST be scalar, contract-backed values. Free-text columns and multi-valued relationship cells such as Hosts, Identities, and Tags are not eligible grouping keys in timeline sheets.
- Group order MUST be deterministic:
  - `timeline.occurred_day` and `timeline.recorded_day` sort by bucket value descending, with null buckets last
  - `timeline.capture_state` sorts `rough`, `enriched`, `reviewed`, `superseded`
  - `timeline.has_evidence` and `timeline.has_unresolved_mentions` sort `true` then `false`
  - the current row sort applies unchanged within each group
- The outline affordance for grouped timeline sheets is limited to one derived group-header level with these operations: `expand group`, `collapse group`, `expand all`, `collapse all`, and `ungroup` via `Group: None`.
- Group headers are derived UI rows only. They MUST NOT have a `record_id`, MUST NOT accept inline edits or paste targets, MUST NOT appear in exports or revision history, and MUST NOT become mutation targets.
- Sorting and filtering apply to underlying rows first; grouping is computed second. Edits, conflicts, autosave, and rollback remain row-based and target only underlying records by `record_id` and `base_row_version`.
- A row MAY move between visible groups only when an edit changes the grouped field value. Dragging a row between groups MUST NOT be a write path.
- Transient expand/collapse state SHOULD remain client-local and MUST NOT be broadcast as collaborative state. Saved views MAY persist the default grouping key, but not another user’s live open/closed state.
- Timeline grouping non-goals:
  - manual row-range grouping or ungrouping
  - nested outline depth greater than `1`
  - subtotal, summary, or spacer rows inserted into the grid
  - pivot-style aggregation or chart-like rollups inside the timeline sheet
  - grouping by formulas, ad hoc expressions, or visible labels
  - merged cells, indent-based hierarchy, or parent/child tree rows
- Views are saveable and shareable within the incident.

Non-normative request example with one user sort override and omitted grouping:

```json
{
  "sort": [
    { "field_key": "timeline.capture_state", "direction": "desc" }
  ],
  "filters": [],
  "limit": 100
}
```

Non-normative response fragment showing the effective applied sort in `meta.query`:

```json
{
  "meta": {
    "query": {
      "sort": [
        { "field_key": "timeline.capture_state", "direction": "desc" },
        { "field_key": "timeline.sort_ts", "direction": "asc" },
        { "field_key": "record_id", "direction": "asc" }
      ],
      "filters": []
    }
  }
}
```

Column-header note: clicking the visible `Time` header emits the stable sort key `timeline.sort_ts`, not `timeline.occurred_at`.

#### Quick-add patterns

- Blank trailing row.
- Keyboard shortcut for new row.
- Paste image from clipboard onto selected row to create or attach evidence.
- Typing into Hosts/Identities cells creates unresolved mentions if nothing matches.

#### Creating and surfacing links

Linked entities surface as chips in cells:

- **resolved canonical link**: plain chip; auto-resolved links add an inspectable auto-match marker
- **unresolved mention**: dotted/outlined chip with raw text
- **ambiguous**: warning badge on chip

When an inline edit or interactive paste auto-resolves a token, the sheet shows a same-surface non-modal disclosure with `Undo` and `Review`.

That lets the grid display relational state without making the user think about join tables.

#### Evidence access without breaking flow

The Evidence column shows a count and preview affordance. Clicking or pressing Space opens a bottom or side preview, not a separate page. Screenshot attachment is drag/drop or clipboard-paste onto the current row.

#### Authorship and version history with low friction

- Row `Edited` column shows last editor and relative time.
- Cell hover can show “last changed by B at 10:14”.
- Full history lives in the inspector, one keypress away.

#### Multi-user presence

Presence is ambient:

- sheet-level avatars in header,
- row-level badge in gutter,
- same-cell indicator when relevant.

No locking banners for normal work.

#### What should feel like Excel vs intentionally differ

**Should feel like Excel:**

- tabular grid
- direct typing
- paste
- fill-down
- keyboard navigation
- flexible sorting/filtering

**Should intentionally differ:**

- relationship cells render chips, not raw comma-separated strings forever
- evidence is attached objects, not file paths in cells
- history is built-in
- formulas/macros/merged cells are not part of the model

#### How denormalized timeline views are composed

The timeline sheet reads from `timeline_grid_projection`. The grid does not query raw joins on every paint.

| Timeline column | Read from projection                           | Write-back behavior                                                                                                                                                                                      |
| --------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Time            | `occurred_at`                                  | update `timeline_events.occurred_at`                                                                                                                                                                     |
| Summary         | `summary`                                      | update `timeline_events.summary/details`                                                                                                                                                                 |
| Hosts           | `host_labels + unresolved_host_tokens`         | interactive unique exact normalized alias match → insert resolved `entity_mentions` + create `record_links` (`provenance='auto_match'`, `confidence=100`); otherwise insert unresolved `entity_mentions` |
| Identities      | `identity_labels + unresolved_identity_tokens` | interactive unique exact normalized alias match → insert resolved `entity_mentions` + create `record_links` (`provenance='auto_match'`, `confidence=100`); otherwise insert unresolved `entity_mentions` |
| Evidence        | `evidence_count`                               | create `object_blob` + `evidence_record` + `record_link`                                                                                                                                                 |
| Tags            | `tag_names`                                    | upsert `tags` + `record_tags`                                                                                                                                                                            |

That is the critical design mechanism: **denormalized reads, intent-aware writes**. The same rule applies to every system view and export surface: reads may be denormalized, but write-back and derivation semantics come from explicit contracts, not visible labels.

Each visible grid row must stay bound to `record_id` and `row_version` from the projection even when the user sorts, filters, or groups the sheet. The visible row number is presentation only; it is never a mutation target.

### UI concept 2: Entity/evidence workbook view

```text
+------------------------------------------------------------------------------------------------------------------+
| Incident IR-2026-017 | Timeline | Hosts* | Identities | Evidence | Notes                                       |
+------------------------------------------------------------------------------------------------------------------+
| View: [All hosts v]  Filters: [State: stub] [Linked events > 0] [Has unresolved aliases]                       |
+----+------------------------+-------------------------+------------+---------------+----------+----------------+
| #  | Host                   | Aliases                 | State      | Linked Events | Evidence | Last Updated   |
+----+------------------------+-------------------------+------------+---------------+----------+----------------+
| 14 | WS-023.corp.example    | WS-023 ; ws023         | canonical  | 7             | 3        | B 2m           |
| 15 | WS-023?                | observed from row 81   | stub       | 1             | 0        | A 15m          |
+----+------------------------+-------------------------+------------+---------------+----------+----------------+
| Split toggle: [Hosts] [Identities] [Evidence]                                                             [>]   |
+------------------------------------------------------------------------------------------------------------------+
```

#### Screen regions and tab model

This is still workbook-shaped. The “Hosts”, “Identities”, and “Evidence” tabs are peer sheets, each backed by its own projection table.

- Hosts sheet → `host_grid_projection`
- Identities sheet → `identity_grid_projection`
- Evidence sheet → `evidence_grid_projection`
- Notes sheet → `artifact_grid_projection WHERE artifact_type='note'`
- Indicators view → `indicator_grid_projection` over canonical indicator records, with pivots to source-bound observations and lifecycle history
- Assessments, ATT&CK, or VERIS views → contract-backed system views keyed by `view_schema_id`, reusing these projections or dedicated overlay projections as needed

#### Inline editing behavior

- Canonical fields like `display_name`, `hostname`, `upn`, `title` are inline-editable.
- Alias cells behave like chip editors: type to add alias, Backspace to remove alias.
- Relationship-derived columns such as `Linked Events` are read-only and clickable.

#### Keyboard, paste, and bulk editing

- Paste a column of hostnames directly into Hosts.
- Pasting into the aliases column creates alias rows.
- Multi-row state changes (`stub -> canonical`) can be applied to selection.
- Bulk merge is **not** a grid action in MVP; it belongs in the inspector because it is destructive.

#### Sorting/filtering/grouping

- Sort by linked event count, last updated, or state.
- Filter to stub records needing cleanup.
- Saved views like “Unresolved hosts” or “High-value identities” matter more here than arbitrary sheets.

#### Quick-add patterns

- Create stub host/identity directly from pasted names.
- Convert unresolved mentions into a selected host/identity from within the inspector without leaving the current grid context.
- Evidence sheet allows drag/drop upload directly into the sheet as well as attachment from a row.

#### Links and evidence surfacing

Clicking `Linked Events` filters the Timeline sheet to the related rows rather than taking the user to a separate module. Evidence counts and previews are available inline.

#### Authorship, history, and presence

Same model as Timeline: last editor column, history in inspector, row presence in gutter.

#### Excel-like vs deliberate differences

This should feel like a workbook tab with sortable rows. It should **not** feel like a CMDB or identity management tool. The deliberate difference from Excel is that a host row is a canonical record with aliases and links, not just a text string on a tab.

#### How denormalized entity/evidence views are composed

`host_grid_projection` can aggregate:

- canonical host fields from `hosts`
- aliases from `entity_aliases`
- linked event counts from `record_links`
- evidence counts from `record_links` to evidence records
- tag names and last editor from `records` / `record_tags`

The grid remains denormalized; writes still go back to source tables. Type chips, icons, and evidence labels should resolve through registry keys such as `host_type_key` and `evidence_type_key`, not hard-coded display strings.

### UI concept 3: Detail / relationship inspector

```text
+-------------------------------- Inspector: Timeline row #81 --------------------------------+
| Summary                                                                 [History] [Links]    |
| Possible VPN logon by jdoe? on WS-023?                                                  A   |
|---------------------------------------------------------------------------------------------|
| Tabs: [Details] [Relationships] [Evidence] [History]                                       |
|                                                                                             |
| Relationships                                                                               |
|   Hosts                                                                                     |
|   - WS-023?                    [Resolve] [Create host] [Dismiss]                            |
|   - WS-032.corp.example        linked by B 2m ago                                           |
|                                                                                             |
|   Identities                                                                                |
|   - jdoe?                      [Resolve] [Create identity]                                  |
|   - john.doe@corp.example      linked by B 2m ago                                           |
|                                                                                             |
| Evidence                                                                                    |
|   [signin.png thumbnail]  screenshot  184 KB  uploaded by A 15m ago                         |
|   [Open preview] [Download]                                                                 |
|                                                                                             |
| History                                                                                     |
|   Rev 5  Reviewer 10:22  Rolled back host link WS-032 -> unresolved                         |
|   Rev 4  B        10:18  Linked WS-032.corp.example                                         |
|   Rev 3  B        10:17  Linked john.doe@corp.example                                       |
|   Rev 2  A        10:03  Attached signin.png                                                |
|   Rev 1  A        10:02  Created event                                                      |
+---------------------------------------------------------------------------------------------+
```

#### Screen regions

- Header with current record identity and quick tabs.
- Body tabs for details, relationships, evidence, history.
- Actions stay in-panel; the main grid remains visible.
- `[Open preview]` issues a fresh preview handle and stays in-panel. `[Download]` issues a fresh single-use download handle. If preview is blocked or unsupported, the inspector remains in place and shows an inline explanation plus `[Download]` when allowed.

#### Inline editing and linking

The inspector is where deeper structure happens:

- resolve, dismiss, and restore mentions,
- create stub or canonical host or identity,
- link a raw source value or text span to an existing indicator or create a canonical indicator,
- inspect or edit indicator lifecycle windows,
- add notes or artifacts,
- inspect linked evidence,
- run rollback.

This is enrichment, not primary capture.

#### Keyboard-first behavior

- `Ctrl+K` opens relationship resolution anchored to the current chip.
- `Esc` closes the inspector and returns focus to the previous cell.
- Arrow navigation in the grid updates the inspector contents live if pinned.

#### Copy/paste and bulk actions

The inspector is not the main paste target, but it should support copying hashes, filenames, aliases, and structured details. Bulk resolution actions can be launched from selected rows but executed here.

#### How links are created and surfaced

The inspector shows both:

- **raw mention lineage** (“A typed `WS-023?` at row creation”),
- **dismissed mentions** in a secondary inspector section or toggle with `Restore to unresolved`, while remaining excluded from the active relationship list,
- **raw indicator observation lineage** when a source value or span has been linked to an indicator, and
- **current canonical links**.

Active relationship lists and default unresolved queues exclude dismissed mentions, but dismissal must preserve inspectable lineage rather than erase it. That distinction is important. It prevents later cleanup from erasing what was actually observed during the incident.

#### Authorship, version history, rollback

The history tab is the reviewer’s primary tool. It shows:

- actor,
- timestamp,
- operation,
- diff summary expanded to changed field/link/mention/tag/evidence-entry units,
- rollback actions for a single logical history entry, whole-row restore, and whole-change-set rollback.

Arbitrary user-selected subsets of fields from historical snapshots are not required in MVP. Rollback should create a new revision; history remains intact.

#### Multi-user presence and modal avoidance

If another analyst is editing the same record, the inspector shows that inline, but does not lock them out. This panel should be a drawer, not a blocking modal.

#### Why this does not become a rigid case-management app

Because the inspector is optional for common work. Analysts can live in the grid for most of the incident and only open the inspector when they need structure, history, or relationship cleanup.

#### Why this does not become an uncontrolled spreadsheet clone

Because structure lives underneath the sheet:

- mentions are first-class,
- links are typed,
- evidence is attached objects,
- history is immutable,
- tabs are views over source records, not independent data islands.
