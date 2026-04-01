# Cartulary Progressive Implementation and Testing Guide

**Status**: Derived implementation-planning artifact  
**Authoritative sources**: Core 00–04  
**Traceability source**: Appendix F  
**Scope**: Base profile, with extension-profile testing hooks in Phase 10

---

## 1. How to read this guide

This guide is a **derived implementation-planning artifact**, not an independent source of contract truth. Core 00 through Core 04 are authoritative. Appendix F is the traceability aid. When this guide and an owner section diverge, the owner section governs and this guide must be repaired.[^precedence]

This guide preserves the ten-phase implementation shape because it is useful for delivery planning, but the phase order is an **implementation aid**, not a parallel specification. A phase may group work for sequencing convenience. It does not change requirement ownership, route semantics, data-model rules, or conformance scope.[^precedence][^traceability]

Phase headers therefore name **primary owner sections**, not broad REQ blocks. The phase tables carry the exact REQ and AC identifiers for each planned test. Use the phase header to find the owner section. Use the row-level mappings to build the test and to prove traceability.[^precedence][^traceability]

This guide also resolves one dependency error from the prior version: reviewer-facing history, delete or restore, and rollback remain late-phase work, but the **minimal write-side mutation substrate** cannot wait until that phase. As soon as the first record-row mutation path exists, it must already emit attributed mutations, maintain projections, and satisfy normalized idempotency and optimistic-concurrency contracts. Phase 7 now completes the reviewer-facing history and rollback surface instead of introducing mutation history for the first time.[^traceability]

### 1.1 Test categories used in this guide

- **Unit test**: verifies one module, validation rule, serializer, reducer, mapper, or deterministic algorithm in isolation.
- **Integration test**: verifies one or more modules against real backing services or real transport boundaries such as PostgreSQL, object storage, or the WebSocket boundary.
- **E2E test**: verifies a user-observable flow through the full deployed stack against the visible contract and the cited ACs.

### 1.2 Conformance posture

Phases 0 through 9 are the base-profile implementation sequence. A base-profile claim must satisfy the current Base claim manifest in Core 04, including `AC-299`. Phase 10 is intentionally not a base-profile phase. It is an extension-profile hook section that keeps extension work aligned to current route families and profile claim boundaries.[^base-manifest][^traceability]

---

## 2. Phase 0 — Infrastructure, deployment configuration, and schema bootstrap

### 2.0.1 Scope

This phase establishes the deployable shell and startup control plane:

- modular-monolith application shell,
- required services and runtime roots,
- deployment-configuration artifact discovery and validation,
- canonical disconnected-layout defaults,
- fail-closed startup,
- schema bootstrap idempotency,
- object-store reachability.

No domain routes beyond health or startup diagnostics should be treated as complete at the end of this phase.

### 2.0.2 Primary owner sections

- Core 01 §1 Architecture pattern
- Core 04 §5–§8 deployment topology, runtime roots, container boundary, and required services
- Core 04 §12 deployment-configuration contract
- Core 04 §9.0.1 base-claim manifest for `AC-294..AC-298`

### 2.0.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-0-01 | Configuration discovery follows the canonical precedence chain and rejects unknown or mismatched `config_schema_id` / `deployment_profile` combinations before startup. | REQ-04-066..REQ-04-071 | AC-294, AC-298 |
| U-0-02 | The runtime-root registry requires exactly the configured root keys for the active profile and rejects missing or malformed bindings. | REQ-04-058, REQ-04-069, REQ-04-071..REQ-04-073, REQ-04-077 | AC-295 |
| U-0-03 | Filesystem-root path validation canonicalizes paths and rejects traversal, root escape, and invalid overlap cases. | REQ-04-059, REQ-04-074..REQ-04-075, REQ-04-077 | AC-296 |
| U-0-04 | Disconnected-layout defaults are applied only where the owner section allows them. Missing required roots are not silently defaulted away. | REQ-04-067, REQ-04-069, REQ-04-071..REQ-04-076 | AC-297 |
| U-0-05 | Invalid deployment configuration blocks HTTP startup, WebSocket startup, and background-job startup. | REQ-04-077..REQ-04-078 | AC-298 |
| U-0-06 | Schema bootstrap is idempotent. Running the bootstrap sequence twice does not create duplicate schema objects or fail the second run. | REQ-01-001..REQ-01-003, REQ-04-061 | AC-231 |

### 2.0.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-0-01 | Against a real PostgreSQL instance, bootstrap creates the required extensions and base schema objects and can be rerun without drift. | REQ-01-001..REQ-01-003, REQ-04-061 | AC-231 |
| I-0-02 | Against a real S3-compatible object store, the application can initialize the configured storage binding and perform a minimal round-trip object write and read. | REQ-04-058..REQ-04-061 | AC-231 |
| I-0-03 | A configuration with a path-validation failure or missing required root never reaches ready state, even when PostgreSQL and object storage are otherwise healthy. | REQ-04-074..REQ-04-078 | AC-296, AC-298 |

### 2.0.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-0-01 | A fresh minimum deployment starts, passes health checks, and exposes a ready state only after PostgreSQL, object storage, and configuration validation all succeed. | REQ-04-054..REQ-04-061, REQ-04-066..REQ-04-078 | AC-294..AC-298 |
| E-0-02 | A deployment with an invalid config artifact never exposes a healthy application surface and exits with structured startup diagnostics. | REQ-04-077..REQ-04-078 | AC-298 |

---

## 3. Phase 1 — Authentication, sessions, and deployment-local user administration

### 3.1.1 Scope

This phase establishes the authenticated shell:

- local account login,
- Argon2id password hashing,
- TOTP MFA,
- CSRF protection,
- session lifecycle and concurrency cap,
- session inspection and logout,
- deployment-local user creation and patch,
- immediate socket revocation behavior for revoked sessions.

Incident-specific work does not begin until the session and deployment-local account boundary is trustworthy.

### 3.1.2 Primary owner sections

- Core 01 §3.3.2 session and authentication routes
- Core 01 §3.3.5.1 deployment-local user administration
- Core 01 §3.3.10 WebSocket session lifecycle consequences
- Core 04 §1 authentication model and session lifecycle boundaries
- Core 04 §3 attribution and administrative audit requirements

### 3.1.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-1-01 | `POST /api/v1/auth/login` rejects malformed top-level members, malformed `second_factor`, and unsupported base-profile MFA kinds with `invalid_auth_request`. | REQ-01-025, REQ-01-234 | AC-244, AC-247, AC-249 |
| U-1-02 | `username` is trimmed for login matching, while `password` is compared exactly as supplied after JSON decoding. | REQ-01-025 | AC-244, AC-245, AC-246 |
| U-1-03 | Successful login creates one server-side session record with `authenticated_at`, `last_qualifying_activity_at`, `idle_expires_at`, `absolute_expires_at`, and `session_expires_at=min(idle, absolute)`. | REQ-01-026..REQ-01-028, REQ-04-005..REQ-04-011 | AC-123, AC-131, AC-156..AC-163 |
| U-1-04 | `GET /api/v1/auth/session` returns the singleton session resource, rejects pagination members, and does not extend idle expiry by itself. | REQ-01-027..REQ-01-028 | AC-123, AC-130 |
| U-1-05 | A sixth concurrent session revokes the least-recently-used non-current session and records `concurrency_limit`. | REQ-04-013..REQ-04-014 | AC-131, AC-136, AC-163 |
| U-1-06 | `POST /api/v1/auth/logout` revokes only the current session. Password change, MFA reset, account disablement, or explicit revoke-all revokes every active session for that user. | REQ-01-029, REQ-04-015..REQ-04-016 | AC-131, AC-136, AC-156..AC-163 |
| U-1-07 | `POST /api/v1/users` applies the required defaults: omitted `mfa_required=true`, omitted `is_deployment_admin=false`, server-managed `is_active=true`, and no `initial_password` echo. | REQ-01-117..REQ-01-123 | AC-175..AC-177 |
| U-1-08 | `PATCH /api/v1/users/{user_id}` enforces `base_user_version` and rejects demotion or deactivation of the last active deployment admin. | REQ-01-124..REQ-01-126 | AC-178..AC-180 |
| U-1-09 | Cookie-authenticated state-changing routes fail closed on missing or invalid CSRF proof. | REQ-04-003 | AC-123, AC-130 |

