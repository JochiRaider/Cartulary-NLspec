# Appendix B: Architecture Diagrams and Explanatory Source Extract

This appendix is **non-normative**.

It preserves the architecture-oriented explanatory material, diagrams, tables, and rationale from the exploratory source artifact.

## 4. Recommended architecture

### Primary recommendation

Use a **single web application container** (UI + API + WebSocket hub + background jobs) with **Postgres** and **S3-compatible object storage** as separate services. This is a modular monolith, not a distributed platform.

### System context diagram

```mermaid
flowchart LR
    AnalystA[Analyst A browser]
    AnalystB[Analyst B browser]
    Reviewer[Reviewer browser]

    subgraph Deploy[Single-tenant deployment]
        App[Workbook web app\nUI + API + WebSocket + background jobs]
        PG[(Postgres)]
        OBJ[(S3-compatible object store)]
    end

    IdP[Optional OIDC / SAML IdP]

    AnalystA <-- HTTPS / WebSocket --> App
    AnalystB <-- HTTPS / WebSocket --> App
    Reviewer <-- HTTPS / WebSocket --> App

    App <-- SQL --> PG
    App <-- Presigned upload --> OBJ
    App <-- Validated preview/download handle redeem --> OBJ
    App -. optional enterprise-auth extension .-> IdP
```

### Container/component diagram

```mermaid
flowchart TB
    subgraph Browser[Browser client]
        Grid[Workbook grid]
        Inspector[Detail / relationship inspector]
        Sync[Local pending patch queue]
    end

    subgraph App[Modular monolith]
        Auth[Auth & session module]
        Incident[Incident & membership module]
        Timeline[Timeline capture module]
        Entity[Entity & mention-resolution module]
        Evidence[Evidence & object module]
        Imports[Imports & tabular-ingest module]
        Links[Link graph & tagging module]
        Revisions[Revision / rollback module]
        Projections[Projection & search module]
        Collab[Presence / live update hub]
    end

    PG[(Postgres)]
    OBJ[(Object store)]

    Browser --> Auth
    Browser --> Timeline
    Browser --> Entity
    Browser --> Evidence
    Browser --> Imports
    Browser --> Links
    Browser --> Revisions
    Browser --> Collab

    Timeline --> PG
    Entity --> PG
    Evidence --> PG
    Evidence --> OBJ
    Imports --> PG
    Links --> PG
    Revisions --> PG
    Projections --> PG
    Collab --> PG
```

### Major components

| Component                   | Concrete responsibility                                                                                |
| --------------------------- | ------------------------------------------------------------------------------------------------------ |
| Browser client              | Virtualized grid, keyboard navigation, paste handling, inspector, evidence preview, presence UI        |
| Auth module                 | Local accounts, sessions, MFA, and optional OIDC/SAML provider mapping                                  |
| Timeline module             | Rapid row creation, inline edits, rough capture storage                                                |
| Entity module               | Host/identity records, aliases, unresolved mentions, resolution workflows                              |
| Evidence module             | Evidence lifecycle, upload finalization, object metadata, same-origin handle issuance and redemption, safe preview transforms, blocked preview states, linking evidence to records        |
| Imports & tabular-ingest module | CSV/XLSX adaptation, workbook inspection, preview/header mapping, provenance capture, compatibility shims, and background import apply |
| Link graph & tagging        | Typed relationships and lightweight labels                                                             |
| Revision module             | Change sets, mutation-entry history, row-centric revisions, rollback                                   |
| Projection & search         | Build `*_grid_projection` tables and search vectors for sheet-like views                               |
| Reference data module       | Reference-pack manifests, type/icon registries, framework mappings, integrity verification             |
| Reporting & snapshot module | Immutable incident snapshots, canonical export-model generation, self-contained report/render pipeline |
| Collaboration hub           | WebSocket presence and live row updates                                                                |

### Preferred architecture pattern

A **modular monolith** is the right fit. This problem’s complexity is in mutation semantics, projections, and UX; microservices would add operational and debugging cost without helping the hardest problem. A single codebase with clear module boundaries is easier to deploy in a flyaway kit, easier to reason about during incident work, and easier to ship with deterministic versions.

### Module boundaries

I would define internal module boundaries as:

- `auth`
- `incidents`
- `timeline`
- `entities`
- `evidence`
- `imports`
- `links`
- `revisions`
- `projections`
- `reference_data`
- `reporting`
- `collaboration`