### 3.1.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-1-01 | Login, session inspection, idle sliding, and logout all persist against a real PostgreSQL session store. | REQ-01-023..REQ-01-031, REQ-04-005..REQ-04-017 | AC-123, AC-131, AC-156..AC-163 |
| I-1-02 | Session revocation pushes `session_revoked` to an attached socket owned by that session and then closes the connection. | REQ-01-029, REQ-01-250..REQ-01-277, REQ-04-015..REQ-04-016 | AC-131, AC-136, AC-156..AC-163 |
| I-1-03 | Deployment-admin user creation and patch persist the expected defaults, version changes, and administrative audit records. | REQ-01-117..REQ-01-126, REQ-04-038 | AC-175..AC-180, AC-231 |

### 3.1.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-1-01 | A local user logs in, retrieves the session resource, and sees `memberships[]`, expiry fields, and `provider_type='local'`. | REQ-01-023..REQ-01-031, REQ-04-001..REQ-04-017 | AC-123, AC-130 |
| E-1-02 | An MFA-required account receives `mfa_required` when `second_factor` is omitted, succeeds with a valid TOTP code, and fails with `invalid_second_factor` for a wrong but structurally valid code. | REQ-01-025, REQ-01-234, REQ-04-001 | AC-246, AC-248, AC-249 |
| E-1-03 | Invalid credentials return `invalid_credentials` without creating a session cookie. | REQ-01-025, REQ-01-234 | AC-245 |
| E-1-04 | Session idle expiry causes subsequent authenticated requests to fail closed until the user logs in again. | REQ-04-008..REQ-04-012 | AC-131, AC-136 |
| E-1-05 | A deployment admin creates a user, patches it, and sees optimistic-concurrency enforcement and the last-admin guard. | REQ-01-117..REQ-01-126 | AC-175..AC-180 |

---

## 4. Phase 2 — Incidents, memberships, and the incident-scoped control envelope

### 4.2.1 Scope

This phase establishes incident-scoped administration and the first stable incident-level API behavior:

- incident create, list, get, and patch,
- creator bootstrap membership,
- incident membership create, patch, and delete,
- role-gated incident authorization,
- common response-envelope expectations on first incident routes,
- incident create and patch idempotency or optimistic versioning,
- automatic creation of workbook preference objects at incident create.

This phase does **not** yet complete record-row history for workbook records. It does establish the expectation that every new mutating route from this point forward must already satisfy its route-owned validation, idempotency, versioning, authorization, and audit contracts.

### 4.2.2 Primary owner sections

- Core 01 §3.3.1 versioning and compatibility
- Core 01 §3.3.3 route families
- Core 01 §3.3.5.1 incident membership routes
- Core 01 §3.3.5.2 saved-view and workbook-preference routes
- Core 01 §3.3.5.3 incident create, list, get, and patch
- Core 02 §4.5 current-profile incident promoted fields
- Core 04 §2 authorization model
- Core 04 §3 attribution and administrative audit requirements

### 4.2.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-2-01 | `POST /api/v1/incidents` accepts only the declared top-level members, rejects `initial_memberships`, and normalizes `incident_key` using trim + NFC before uniqueness checks. | REQ-01-154..REQ-01-167, REQ-02-015..REQ-02-016 | AC-170..AC-174, AC-211..AC-214, AC-219..AC-220 |
| U-2-02 | Incident create bootstraps the creator as incident `admin` and creates both incident-wide and per-user workbook-preference rows. | REQ-01-156..REQ-01-159, REQ-01-145..REQ-01-151 | AC-170..AC-174, AC-150, AC-153 |
| U-2-03 | Successful incident create returns a stable `Location` header rooted at `/api/v1/incidents/{incident_id}`. | REQ-01-160 | AC-170..AC-174 |
| U-2-04 | Incident create idempotency is keyed by `(actor_user_id, client_txn_id)`. Replay of the same normalized request returns the original result. Divergent replay returns `client_txn_conflict`. | REQ-01-161..REQ-01-167 | AC-170..AC-174, AC-219..AC-220 |
| U-2-05 | Incident patch accepts only `tlp`, `current_phase`, and `primary_external_case_ref`, requires `base_incident_version`, and treats a structurally valid no-op as version-stable. | REQ-01-168..REQ-01-180, REQ-02-015 | AC-170..AC-174, AC-211..AC-214 |
| U-2-06 | Membership create requires exactly one of `user_id` or `email`, uses the closed role vocabulary, and never auto-creates a user or invitation. | REQ-01-127..REQ-01-132 | AC-175..AC-180 |
| U-2-07 | Membership patch and delete enforce `base_membership_version` and the `last_incident_admin` guard. | REQ-01-133..REQ-01-137 | AC-178..AC-180 |
| U-2-08 | `deployment_admin` alone does not grant incident read, incident write, incident job, preview, download, or WebSocket access. | REQ-04-028..REQ-04-030 | AC-178..AC-180, AC-261 |

### 4.2.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-2-01 | Incident create persists the incident row, creator bootstrap membership, and both workbook-preference objects atomically. | REQ-01-154..REQ-01-160, REQ-01-145..REQ-01-151 | AC-170..AC-174, AC-150, AC-153 |
| I-2-02 | Duplicate `incident_key` on a distinct create request fails with `incident_key_conflict` after normalization. | REQ-01-165, REQ-02-016 | AC-170..AC-174, AC-211..AC-214 |
| I-2-03 | Membership changes re-derive incident authorization immediately for subsequent reads and mutating routes. | REQ-01-127..REQ-01-137, REQ-04-021..REQ-04-030 | AC-175..AC-180 |
| I-2-04 | Incident patch persists only the allowed promoted fields and advances `incident_version` only on material change. | REQ-01-168..REQ-01-180, REQ-02-015 | AC-170..AC-174, AC-211..AC-214 |

### 4.2.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-2-01 | An authenticated user creates an incident, is bootstrapped as incident `admin`, and lands on a valid workbook surface for that incident. | REQ-01-154..REQ-01-160, REQ-04-021..REQ-04-026 | AC-170..AC-174 |
| E-2-02 | The same incident appears in incident discovery, can be retrieved directly, and can be patched only through the allowed promoted fields. | REQ-01-168..REQ-01-180, REQ-02-015 | AC-170..AC-174, AC-211..AC-214 |
| E-2-03 | An incident admin adds, changes, and removes memberships. A non-admin incident member cannot perform the same actions in place. | REQ-01-127..REQ-01-137, REQ-04-021..REQ-04-030 | AC-175..AC-180 |
| E-2-04 | Unknown or forbidden top-level members on incident create or patch are rejected with the route-owned error contract. | REQ-01-021, REQ-01-154..REQ-01-180 | AC-219, AC-220 |

---

## 5. Phase 3 — Timeline, grid hot path, and first record-row mutation substrate

### 5.3.1 Scope

This phase introduces the first high-volume workbook record path and therefore the first complete record-row mutation substrate:

- Timeline row create and patch,
- partial and uncertain capture,
- autosave and exact save-state labels,
- first record-row `change_set` / revision emission,
- transactionally maintained timeline projection rows,
- explicit `mark-reviewed` and `supersede` actions,
- current Timeline `capture_state` machine: `rough`, `enriched`, `reviewed`, `superseded`.

This is the first phase where record-row history exists at all. Phase 7 will later complete the reviewer-facing history, delete or restore, and rollback surface.

### 5.3.2 Primary owner sections

- Core 01 §3.3.5 mutation contract and Timeline review actions
- Core 01 §7.4 Timeline view contract
- Core 01 §8 projection model
- Core 02 §5 partial and uncertain data
- Core 02 §14 history substrate minima
- Core 03 §1 interaction model
- Core 03 §4 save-state and autosave behavior
- Core 03 §6 Timeline lifecycle
- Core 03 §7 Timeline create workflow
- Core 03 §15 Timeline read and write contract

### 5.3.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-3-01 | A Timeline row can be created with one non-empty user-entered value. System-managed fields, `record_id`, and `row_version` are assigned after commit. | REQ-01-057, REQ-03-111..REQ-03-115 | AC-001, AC-002, AC-125 |
| U-3-02 | Initial Timeline create sets `capture_state='rough'`. No obsolete state tokens such as `developing` or `complete` are accepted. | REQ-03-102, REQ-03-236..REQ-03-241 | AC-107..AC-111, AC-191 |
| U-3-03 | The first capture-state-material mutation transitions `rough -> enriched`. A reviewer or admin can explicitly mark `rough` or `enriched` as `reviewed`. | REQ-03-103..REQ-03-104, REQ-01-083..REQ-01-085 | AC-107..AC-111, AC-194..AC-197 |
| U-3-04 | A later material edit on a reviewed Timeline row demotes it to `enriched`. An explicit supersede action moves a legal row to `superseded` and blocks ordinary forward editing semantics. | REQ-03-105..REQ-03-110, REQ-01-086..REQ-01-088 | AC-107..AC-111, AC-198..AC-199 |
| U-3-05 | Autosave commits on Enter, Tab, blur, and paste completion. No explicit Save button is required. Save-state labels are exactly `Syncing`, `Saved`, and `Conflict`. | REQ-03-087..REQ-03-089 | AC-043 |
| U-3-06 | `PATCH /api/v1/records/{record_id}` requires `view_schema_id`, `base_row_version`, `client_txn_id`, and non-empty `changes[]`. Duplicate `field_key` entries or `changes[]: []` are rejected as malformed mutation payloads. | REQ-01-058..REQ-01-060, REQ-01-069..REQ-01-070 | AC-125, AC-126, AC-299 |
| U-3-07 | Patch-route idempotent replay returns the original committed result and creates no second mutation, no second revision, and no second collaboration event. | REQ-01-058, REQ-01-069..REQ-01-070 | AC-126, AC-299 |
| U-3-08 | Timeline projection rows carry the view-owned derived fields and maintain stable `record_id` / `row_version` binding for the grid. | REQ-01-312..REQ-01-322, REQ-01-349..REQ-01-350, REQ-03-236..REQ-03-241 | AC-116, AC-119, AC-120, AC-191..AC-193 |
| U-3-09 | Every successful Timeline create or patch writes an attributed mutation entry and a new row revision for that record. | REQ-02-205..REQ-02-207, REQ-04-036..REQ-04-037 | AC-215, AC-231 |

### 5.3.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-3-01 | Timeline create or patch writes the source row, mutation history rows, and projection row atomically in one transaction. | REQ-01-057..REQ-01-070, REQ-01-351..REQ-01-353, REQ-02-205..REQ-02-207 | AC-125, AC-210, AC-215, AC-299 |
| I-3-02 | The Timeline query route reads from projection rows and returns stable row identity and deterministic ordering without scanning source blobs. | REQ-01-034..REQ-01-037, REQ-01-355..REQ-01-366, REQ-03-236..REQ-03-241 | AC-124, AC-184, AC-191..AC-193 |
| I-3-03 | Review and supersede actions persist the correct lifecycle transitions and reject illegal transitions. | REQ-01-083..REQ-01-088, REQ-03-104..REQ-03-110 | AC-107..AC-111, AC-194..AC-199 |

### 5.3.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-3-01 | An analyst creates a Timeline row with one non-empty value and immediately continues editing in-grid without leaving the workbook surface. | REQ-03-001..REQ-03-003, REQ-03-111..REQ-03-115 | AC-001, AC-002 |
| E-3-02 | Typing acknowledgement and visible save-state transitions stay inside the declared hot-path interaction envelope on the reference fixture. | REQ-01-015..REQ-01-017, REQ-03-087..REQ-03-089, REQ-03-217..REQ-03-219 | AC-043 |
| E-3-03 | A reviewer marks a row as reviewed, later edits demote it to enriched when material, and a legal supersede action moves it to `superseded`. | REQ-01-083..REQ-01-088, REQ-03-103..REQ-03-110 | AC-107..AC-111 |
| E-3-04 | Replaying the same patch request with the same `client_txn_id` returns the original committed result and does not create duplicate history or duplicate visible updates. | REQ-01-058, REQ-01-069..REQ-01-070 | AC-299 |

---

## 6. Phase 4 — Entities, mentions, resolution, merge, and canonical-indicator foundations

### 6.4.1 Scope

This phase introduces progressive normalization and exact-match entity behavior:

- `mention_origin` vs `entity_origin`,
- entity-mention lifecycle (`unresolved`, `resolved`, `dismissed`),
- explicit resolve, dismiss, and ordinary restore,
- explicit create-from-mention,
- exact-match reuse precedence,
- no auto-merge of pre-existing entities,
- explicit merge,
- source-bound indicator observations vs canonical indicators.

### 6.4.2 Primary owner sections

- Core 01 §3.3.5 mention action route and merge route
- Core 02 §6 binding-mode contract
- Core 02 §7 provenance requirements
- Core 02 §8 deduplication and exact-match reuse
- Core 02 §9 merge behavior
- Core 02 §10 indicator model foundations
- Core 03 §9 resolution workflows and auto-resolution boundaries
- Core 03 §16 inspector and entity/evidence interaction

### 6.4.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-4-01 | A `mention_origin` write creates `entity_mention` rows and never implicitly creates a host or identity record. An `entity_origin` write creates or upserts an entity row and does not synthesize mentions. | REQ-02-028..REQ-02-036 | AC-019, AC-020, AC-022 |
| U-4-02 | Repeated identical mention text values remain separate mention rows with distinct source provenance. | REQ-02-031..REQ-02-032, REQ-02-058 | AC-019, AC-021 |
| U-4-03 | Explicit `Create host` or `Create identity` from a mention creates a stub only when no unique exact-match entity exists, keeps the raw mention, and resolves only the selected mention by default. | REQ-02-034, REQ-02-038, REQ-02-054..REQ-02-055 | AC-020, AC-021, AC-186 |
| U-4-04 | Dismissing a mention preserves the mention row, clears active resolution metadata, and excludes it from active relationship-cell values. Ordinary restore returns it to `unresolved`, not to a historical resolved target. | REQ-02-039..REQ-02-041 | AC-188..AC-190, AC-224, AC-225 |
| U-4-05 | Exact-match reuse follows the stable precedence rules for hosts and identities. Alias and fuzzy matches remain suggestions only and never auto-resolve or auto-merge. | REQ-02-059..REQ-02-063 | AC-021, AC-022 |
| U-4-06 | Entity merge is explicit only. The survivor `record_id` remains unchanged, the loser remains historical with merge lineage, and raw source mentions are not rewritten. | REQ-02-064..REQ-02-066 | AC-023, AC-186, AC-209 |
| U-4-07 | Indicator observations remain source-bound rows. Canonical indicators use incident-scoped dedupe identity and lifecycle state that is separate from observations. | REQ-02-027, REQ-02-056..REQ-02-057, REQ-02-072..REQ-02-082 | AC-017, AC-077..AC-079 |

### 6.4.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-4-01 | `POST /api/v1/entity-mentions/{entity_mention_id}/resolve` updates resolution state, provenance, and active links for `resolve_item`, `dismiss_item`, and `revert_to_unresolved` using the route-owned concurrency and idempotency contract. | REQ-01-196..REQ-01-227, REQ-02-039..REQ-02-044 | AC-188..AC-190, AC-221..AC-225 |
| I-4-02 | Direct create or paste on Hosts or Identities surfaces upserts on a unique exact match and otherwise creates a stub with preserved aliases and structured provenance. | REQ-02-035..REQ-02-036, REQ-02-054..REQ-02-055, REQ-02-059..REQ-02-063 | AC-022, AC-186 |
| I-4-03 | Entity merge repoints live resolutions and live links to the survivor while preserving loser lineage and history. | REQ-01-181..REQ-01-195, REQ-02-064..REQ-02-066 | AC-023, AC-186, AC-209 |