These are internal packages/modules with explicit service interfaces, not separate deployables.

The split worth making explicit is that clipboard interaction stays on the core workbook hot path, while file-based import sits behind a dedicated `imports` module. Clipboard, CSV, and XLSX adapters can still normalize into the same canonical `TabularSource` and shared mapping engine, but only the imports module should absorb parser drift, workbook-shape heuristics, preview/header mapping, and other spreadsheet-compatibility maintenance.

A bounded import contract is more realistic than “Excel support” in the abstract. The first file-based onboarding path should focus on CSV and selected-sheet or selected-region XLSX import, preserve provenance and unknown columns, treat formulas as inert input, and warn, downgrade, or reject unsupported workbook features instead of leaking those semantics into the core workbook modules.

### Storage choices

- **Postgres** stores all structured records, metadata, links, revisions, tags, saved views, projections, reference-pack manifests, and snapshot metadata.
- **S3-compatible object storage** stores binary evidence and optional rendered export artifacts. In flyaway/on-prem, use **MinIO**. In cloud, use native S3/GCS/Azure Blob behind the same abstraction.
- **Reference packs** such as ATT&CK/D3FEND/VERIS mappings, host/evidence type registries, and other optional vocabularies version separately from incident records. Their manifests and integrity metadata belong in Postgres; pack payloads may live on local disk or object storage behind the same abstraction.
- Do **not** store large binary evidence in Postgres. It bloats backups, complicates restore times, and makes portability worse.

### Deployment configuration examples

The examples below are illustrative only. Core 04 §12 remains the normative owner of the deployment-configuration contract.

#### Example 1: disconnected `config.toml`

```toml
config_schema_id = "cartulary.deployment_config.v1"
deployment_profile = "disconnected"
bootstrap.first_admin_manifest_path = "/run/secrets/cartulary-bootstrap-admin.json"

[roots.database_storage]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/postgres"

[roots.object_storage]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/object-store"

[roots.reference_pack_storage]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/reference-packs"

[roots.temporary_work]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/tmp"

[roots.export_outputs]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/exports"
```

#### Example 2: externalized `config.toml`

```toml
config_schema_id = "cartulary.deployment_config.v1"
deployment_profile = "cloud"
bootstrap.first_admin_manifest_path = "/run/secrets/cartulary-bootstrap-admin.json"

[roots.database_storage]
binding_kind = "managed_service"
service_ref = "postgres.primary"

[roots.object_storage]
binding_kind = "managed_service"
service_ref = "object.primary"

[roots.reference_pack_storage]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/reference-packs"

[roots.temporary_work]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/tmp"

[roots.export_outputs]
binding_kind = "filesystem_root"
path = "/var/lib/cartulary/exports"
```

#### Example 3: disconnected container mounts

```yaml
services:
  app:
    volumes:
      - /etc/cartulary/config.toml:/etc/cartulary/config.toml:ro
      - /var/lib/cartulary/bootstrap/cartulary-bootstrap-admin.json:/run/secrets/cartulary-bootstrap-admin.json:ro
      - /var/lib/cartulary/reference-packs:/var/lib/cartulary/reference-packs
      - /var/lib/cartulary/tmp:/var/lib/cartulary/tmp
      - /var/lib/cartulary/exports:/var/lib/cartulary/exports

  postgres:
    volumes:
      - /var/lib/cartulary/postgres:/var/lib/postgresql/data

  minio:
    volumes:
      - /var/lib/cartulary/object-store:/data
```

Missing required runtime-root keys remain invalid at runtime even when the official disconnected examples use canonical paths.

The read-only bootstrap manifest mount above remains conformant because one-time bootstrap semantics derive from persisted deployment-local completion state, not from deleting or mutating the manifest file after consumption.

#### Example 3a: illustrative bootstrap-admin manifest

```json
{
  "bootstrap_schema_id": "cartulary.bootstrap_admin.v1",
  "bootstrap_artifact_id": "00000000-0000-0000-0000-000000000001",
  "email": "admin@example.com",
  "display_name": "Deployment Admin",
  "initial_password": "correct horse battery staple 123",
  "mfa_required": true
}
```

#### Illustrative operator note: bootstrap failure modes

- `bootstrap_manifest_path_missing`: the configured path does not exist when bootstrap is required.
- `bootstrap_manifest_schema_invalid`: the file parses but does not satisfy `cartulary.bootstrap_admin.v1`.
- `bootstrap_email_conflict`: the manifest email already exists on a local user row.
- `bootstrap_recovery_not_supported`: the deployment has no active deployment admin but already has a persisted bootstrap-completion marker.

#### Example 4: illustrative resource-limit registry excerpt

```toml
[limits.object_blobs]
max_declared_byte_size = 536870912

[limits.imports]
max_csv_source_bytes = 33554432
max_xlsx_source_bytes = 67108864
max_rows = 100000
max_columns = 256
max_cells = 5000000

[limits.archives]
default_max_extracted_bytes = 2147483648
max_compression_ratio = 100
max_members = 10000

[limits.reference_packs]
max_extracted_bytes = 536870912

[limits.incident_bundles]
max_extracted_bytes = 68719476736

[limits.previews]
max_previewable_payload_bytes = 33554432
max_text_inline_bytes = 1048576
```


### Illustrative benchmark topology and benchmark-manifest example

The examples below are illustrative only. Core 04 §9 remains the normative owner of the claim-bearing benchmark profile, benchmark-manifest contract, and measurement-predicate registry. These examples do not create alternate thresholds or alternate profile identifiers.

#### Example 5: claim-bearing benchmark topology

```mermaid
flowchart LR
    Harness[Playwright harness\ncartulary.bench.harness.playwright.v1]
    Client[Benchmark client\naws.ec2.c7i.2xlarge\nChromium headed]
    Trace[Seeded live-update trace\ncartulary.perf.live_updates_25sessions.v1]

    subgraph Bench[Claim-bearing benchmark rig]
        App[Application server\naws.ec2.c7i.2xlarge]
        PG[Postgres\naws.ec2.i4i.2xlarge\ninstance-store NVMe]
        OBJ[Object store\naws.ec2.c7i.xlarge\ngp3 SSD]
    end

    Harness --> Client
    Trace --> App
    Client <-- 1 Gbps / RTT <= 2 ms --> App
    App <-- RTT <= 1 ms --> PG
    App <-- RTT <= 1 ms --> OBJ
```

#### Example 6: illustrative `benchmark_manifest.json`

```json
{
  "benchmark_manifest_schema_id": "cartulary.benchmark_manifest.v1",
  "benchmark_profile_id": "cartulary.perf.desktop_ref.v1",
  "criterion_ids": ["AC-043", "AC-044"],
  "measurement_predicate_ids": [
    "perf.selection_change.v1",
    "perf.focus_change.v1",
    "perf.typing_ack.v1",
    "perf.timeline_blank_row_create.v1",
    "perf.view_change.first_useful_viewport.v1",
    "perf.view_change.stable_viewport.v1"
  ],
  "fixture_ids": ["fixture_a"],
  "traffic_trace_id": "cartulary.perf.live_updates_25sessions.v1",
  "seed": 20260405,
  "warmup_passes": 1,
  "browser_engine": "chromium",
  "browser_build": "134.0.6998.35",
  "client_runner_id": "aws.ec2.c7i.2xlarge",
  "client_os_image_id": "cartulary.bench.ubuntu_24_04_client.2026q1",
  "app_runner_id": "aws.ec2.c7i.2xlarge",
  "app_os_image_id": "cartulary.bench.ubuntu_24_04_app.2026q1",
  "postgres_runner_id": "aws.ec2.i4i.2xlarge",
  "postgres_os_image_id": "cartulary.bench.ubuntu_24_04_postgres.2026q1",
  "postgres_storage_class": "instance_store_nvme",
  "object_store_runner_id": "aws.ec2.c7i.xlarge",
  "object_store_os_image_id": "cartulary.bench.ubuntu_24_04_object.2026q1",
  "object_store_storage_class": "gp3_ssd",
  "benchmark_harness_id": "cartulary.bench.harness.playwright.v1",
  "benchmark_harness_version": "2026.04.0",
  "run_started_at": "2026-04-05T12:00:00Z",
  "run_completed_at": "2026-04-05T12:08:00Z",
  "sample_count": 100,
  "artifact_bundle_sha256": "3c4e14d2c8f4d19c2f8c9a04d5f0b90d5d8e4fca6b6ef1bc6a4e2a7c3a35d8f1",
  "security_controls_state": "enabled"
}
```

#### Example 7: illustrative network-shaping recipe