### 6.4.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-4-01 | An analyst types raw host and identity text on a Timeline row, resolves one token to an existing entity, and creates a stub from another token in the inspector without leaving the workbook surface. | REQ-02-030..REQ-02-034, REQ-03-129..REQ-03-134, REQ-03-247..REQ-03-249 | AC-006, AC-019, AC-020 |
| E-4-02 | An analyst dismisses and later ordinarily restores a mention. The restored mention returns to the unresolved queue and does not silently recover an old resolved target. | REQ-02-039..REQ-02-041, REQ-03-129..REQ-03-134 | AC-188..AC-190, AC-224, AC-225 |
| E-4-03 | A reviewer merges two duplicate entities from the inspector. The surviving row identity remains stable and dependent links or resolutions follow the survivor. | REQ-01-181..REQ-01-195, REQ-02-064..REQ-02-066, REQ-03-247..REQ-03-249 | AC-023, AC-186, AC-209 |

---

## 7. Phase 5 — Evidence, blob lifecycle, evidence access, and object storage

### 7.5.1 Scope

This phase completes the binary-evidence path:

- evidence records without blobs,
- blob-slot creation,
- accepted upload contract echo,
- final attach or replacement on an evidence record,
- evidence lifecycle vs blob lifecycle separation,
- preview-handle and download-handle issuance,
- same-origin handle redemption with current authorization re-check,
- fail-closed preview and download behavior,
- evidence projection updates on workbook surfaces.

### 7.5.2 Primary owner sections

- Core 01 §3.3.8 blob-slot and evidence-attach routes
- Core 01 §16 evidence-access handle contract
- Core 02 §4.5 promoted evidence fields
- Core 02 §13 evidence and blob schema
- Core 03 §8 evidence attachment workflow
- Core 03 §16 evidence sheet and inspector behavior
- Core 04 §4.3 evidence upload trust boundary
- Core 04 §4.5 hostile upload and preview constraints

### 7.5.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-5-01 | `POST /api/v1/object-blobs` requires `incident_id`, `client_txn_id`, and declared `byte_size`, rejects unknown top-level members, and echoes `accepted_contract` on success. | REQ-01-243..REQ-01-247 | AC-128, AC-154, AC-155 |
| U-5-02 | Blob-create idempotency is keyed by `(actor_user_id, incident_id, client_txn_id)`. Replay of the same normalized request returns the original slot. Divergent replay returns route-owned conflict. | REQ-01-243..REQ-01-247 | AC-128, AC-154, AC-155 |
| U-5-03 | `POST /api/v1/evidence-records/{record_id}/attach-blob` requires `object_blob_id`, `base_row_version`, and `client_txn_id` and fails closed when the blob is pending, failed, missing, expired, or contract-mismatched. | REQ-01-245..REQ-01-247 | AC-102, AC-103, AC-128 |
| U-5-04 | Evidence lifecycle state remains separate from blob `upload_state`. Requested or pending-receipt evidence can exist without a blob and later advance without mutating unrelated custody history. | REQ-02-186..REQ-02-201 | AC-102, AC-103, AC-154, AC-155 |
| U-5-05 | Preview-handle and download-handle issuance routes accept only `{}` and are intentionally non-idempotent. Each success yields a fresh opaque same-origin handle. | REQ-01-458..REQ-01-463 | AC-251, AC-252, AC-253 |
| U-5-06 | Handle redemption re-derives current session validity, current incident membership, and current evidence/blob state. Preview blocks explicitly on unsupported or unsafe preview conditions rather than silently falling back. | REQ-01-463..REQ-01-465, REQ-04-023, REQ-04-053 | AC-252..AC-255 |
| U-5-07 | Download responses use authoritative metadata for filename disposition and apply deterministic fallback naming when authoritative names are absent. | REQ-01-459..REQ-01-465 | AC-251, AC-253 |
| U-5-08 | Evidence attachment updates workbook-visible evidence counts and `has_evidence`-style derived flags without forcing navigation away from the current surface. | REQ-03-116..REQ-03-126, REQ-03-242..REQ-03-246 | AC-004, AC-015, AC-016 |

### 7.5.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-5-01 | A real object-store upload can be created through a blob slot, finalized onto an evidence record, and then queried through the workbook projection without duplicate attachment rows. | REQ-01-243..REQ-01-247, REQ-02-186..REQ-02-201, REQ-03-116..REQ-03-126 | AC-015, AC-016, AC-128, AC-154, AC-155 |
| I-5-02 | Expired-slot replay returns the same expired slot. A fresh upload target requires a new `client_txn_id`. | REQ-01-243..REQ-01-247 | AC-128, AC-154, AC-155 |
| I-5-03 | A redeemed preview or download handle fails closed after logout, membership removal, or evidence/blob state invalidation. | REQ-01-463..REQ-01-465, REQ-04-023, REQ-04-053 | AC-252..AC-255 |
| I-5-04 | If the deployment uses an upload-scanning adjunct service, the two-step attach semantics remain intact and preview never bypasses the scanning or quarantine boundary. | REQ-04-048, REQ-04-053 | AC-053 |

### 7.5.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-5-01 | An analyst attaches a screenshot to a selected Timeline row without leaving the workbook surface. The row reflects the new evidence count after commit. | REQ-03-116..REQ-03-126 | AC-004, AC-015, AC-016 |
| E-5-02 | A screenshot-only Timeline row can be persisted through the two-step evidence path. | REQ-03-102, REQ-03-116..REQ-03-126 | AC-002, AC-102, AC-103 |
| E-5-03 | An inline-safe type receives a preview handle and renders through the same-origin redeem path. An unsupported or unsafe type returns an explicit blocked-preview outcome. | REQ-01-458..REQ-01-465, REQ-04-053 | AC-252..AC-255 |
| E-5-04 | Requested evidence can be tracked before the blob exists and later advanced to an available state without breaking workbook pivots or counts. | REQ-02-186..REQ-02-201, REQ-03-242..REQ-03-246 | AC-015, AC-154, AC-155 |

---

## 8. Phase 6 — Collaboration, presence, and same-field conflict resolution

### 8.6.1 Scope

This phase completes the live multi-user path:

- incident-scoped WebSocket route,
- `hello` and `resume`,
- replay window and reset behavior,
- presence snapshots and deltas,
- `record_changed`, `job_progress`, `session_revoked`, and heartbeat behavior,
- field-level optimistic concurrency,
- same-field conflict transport,
- `atomic_replace`, `text_compare_merge`, and `collection_review`,
- same-surface resolver behavior and client-local conflict queue handling.

### 8.6.2 Primary owner sections

- Core 01 §3.3.10 WebSocket public contract
- Core 01 §3.3.5 mutation contract
- Core 03 §3 concurrency and same-field conflict resolution
- Core 03 §4 save-state, presence, pending queue, and local draft boundaries
- Core 04 §1 session lifecycle consequences
- Core 04 §2 authorization re-derivation
- Core 04 §4.5 origin and hostile-content constraints relevant to sockets