```bash
# Cap client-to-app throughput at 1 Gbps and apply symmetric delay so effective RTT
# stays at or below 2 ms with jitter at or below 1 ms and no packet loss.
tc qdisc replace dev eth0 root handle 1: tbf rate 1000mbit burst 256kb limit 1mbit
tc qdisc add dev eth0 parent 1:1 handle 10: netem delay 1ms 0.5ms loss 0%

# Apply the same recipe in the reverse direction on the paired link.
```

### Reference packs, type registries, and view contracts

- Incident records, evidence envelopes, revisions, saved views, and report snapshots are **incident data**.
- Framework mappings, type/icon registries, evidence vocabularies, and optional enrichment datasets are **reference packs** that version independently of incidents.
- Each built-in sheet or system view is declared by a **`view_schema`** contract that names the source record types, computed columns, required reference packs, default sort key, filter semantics, and write-back rules.
- For required base coordination surfaces `cartulary.view.comm_log.v1`, `cartulary.view.handoff.v1`, `cartulary.view.status_review.v1`, and `cartulary.view.lesson.v1`, the `view_schema_id` is the canonical public workbook-surface identity. Helper or preset saved views over the same schema are separate, non-canonical objects.
- The current profile does not standardize pack-dependent ATT&CK, D3FEND, or VERIS workbook `view_schema` surfaces. Any future framework workbook surface should be standardized per framework or tightly bounded family, including canonical `view_schema_id`, required pack keys, field registry, discoverability, writeability, degradation, and export behavior.
- The core workbook must remain usable when optional reference packs are absent. Missing packs may disable overlays or show degraded labels, but they must not block capture or editing.
- Pack activation and updates must verify checksum and, when available, signature or trusted-source metadata before use.

### Backup, restore, portability, failure modes

- **Backups**: Postgres base backup + WAL archiving; object-store bucket snapshot/versioning.
- **Restore**: restore Postgres, restore blob store, then rebuild projection tables. Projection tables are disposable caches.
- **Portability**: export/import of a whole incident should be possible as a manifest + NDJSON/CSV + referenced blobs archive.
- **Failure modes**:
  - App container down: sessions drop, no data loss.
  - Postgres down: system unavailable.
  - Object store down: rows remain editable, but evidence upload/download fails.
  - Projection corruption: rebuild from source tables; source of truth remains intact.

### Projections for grid-like views

Do **not** use Postgres materialized views for hot workbook screens. Their refresh semantics are too coarse for row-by-row collaborative editing. Use **projection tables** such as:

- `timeline_grid_projection`
- `host_grid_projection`
- `identity_grid_projection`
- `artifact_grid_projection`
- `evidence_grid_projection`
- `indicator_grid_projection` over canonical indicator records, with observation-derived counts and lifecycle summaries

Each projection table is **one row per primary record**, denormalized for sheet use. For the Indicators system view, the primary record is the canonical indicator, not the source artifact or observation row. The app updates affected projection rows in the same transaction as the source write. Every projection row exposed to the client must carry the stable `record_id` and `row_version` used for optimistic writes; the client must not infer identity from row position or displayed values. If needed, a rebuild command can regenerate the projections.

### Report and presentation export direction

Reports and presentation artifacts should be treated as a **subsystem**, not as direct ad hoc reads from live workbook tables. The system should capture a `snapshot_at`, materialize a canonical export model such as `incident_report_model.json`, and render derivative outputs like Markdown reports, Mermaid diagram sources, Slidev decks, and HTML reports from that immutable view.

UI visualizations, report sections, framework rollups, and future exports should consume the same canonical derivation/query layer or an explicitly versioned snapshot of it. That keeps filtering, counts, and inclusion semantics consistent across interactive and exported surfaces, provides stable exported identifiers and ordering, and creates a clean place to apply redaction rules. It also leaves room for operator-facing reenactment surfaces, such as Asciinema-style terminal walkthroughs generated from selected command-line evidence, while maintaining a clear distinction between source evidence and generated presentation material.

Generated report artifacts must be **self-contained**: they cannot depend on remote JS, CSS, or font assets at render time. Report builds, snapshot generation, and heavy presentation rendering should run as background jobs so live grid editing remains responsive.

### Long-running operations and background jobs

Lookups, imports, reference-pack refreshes, snapshot generation, report builds, and evidence processing should run as background jobs with progress, cancellation, retry-safe status, and non-blocking UI behavior. Grid editing and row creation must remain responsive while those jobs run.