### 8.6.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-6-01 | Every grid write includes `record_id`, `base_row_version`, and changed fields only. Different-field concurrent edits auto-rebase. Same-field concurrent edits reject with an explicit conflict payload. | REQ-03-033..REQ-03-040 | AC-009, AC-013, AC-126 |
| U-6-02 | Same-field conflict transport uses `409`, `error.code='same_field_conflict'`, and an `error.conflict` object with the required base and current values. | REQ-03-063..REQ-03-068 | AC-126, AC-203, AC-204, AC-226..AC-230 |
| U-6-03 | `text_compare_merge` treats the field as plain text, normalizes line endings only for merge computation, and never silently auto-commits a clean merge suggestion. | REQ-03-054..REQ-03-061 | AC-226..AC-230 |
| U-6-04 | `collection_review` fields use `collection_value_v1` in read or conflict payloads and `collection_actions_v1` in explicit resolution writes. | REQ-01-062..REQ-01-067, REQ-03-052..REQ-03-053 | AC-118, AC-203, AC-204 |
| U-6-05 | The resolver keeps the grid visible, leaves conflict state unresolved until explicit action, and returns focus to the same cell after resolution or clear. | REQ-03-041..REQ-03-051 | AC-037..AC-042 |
| U-6-06 | `keep_saved` clears the local conflict without creating a new revision. `use_unsaved` and `merged_value` create a new attributed change set. | REQ-03-077..REQ-03-082 | AC-041, AC-163 |
| U-6-07 | The first application message on the incident socket is exactly one of `hello` or `resume`. Resume outside the replay window yields reset behavior rather than partial replay. | REQ-01-250..REQ-01-277 | AC-129, AC-131, AC-135 |
| U-6-08 | Presence payloads are incident-scoped and ephemeral. Heartbeats do not extend idle expiry. `session_revoked` closes the socket with the route-owned reason code set. | REQ-01-250..REQ-01-277, REQ-03-090..REQ-03-100, REQ-04-010 | AC-008, AC-131, AC-132..AC-136, AC-156..AC-163 |

### 8.6.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-6-01 | Two real clients connect to the same incident socket, exchange presence snapshots and deltas, and observe deterministic replay ordering within the replay window. | REQ-01-250..REQ-01-277, REQ-03-090..REQ-03-098 | AC-129, AC-131..AC-135 |
| I-6-02 | Resume with a valid replay token replays replayable messages only. Presence is re-hydrated through the documented presence flow rather than replay. | REQ-01-250..REQ-01-277 | AC-129, AC-132..AC-135 |
| I-6-03 | Concurrent edits to different fields succeed without operator intervention. Concurrent edits to the same field produce the resolver path and preserve both saved and local drafts correctly. | REQ-03-033..REQ-03-082 | AC-009, AC-037..AC-042, AC-203..AC-204, AC-226..AC-230 |
| I-6-04 | Cookie-authenticated browser socket upgrades reject untrusted `Origin` values before incident subscription is granted. | REQ-01-250..REQ-01-277, REQ-04-053 | AC-131, AC-255 |

### 8.6.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-6-01 | Two analysts on the same incident see each other’s presence on the workbook surface within the expected interaction window. | REQ-03-090..REQ-03-098 | AC-008, AC-132, AC-133 |
| E-6-02 | Concurrent edits to different fields on the same row auto-merge. Concurrent edits to the same field open the same-surface resolver and require explicit analyst resolution. | REQ-03-033..REQ-03-082 | AC-009, AC-037..AC-042, AC-226..AC-230 |
| E-6-03 | Logout, expiry, or concurrency-limit revocation emits `session_revoked`, closes the socket, and preserves unsaved local work for later explicit recovery after re-authentication. | REQ-01-029, REQ-01-250..REQ-01-277, REQ-03-099..REQ-03-100, REQ-04-013..REQ-04-016 | AC-131, AC-136, AC-156..AC-163 |
| E-6-04 | Live updates never retarget a pending local edit away from the bound `record_id` during sort, filter, or grouping changes. | REQ-01-015..REQ-01-017, REQ-03-086, REQ-03-223..REQ-03-235 | AC-047 |

---

## 9. Phase 7 — Reviewer-facing history, delete or restore, and rollback

### 9.7.1 Scope

This phase completes the reviewer and destructive-operation surface:

- record-history retrieval,
- history pagination,
- soft-delete and restore,
- history-entry rollback,
- whole-change-set rollback,
- whole-row restore,
- tombstone `row_version` restore preconditions,
- `history_entry_ref`,
- `available_rollback_actions[]`,
- record-lock enforcement for destructive paths.

### 9.7.2 Primary owner sections

- Core 01 §3.3.4.2 record-history read contract
- Core 01 §3.3.5 delete, restore, review actions, and rollback routes
- Core 02 §14–§15 history and mutation-target substrate
- Core 03 §10 reviewer history and rollback workflows
- Core 04 §2 authorization model
- Core 04 §3 attribution and audit requirements

### 9.7.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-7-01 | `GET /api/v1/records/{record_id}/history` returns newest-first items with deterministic order, current tombstone `row_version` for deleted rows, and canonical `available_rollback_actions[]` ordering. | REQ-01-048..REQ-01-056 | AC-184, AC-185, AC-215 |
| U-7-02 | `history_entry_ref` is present only when a logical history item maps to exactly one reversible mutation target and remains opaque and stable across repeated reads. | REQ-01-054..REQ-01-055 | AC-215, AC-216 |
| U-7-03 | Delete requires the current `row_version`, respects role gates, and returns route-owned failures such as `record_deleted_use_restore`, `record_not_deleted`, and `record_locked` where applicable. | REQ-01-071..REQ-01-076, REQ-04-021..REQ-04-024 | AC-215, AC-218 |
| U-7-04 | Restore requires the tombstone `row_version`, respects role gates, and returns the record to active state without mutating prior history rows in place. | REQ-01-077..REQ-01-082 | AC-215, AC-216 |
| U-7-05 | Rollback accepts only the documented selector kinds `history_entry`, `change_set`, and `row_restore`, and creates a new `change_set` with source `rollback` rather than mutating prior history. | REQ-01-089..REQ-01-111 | AC-216, AC-217 |
| U-7-06 | Record locks are enforced for destructive or reviewer-only operations such as merge, rollback, and restore when the owner section requires them. | REQ-01-071..REQ-01-111, REQ-03-101 | AC-182, AC-187, AC-218 |

### 9.7.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-7-01 | Delete, restore, and rollback update source rows, projections, history rows, and emitted collaboration events atomically. | REQ-01-071..REQ-01-111, REQ-01-351..REQ-01-353, REQ-02-205..REQ-02-220 | AC-210, AC-215..AC-218 |
| I-7-02 | History pagination remains bound to `record_id` and preserves deterministic item ordering across pages. | REQ-01-056 | AC-215 |
| I-7-03 | A stale restore or rollback precondition fails closed and never mutates current row state. | REQ-01-077..REQ-01-082, REQ-01-089..REQ-01-111 | AC-215, AC-218 |

### 9.7.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-7-01 | A reviewer opens row history from the workbook surface and sees actor, timestamp, operation, diff summary, and legal rollback actions. | REQ-01-048..REQ-01-056, REQ-03-261..REQ-03-262 | AC-007, AC-215 |
| E-7-02 | A reviewer rolls back one mistaken link, tag, mention resolution, or evidence association without reverting later unrelated edits on the same row. | REQ-01-089..REQ-01-111, REQ-02-212..REQ-02-220 | AC-010, AC-216, AC-217 |
| E-7-03 | A reviewer soft-deletes and restores a row using tombstone concurrency. Other clients observe `remove` on delete and `invalidate` on restore. | REQ-01-071..REQ-01-082, REQ-01-250..REQ-01-277 | AC-011, AC-215, AC-218 |
| E-7-04 | Whole-row restore creates a new attributed revision and moves the visible row back to the selected historical snapshot without erasing prior history. | REQ-01-089..REQ-01-111 | AC-011, AC-012, AC-217 |

---

## 10. Phase 8 — Links, tags, saved views, sorting, filtering, grouping, startup selection, and projection-backed query semantics

### 10.8.1 Scope

This phase completes workbook configuration and projection-backed navigation:

- typed record links and incident-scoped tags,
- saved-view object lifecycle with exact scope vocabulary `private`, `shared`, `system`,
- workbook startup pointers `home_sheet_ref` and `default_sheet_ref`,
- query-shape validation for sort, filter, and group,
- Timeline grouping-key whitelist,
- group headers as presentation-only artifacts,
- no-op saved-view patch semantics,
- duplicate-view semantics.

### 10.8.2 Primary owner sections

- Core 01 §3.3.4 view-shaped read contract
- Core 01 §3.3.5.2 saved-view and workbook-preference routes
- Core 01 §7.4 base view-schema registry
- Core 01 §8 projection model
- Core 02 §11 saved views and workbook preferences
- Core 02 §12 typed relationships and tags
- Core 03 §2 workbook surfaces, saved views, and startup selection
- Core 03 §14 sorting, filtering, and grouping
- Core 04 §2 authorization model

### 10.8.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-8-01 | Typed links and tags are stored as structured relationship rows, not only in JSON payloads, and use the closed base relationship vocabulary. | REQ-02-011, REQ-02-163..REQ-02-176 | AC-205..AC-210 |
| U-8-02 | Ordinary saved-view create defaults omitted `scope` to `private`, rejects `scope='system'`, and persists exactly one `view_schema_id` per saved view. | REQ-03-012..REQ-03-023, REQ-01-138..REQ-01-151 | AC-146..AC-152 |
| U-8-03 | Saved-view scope uses exactly `private`, `shared`, and `system`. No obsolete `team` scope token is accepted. | REQ-03-017..REQ-03-020 | AC-146..AC-149 |
| U-8-04 | Ordinary saved-view patch allows only mutable members, rejects immutable-field mutation, and leaves `saved_view_version` / `updated_at` unchanged on a structurally valid no-op. | REQ-03-024..REQ-03-026, REQ-01-138..REQ-01-151 | AC-152 |
| U-8-05 | `home_sheet_ref` and `default_sheet_ref` remain separate objects. Workbook-open fallback order is explicit launch pointer, home pointer, default pointer, then Timeline. Invalid or hidden pointers are cleared before fallback continues. | REQ-03-027..REQ-03-032 | AC-150, AC-153 |
| U-8-06 | `filters[]`, `sort[]`, and `group_by` use stable `field_key` values only. Invalid operators, invalid arg shapes, duplicate filter keys after normalization, and invalid grouping keys are rejected by the route-owned query contract. | REQ-01-035..REQ-01-047, REQ-03-223..REQ-03-233 | AC-124, AC-184, AC-243, AC-024..AC-026 |
| U-8-07 | Timeline grouping permits only the declared whitelist keys and never serializes group headers as writable rows. | REQ-03-225..REQ-03-235 | AC-024..AC-026 |

### 10.8.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-8-01 | Saved-view create, update, duplicate, and delete persist normalized `query_json` / `layout_json`, scope rules, and authorization consequences against a real database. | REQ-01-138..REQ-01-151, REQ-03-012..REQ-03-026 | AC-146..AC-152 |
| I-8-02 | Workbook startup selection follows the documented fallback order across valid, missing, hidden, and invalid saved-view references. | REQ-01-145..REQ-01-151, REQ-03-027..REQ-03-032 | AC-150, AC-153 |
| I-8-03 | Link and tag mutations update projections, history, and view-query results atomically. | REQ-02-163..REQ-02-176, REQ-01-351..REQ-01-353 | AC-205..AC-210 |

### 10.8.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-8-01 | A user creates a private saved view, converts it to shared when allowed, duplicates a visible system view, and cannot edit the system view in place through ordinary routes. | REQ-03-017..REQ-03-026 | AC-146..AC-152 |
| E-8-02 | Workbook open honors explicit `sheet_ref`, then home pointer, then incident default, then Timeline, clearing invalid pointers along the way. | REQ-03-027..REQ-03-032 | AC-150, AC-153 |
| E-8-03 | Sorting, filtering, and grouping on Timeline produce a stable first useful viewport and deterministic final grouping order without turning group headers into rows. | REQ-01-034..REQ-01-047, REQ-03-223..REQ-03-235 | AC-014, AC-024..AC-026, AC-044, AC-184, AC-185 |

---

## 11. Phase 9 — Keyboard contract, clipboard and bulk-edit behaviors, Notes, Indicators, Parties, Assessments, and analyst-work surfaces

### 11.9.1 Scope

This phase completes the remaining workbook-native operator surfaces and high-value interaction behaviors:

- full keyboard contract,
- clipboard paste and bulk-edit behaviors,
- Notes built-in tab,
- Indicators system view behavior,
- Parties system view and text-plus-link flows,
- Compromise Assessments,
- Task Requests and Decisions,
- coordination surfaces `comm_log`, `handoff`, `status_review`, and `lesson`,
- optional standardized surfaces for findings, investigative queries, and forensic keywords when the implementation exposes them.

### 11.9.2 Primary owner sections

- Core 01 §7.4 base view-schema registry
- Core 01 §19 party and coordination-surface addendum
- Core 02 §10 notes, indicators, assessments, task requests, decisions, and artifact-backed coordination rows
- Core 02 §19 party model
- Core 03 §2 workbook surfaces
- Core 03 §11 clipboard paste and bulk create
- Core 03 §13 keyboard contract
- Core 03 §16–§20 workbook-native analyst-work and party flows
- Core 04 §2 authorization model

### 11.9.3 Unit tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| U-9-01 | The keyboard contract supports Arrow navigation, Enter, Shift+Enter, Tab, Ctrl+V, Ctrl+K, Space, Alt+H, and Esc on the workbook surface without introducing hidden macro semantics. | REQ-03-217..REQ-03-222 | AC-003, AC-005 |
| U-9-02 | Multi-cell paste uses the shared tabular-ingest contract, preserves `entity_binding_mode`, and groups the paste into one visible user action while allowing later same-field conflict resolutions to create separate change sets. | REQ-03-145..REQ-03-152 | AC-003, AC-040 |
| U-9-03 | Notes are artifact-backed `note` rows exposed through the built-in Notes tab, not through a separate storage silo. | REQ-02-067..REQ-02-071, REQ-03-004 | AC-068..AC-070, AC-112 |
| U-9-04 | Indicator system-view rows remain canonical indicator rows with pivots to source-bound observations and lifecycle history. Observation and lifecycle state stay separate from the canonical row identity. | REQ-03-005..REQ-03-006, REQ-02-072..REQ-02-082 | AC-078, AC-079, AC-121, AC-122 |
| U-9-05 | Party create or explicit create-from-text reuses a same-incident party only on unique exact match of normalized `primary_email` or `external_ref`. Raw requester or source text remains preserved alongside any linked `party_id`. | REQ-02-022, REQ-02-060..REQ-02-063, REQ-03-266..REQ-03-271 | AC-277..AC-280 |
| U-9-06 | Compromise assessments remain append-only, use only the closed assessment-state vocabulary, and keep operational response actions out of `assessment_state`. Confidence-band derivation remains deterministic. | REQ-02-083..REQ-02-093, REQ-03-250..REQ-03-254 | AC-018, AC-080..AC-084 |
| U-9-07 | Task Requests and Decisions are first-class workbook surfaces with bounded lifecycle transitions, queue fields, and owner semantics. No generalized workflow engine or mandatory timeline approval fields are introduced. | REQ-02-094..REQ-02-122, REQ-03-255..REQ-03-260 | AC-085, AC-086, AC-137..AC-145 |
| U-9-08 | `comm_log`, `handoff`, `status_review`, and `lesson` are workbook-native surfaces with the declared minimum create signal and projection-backed filter or grouping fields. | REQ-01-503..REQ-01-506, REQ-02-123..REQ-02-133, REQ-03-010..REQ-03-011, REQ-03-259, REQ-03-265 | AC-281..AC-284 |
| U-9-09 | If the implementation exposes standardized Findings, Investigative Queries, or Forensic Keywords surfaces, each exposed surface uses its declared `view_schema_id`, minimum create signal, writable fields, and workbook-native behavior. | REQ-01-507..REQ-01-509, REQ-02-135..REQ-02-138 | AC-285..AC-287 |

### 11.9.4 Integration tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| I-9-01 | A multi-row clipboard paste into Timeline creates ordered mutations, preserves unknown-column remnants where required, and respects `mention_origin` vs `entity_origin`. | REQ-03-145..REQ-03-152, REQ-02-030..REQ-02-036 | AC-003, AC-201 |
| I-9-02 | Notes, indicator, party, assessment, task, decision, and coordination surfaces persist structured state and remain queryable through workbook-native projections. | REQ-01-296..REQ-01-302, REQ-01-303..REQ-01-306, REQ-01-497..REQ-01-506, REQ-02-067..REQ-02-133 | AC-068..AC-090, AC-116..AC-122, AC-277..AC-284 |
| I-9-03 | Party-link helper fields on task or evidence records update only the hidden same-incident `*_party_id` link and do not overwrite source-preserving text. | REQ-01-502, REQ-02-021..REQ-02-022, REQ-03-268..REQ-03-271 | AC-278..AC-280 |

### 11.9.5 E2E tests

| ID | Test | Exact REQs | Exact ACs |
|---|---|---|---|
| E-9-01 | Arrow keys, Tab, Enter, Shift+Enter, Ctrl+V, Ctrl+K, Space, Alt+H, and Esc all work on the grid without forcing a page or module switch. | REQ-03-217..REQ-03-222 | AC-005 |
| E-9-02 | Pasting a representative 20×5 block into the Timeline sheet creates or updates rows through the workbook surface and preserves row identity and selection state. | REQ-03-145..REQ-03-152 | AC-003 |
| E-9-03 | Notes are available as a built-in tab, support in-grid creation, and can link to other records without leaving the workbook interaction model. | REQ-03-004, REQ-02-067..REQ-02-071 | AC-068..AC-070, AC-112, AC-116 |
| E-9-04 | An analyst creates or links a party from requester or source text, keeps the raw text visible, and gains incident-scoped party pivots and queues. | REQ-02-022, REQ-03-266..REQ-03-271 | AC-277..AC-280 |
| E-9-05 | Recording a new assessment appends a new attributed row. The sequence `unknown -> suspected -> confirmed -> cleared` and the alternative path `unknown -> disproven` remain distinguishable in history and filters. | REQ-02-083..REQ-02-093, REQ-03-250..REQ-03-254 | AC-018, AC-080..AC-084 |
| E-9-06 | Task Requests, Decisions, and coordination surfaces appear as workbook surfaces rather than separate application modules. Queue, due-date, blocked-work, handoff, and lesson views are functional. | REQ-03-005..REQ-03-011, REQ-03-255..REQ-03-260 | AC-085..AC-090, AC-281..AC-284 |
| E-9-07 | If the build exposes standardized Findings, Investigative Queries, or Forensic Keywords surfaces, each exposed surface behaves as a workbook-native surface with the declared minimum create semantics. | REQ-01-507..REQ-01-509, REQ-02-135..REQ-02-138 | AC-285..AC-287 |

---

## 12. Phase 10 — Extension profile testing hooks

Extension profiles are not part of the base implementation sequence. This section exists so teams do not drift away from the current extension route families or profile claim boundaries while planning later work.

Each extension claim still requires the **base profile first**. The AC groups listed below are the **extension deltas** that should be planned on top of the fully passing base guide.[^profile-dod]

### 12.10.1 Import Extension Profile

**Primary owner sections**

- Core 01 §2.1 Phase 2 Workbook Import Assistant
- Core 01 §17.2 Import Extension Profile public contract
- Core 02 §7.2 file-based import provenance
- Core 03 §11 file-based import assistant
- Core 04 §4.5 hostile workbook and import-content constraints

**Extension delta ACs**

- `AC-027..AC-029`
- `AC-063..AC-067`
- `AC-232`
- `AC-262..AC-265`

**Key boundaries to test**

- import sessions and import units,
- deterministic `mapping_fingerprint`,
- CSV plus bounded XLSX discovery and mapping,
- provenance capture for source bytes, parser version, and locator,
- inert handling of formulas, macros, automation, and external links,
- no auto-resolution during ingest,
- stable imports-module boundary.

### 12.10.2 Snapshot and Reporting Extension Profile

**Primary owner sections**

- Core 01 §17.3 Snapshot and Reporting Extension Profile public contract
- Core 02 §14 snapshot and release metadata
- Core 04 §2.1 release gate and §4.2 export trust boundary

**Extension delta ACs**

- `AC-030..AC-032`
- `AC-056..AC-062`
- `AC-091`
- `AC-104..AC-106`
- `AC-233`
- `AC-266..AC-269`

**Key boundaries to test**

- immutable snapshot creation,
- release-state machine,
- bound approval tuple,
- invalidation on byte or tuple change,
- self-contained outputs,
- recipient-specific redaction profiles without live-workspace withholding.

### 12.10.3 Reference Pack Extension Profile

**Primary owner sections**

- Core 01 §17.4 Reference Pack Extension Profile public contract
- Core 02 reference-pack metadata and activation state
- Core 04 §4.1 reference-pack trust boundary

**Extension delta ACs**

- `AC-033..AC-035`
- `AC-092..AC-096`
- `AC-234`
- `AC-270..AC-272`

**Key boundaries to test**

- staged import,
- verification metadata,
- explicit activation,
- retained prior active version,
- smallest disconnected bundle,
- fail-closed activation on integrity failure.

### 12.10.4 Incident Portability Extension Profile

**Primary owner sections**

- Core 01 §17.5 Incident Portability Extension Profile public contract
- Core 01 portability bundle and import phases
- Core 04 §4.2 portability trust boundary

**Extension delta ACs**

- `AC-164..AC-169`
- `AC-236`
- `AC-273..AC-276`

**Key boundaries to test**

- deterministic bundle layout,
- authoritative-source-only export,
- checksum-verified staged import,
- preservation of attribution without importing login-capable deployment-local admin state,
- tolerance for unsupported optional embedded sections.

### 12.10.5 Enterprise Authentication Extension Profile

**Primary owner sections**

- Core 01 §20 Enterprise Authentication Extension Profile public contract
- Core 04 §1.2 enterprise-auth model

**Extension delta ACs**

- `AC-036`
- `AC-235`
- `AC-288..AC-293`

**Key boundaries to test**

- providers list and provider begin routes,
- OIDC callback and SAML ACS completion,
- provider-backed sign-in terminating into the same opaque session contract as base auth,
- no auto-provisioning of local users or incident memberships.

---

## 13. Shared cross-cutting harnesses

These harnesses apply across phases and should be implemented once, then reused. The harness list is intentionally small and tied to current owner sections.

### 13.1 Envelope consistency harness

Every HTTP success and error response on a public route must use the owner-level common envelope shape. No public route returns a bare object, bare array, or bespoke error wrapper.

**Owner sections**: Core 01 §3.3.6 and route-family owners in Core 01 §3.3.2, §3.3.4, §3.3.5, §3.3.8, §3.3.9, §17, and §20.

### 13.2 Authorization re-derivation harness

Route handlers, handle issuance, handle redemption, job polling, job cancel, and incident WebSocket subscription must re-derive authorization from the caller’s current role and current incident membership at request time.

**Owner sections**: Core 04 §2, plus route-specific owners in Core 01 §3.3.5, §3.3.8, §3.3.9, and §3.3.10.

### 13.3 Mutation attribution and history-emission harness

Every successful mutating route must emit the required actor, timestamp, source, and mutation detail for its owner substrate. Incident data goes through the incident history substrate. Deployment-local account and membership administration goes through the deployment-local administrative audit substrate.

**Owner sections**: Core 02 §14–§15, Core 04 §3.

### 13.4 Idempotent replay and divergent replay harness

Every route that owns idempotency must be covered for:

- first success,
- same normalized replay,
- divergent replay with the same idempotency key,
- non-idempotent routes explicitly rejecting `client_txn_id` where the owner section says they are intentionally non-idempotent.

This harness must include `PATCH /api/v1/records/{record_id}` and therefore must cover `AC-299`.

**Owner sections**: Core 01 §3.3.5, Core 01 §3.3.8, Core 01 §3.3.9, Core 01 §17, Core 01 §20.

### 13.5 Closed-vocabulary rejection harness

All write paths must reject invalid tokens for the closed vocabularies they own.

**Owner sections**: Core 02 §18 and any route-specific write contracts that bind those tokens.

### 13.6 Writable-string normalization harness

All writable string fields bound to an owner-level string contract must normalize, trim, reject controls, or preserve `null` exactly as the contract requires.

**Owner sections**: Core 01 owner-level writable-string contract bindings and the field-level write contracts in Core 01 §7.4, §19, and Core 03 workbook write surfaces.

### 13.7 View-schema field-key conformance harness

Clients and tests must address writable and queryable fields only by stable `field_key`. Visible column labels, tab names, storage table names, or projection column names are never mutation keys.

**Owner sections**: Core 01 §3.3.4, §3.3.5, §7.4; Core 03 §14–§16.

### 13.8 Projection determinism and rebuild harness

Projection rebuilds must be deterministic from authoritative source state. Projection corruption or rebuild must never alter authoritative source rows.

**Owner sections**: Core 01 §8.

### 13.9 WebSocket lifecycle harness

The socket boundary must enforce first-message rules, replay vs reset semantics, ephemeral presence handling, heartbeat timing, `session_revoked`, and delete/restore `record_changed` semantics.

**Owner sections**: Core 01 §3.3.10 and Core 03 §4.

### 13.10 Performance fixtures and p95 measurement harness

Performance-sensitive ACs must use Fixture A and Fixture B and the owner-level p95 measurement method. Measure end-user-visible completion time, not only backend timing.

**Owner sections**: Core 04 §9 (`REQ-04-063..REQ-04-064`).

---

## 14. Phase completion checklist

A phase is complete only when all of the following are true:

1. Every row in that phase’s test tables passes in the intended test layer.
2. The shared harnesses in §13 pass for every route, event class, and mutation path introduced or materially changed by the phase.
3. All earlier phases still pass after the phase lands.
4. Any new view surface introduced in the phase is covered for:
   - query shape,
   - row identity,
   - allowed create semantics,
   - patch semantics,
   - projection maintenance,
   - authorization,
   - history or audit consequences.
5. Any mutation-bearing phase includes explicit integration coverage against the real backing boundary it depends on. Use `N/A` only when the owner section truly defines no external boundary for that work.
6. No phase claims completion by relying on behavior deferred to a later phase when the current phase already assumes that behavior.

### 14.1 Base-profile completion rule

The base implementation sequence is complete only when Phases 0 through 9 pass and the current Base claim manifest in Core 04 is fully covered, including `AC-299`.[^base-manifest][^traceability]

The current Base claim manifest is:

- `AC-001..AC-026`
- `AC-037..AC-055`
- `AC-097..AC-103`
- `AC-107..AC-112`
- `AC-116..AC-163`
- `AC-170..AC-231`
- `AC-237..AC-261`
- `AC-277..AC-287`
- `AC-294..AC-299`

---

## 15. Coverage ledger

The phase tables above are the authoritative **test-id to REQ / AC mapping**. This section adds the summary views needed to keep the guide from drifting again.

### 15.1 Phase-to-owner-section map

| Phase | Primary owner sections |
|---|---|
| Phase 0 | Core 01 §1; Core 04 §5–§8; Core 04 §12 |
| Phase 1 | Core 01 §3.3.2; Core 01 §3.3.5.1; Core 01 §3.3.10; Core 04 §1; Core 04 §3 |
| Phase 2 | Core 01 §3.3.1; Core 01 §3.3.3; Core 01 §3.3.5.1–§3.3.5.3; Core 02 §4.5; Core 04 §2–§3 |
| Phase 3 | Core 01 §3.3.5; Core 01 §7.4; Core 01 §8; Core 02 §5 and §14; Core 03 §1, §4, §6, §7, §15 |
| Phase 4 | Core 01 §3.3.5 mention / merge routes; Core 02 §6–§10; Core 03 §9 and §16 |
| Phase 5 | Core 01 §3.3.8; Core 01 §16; Core 02 §13; Core 03 §8 and §16; Core 04 §4.3 and §4.5 |
| Phase 6 | Core 01 §3.3.10; Core 01 §3.3.5; Core 03 §3–§4; Core 04 §1–§2 and §4.5 |
| Phase 7 | Core 01 §3.3.4.2 and §3.3.5; Core 02 §14–§15; Core 03 §10; Core 04 §2–§3 |
| Phase 8 | Core 01 §3.3.4; Core 01 §3.3.5.2; Core 01 §7.4; Core 01 §8; Core 02 §11–§12; Core 03 §2 and §14; Core 04 §2 |
| Phase 9 | Core 01 §7.4 and §19; Core 02 §10 and §19; Core 03 §2, §11, §13, §16–§20; Core 04 §2 |
| Phase 10 | Core 01 §17 and §20; Core 02 extension-owned provenance / release / bundle sections; Core 04 extension profile sections |

### 15.2 Base-profile AC coverage index

| AC cluster | Planned owner phase or shared harness |
|---|---|
| `AC-294..AC-298` | Phase 0 |
| `AC-123`, `AC-130..AC-163`, `AC-175..AC-180`, `AC-244..AC-250` | Phase 1 |
| `AC-170..AC-174`, `AC-211..AC-214`, `AC-219..AC-220` | Phase 2 |
| `AC-001..AC-002`, `AC-043`, `AC-107..AC-111`, `AC-125`, `AC-191..AC-199`, `AC-299` | Phase 3 |
| `AC-006`, `AC-017`, `AC-019..AC-023`, `AC-077..AC-079`, `AC-186`, `AC-188..AC-190`, `AC-209`, `AC-221..AC-225` | Phase 4 |
| `AC-004`, `AC-015..AC-016`, `AC-045`, `AC-053`, `AC-102..AC-103`, `AC-128`, `AC-154..AC-155`, `AC-251..AC-255` | Phase 5 |
| `AC-008..AC-009`, `AC-037..AC-042`, `AC-047`, `AC-126`, `AC-129`, `AC-131..AC-163`, `AC-203..AC-204`, `AC-226..AC-230` | Phase 6 |
| `AC-007`, `AC-010..AC-012`, `AC-215..AC-218` | Phase 7 |
| `AC-013..AC-014`, `AC-024..AC-026`, `AC-146..AC-153`, `AC-184..AC-185`, `AC-205..AC-210` | Phase 8 |
| `AC-003`, `AC-005`, `AC-018`, `AC-068..AC-070`, `AC-078..AC-090`, `AC-112`, `AC-116..AC-122`, `AC-137..AC-145`, `AC-277..AC-287` | Phase 9 |
| `AC-048..AC-055`, `AC-097..AC-103`, `AC-237..AC-261` | Shared harnesses across the relevant mutation, evidence, query, authorization, and conformance phases |

### 15.3 Extension-only AC index

| Extension profile | Delta ACs beyond base | Planned hook phase |
|---|---|---|
| Import | `AC-027..AC-029`, `AC-063..AC-067`, `AC-232`, `AC-262..AC-265` | Phase 10 |
| Snapshot and Reporting | `AC-030..AC-032`, `AC-056..AC-062`, `AC-091`, `AC-104..AC-106`, `AC-233`, `AC-266..AC-269` | Phase 10 |
| Reference Pack | `AC-033..AC-035`, `AC-092..AC-096`, `AC-234`, `AC-270..AC-272` | Phase 10 |
| Incident Portability | `AC-164..AC-169`, `AC-236`, `AC-273..AC-276` | Phase 10 |
| Enterprise Authentication | `AC-036`, `AC-235`, `AC-288..AC-293` | Phase 10 |

### 15.4 Conditional surface note

The standardized Findings, Investigative Queries, and Forensic Keywords surfaces are contract-defined as **optional standardized workbook surfaces when exposed**. If the implementation exposes them in the base build, cover them in Phase 9 and include `AC-285..AC-287` in the claimed surface set. If the implementation omits them, record that omission explicitly in the implementation plan and verify the conformance interpretation against the current owner sections before asserting those ACs.[^traceability]

---

## Sources

[^precedence]: `00_document_set_status_and_precedence.md`, especially §§1–5 on authority, precedence, and contract ownership.
[^traceability]: `F_source_traceability_matrix.md`, especially §§F.2, F.4, and F.6 for requirement-to-AC mapping and profile definition-of-done navigation.
[^base-manifest]: `04_security_deployment_and_conformance.md`, especially §9.0.1 Base claim manifest and §12 deployment-configuration contract.
[^profile-dod]: `F_source_traceability_matrix.md`, §F.6 Profile Definition-of-Done navigation.
