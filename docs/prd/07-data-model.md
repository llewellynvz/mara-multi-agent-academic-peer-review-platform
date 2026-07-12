# 7. Data model and storage

Product Requirements Document. MARA (Multi-Agent Academic Peer-Review Architecture).
Psynalytics. Author: Prof. Llewellyn van Zyl, PhD. Status: draft for build. Language: British English.

This section defines the persistent data model, the storage layout on disk, the invariants that protect the evidence ledger, and the rules that keep the pipeline worker as the single writer of pipeline state. The default deployment stores everything in one SQLite database file managed through Drizzle ORM. A Postgres profile is available for laboratories that prefer a server database, and its differences are listed at the end. Requirements in this section carry the `DATA-` prefix and are written as testable statements.

## 7.1 Storage engine and conventions

**DATA-01** The default deployment stores all application state in a single SQLite database file at `data/mara.db`, opened in WAL journal mode (`PRAGMA journal_mode = WAL`) with `PRAGMA foreign_keys = ON` and `PRAGMA busy_timeout = 5000`.

**DATA-02** The schema is created and migrated through Drizzle ORM migrations checked into the repository. Each migration is numbered and forward-only. The current schema version is recorded in a `schema_migrations` table, and the application refuses to start if the file version is ahead of the binary.

**DATA-03** All timestamp columns store text in ISO-8601 UTC form with a trailing `Z` (for example `2026-07-12T14:03:11.482Z`). No timestamp is stored in local time.

**DATA-04** Primary keys are UUIDv7 text values, except `findings.id` which is a namespaced identifier (see DATA-09) and `settings.key` which is the natural key. UUIDv7 is chosen so keys sort by creation time.

**DATA-05** Boolean values are stored as `INTEGER` with the values `0` and `1`. Monetary values are stored as `REAL` in United States dollars. Token counts, byte sizes, and latencies are stored as `INTEGER`.

## 7.2 Entities and columns

### reviews

The run header. One row per manuscript under review.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| slug | TEXT | NOT NULL, UNIQUE |
| title | TEXT | NULL |
| status | TEXT | NOT NULL, DEFAULT `created`, CHECK in (`created`, `queued`, `sanitizing`, `running`, `paused`, `awaiting_input`, `completed`, `failed`, `cancelled`) |
| current_phase | TEXT | NULL, CHECK in (`phase_0` … `phase_8`) |
| recommendation | TEXT | NULL, CHECK in (`accept`, `minor_revision`, `major_revision`, `reject_and_resubmit`, `reject`) |
| recommendation_confidence | REAL | NULL, CHECK between 0 and 1 |
| provider_profile | TEXT | NOT NULL, DEFAULT `default` |
| options_json | TEXT | NOT NULL, DEFAULT `{}` |
| created_at | TEXT | NOT NULL |
| updated_at | TEXT | NOT NULL |
| started_at | TEXT | NULL |
| completed_at | TEXT | NULL |
| error_class | TEXT | NULL |

Indexes: `idx_reviews_status (status)`, `idx_reviews_created_at (created_at)`.

### manuscripts

The submitted document and its derived artefacts. One manuscript per review in v1.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| original_filename | TEXT | NOT NULL |
| mime_type | TEXT | NOT NULL |
| blob_path | TEXT | NOT NULL |
| byte_size | INTEGER | NOT NULL |
| sha256 | TEXT | NOT NULL, 64 hex characters |
| sanitized_text | TEXT | NULL |
| tei_structure_path | TEXT | NULL |
| quarantine_tier | INTEGER | NULL, CHECK in (1, 2, 3) |
| quarantine_log_json | TEXT | NULL |
| ingested_at | TEXT | NOT NULL |
| sanitized_at | TEXT | NULL |

Constraints: `UNIQUE (review_id, sha256)`. Index: `idx_manuscripts_review (review_id)`. The original blob stays on disk (DATA-18) and is never mutated after ingestion. `sanitized_text` and the TEI structure map are written by the worker after Phase 0 clears the quarantine gate.

### findings (the evidence ledger)

The append-only ledger. Every claim in every deliverable traces to a row here. The 13 conceptual columns from the review architecture map to the columns below.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY, namespaced (DATA-09) |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| agent | TEXT | NOT NULL |
| phase | TEXT | NOT NULL |
| type | TEXT | NOT NULL |
| claim | TEXT | NOT NULL |
| manuscript_anchor | TEXT | NOT NULL |
| epistemic_status | TEXT | NOT NULL, CHECK in (`Known`, `Inferred`, `Assumption`) |
| confidence | REAL | NOT NULL, CHECK between 0 and 1 |
| confidence_band | TEXT | NOT NULL, CHECK in (`Green`, `Yellow`, `Red`) |
| severity | TEXT | NOT NULL, CHECK in (`none`, `minor`, `moderate`, `major`, `fatal`) |
| fixability | TEXT | NOT NULL, CHECK in (`easy`, `moderate`, `hard`, `not_fixable_from_current_study`, `unclear`) |
| scope | TEXT | NOT NULL, CHECK in (`author_facing`, `editor_only`, `both`) |
| narrative_context | TEXT | NULL |
| recommended_action | TEXT | NULL |
| supersedes_id | TEXT | NULL, REFERENCES findings(id) |
| created_at | TEXT | NOT NULL |

Indexes: `idx_findings_review (review_id)`, `idx_findings_review_phase (review_id, phase)`, `idx_findings_supersedes (supersedes_id)`, `idx_findings_severity (review_id, severity)`.

**DATA-06** The `confidence_band` is derived from `confidence` at write time by the repository using these exact boundaries: `Red` when `confidence < 0.70`, `Yellow` when `0.70 <= confidence < 0.98`, `Green` when `confidence >= 0.98`. The repository sets the band. It is never supplied by the calling agent, so band and value can never disagree.

**DATA-07** Corrections never edit a row. A correction is a new finding row whose `supersedes_id` points at the identifier of the row it replaces. The superseding row carries its own identifier, confidence, and evidence.

**DATA-08** The current state of the ledger is read through the view `v_current_findings`, which returns every finding whose `id` is not referenced by any other row's `supersedes_id`. There is no mutable `superseded` flag, because a flag would require an UPDATE and UPDATEs are forbidden (DATA-11).

**DATA-09** Finding identifiers follow the pattern `REV-<LENS>-<NNNN>`, where `<LENS>` is the uppercase lens or agent namespace (for example `STAT`, `METH`, `INTEG`) and `<NNNN>` is a zero-padded, per-namespace, per-review sequence starting at `0001`. Identifiers are allocated inside the same transaction that inserts the row, so no two findings in one review share an identifier.

### phase_checkpoints

Durable workflow state for the Mastra pipeline, one row per review per phase.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| phase | TEXT | NOT NULL |
| status | TEXT | NOT NULL, CHECK in (`pending`, `in_progress`, `completed`, `failed`, `skipped`) |
| gate_verdict | TEXT | NULL, CHECK in (`pass`, `revise`, `revise_specialist`, `block`, `arbitrated`) |
| fix_cycle_count | INTEGER | NOT NULL, DEFAULT 0 |
| snapshot_json | TEXT | NULL |
| started_at | TEXT | NULL |
| completed_at | TEXT | NULL |
| updated_at | TEXT | NOT NULL |

Constraints: `UNIQUE (review_id, phase)`. Index: `idx_checkpoints_review (review_id)`. `snapshot_json` holds the Mastra resume token so a run can restart after a crash from the last completed phase. `fix_cycle_count` enforces the two-cycle release-gate cap.

### review_events (timeline, append-only)

The audit timeline. Phase transitions, gate verdicts, arbitrations, control acknowledgements, and every outbound web query are recorded here.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| seq | INTEGER | NOT NULL |
| ts | TEXT | NOT NULL |
| kind | TEXT | NOT NULL, CHECK in (`phase_transition`, `gate_verdict`, `arbitration`, `web_query`, `control_ack`, `deliverable_released`, `finding_recorded`, `run_terminal`, `error`) |
| phase | TEXT | NULL |
| payload_json | TEXT | NOT NULL, DEFAULT `{}` |
| egress_target | TEXT | NULL, CHECK in (`crossref`, `openalex`, `semantic_scholar`) |
| egress_query | TEXT | NULL |

Constraints: `UNIQUE (review_id, seq)`. Index: `idx_events_review_seq (review_id, seq)`. `seq` is a per-review monotonic counter that also drives Server-Sent Events reconnection (see API-26 in Section 8). The `finding_recorded` and `run_terminal` kinds carry the persisted finding-headline and terminal-outcome events that the stream replays exactly by `Last-Event-ID`. The ephemeral lens, cost, ETA, and log ticks are not stored here per tick. Cost lives in `dispatches` and lens state in `phase_checkpoints`, and both are re-derived on reconnect. For `web_query` events, `egress_target` and `egress_query` record the exact signed query that left the machine.

### dispatches (append-only)

One row per model call. Drives the local statistics panel and the optional Langfuse export.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| phase | TEXT | NOT NULL |
| agent | TEXT | NOT NULL |
| provider | TEXT | NOT NULL, CHECK in (`anthropic`, `openai`, `google`, `local`) |
| model | TEXT | NOT NULL |
| prompt_version | TEXT | NOT NULL |
| tokens_in | INTEGER | NOT NULL, DEFAULT 0 |
| tokens_out | INTEGER | NOT NULL, DEFAULT 0 |
| tokens_cached | INTEGER | NOT NULL, DEFAULT 0 |
| latency_ms | INTEGER | NOT NULL |
| cost_usd | REAL | NOT NULL, DEFAULT 0 |
| retries | INTEGER | NOT NULL, DEFAULT 0 |
| status | TEXT | NOT NULL, CHECK in (`success`, `error`) |
| error_class | TEXT | NULL |
| langfuse_trace_id | TEXT | NULL |
| created_at | TEXT | NOT NULL |

Index: `idx_dispatches_review_phase (review_id, phase)`. `langfuse_trace_id` is null when Langfuse is not configured (Section 13).

### rubric_scores

The 15-criterion scoring grid. Provisional scores are written by the meta-reviewer and promoted to final at Phase 7.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| criterion | TEXT | NOT NULL |
| criterion_index | INTEGER | NOT NULL, CHECK between 1 and 15 |
| score | INTEGER | NOT NULL, CHECK between 0 and 5 |
| justifying_finding_ids | TEXT | NOT NULL, DEFAULT `[]` |
| state | TEXT | NOT NULL, CHECK in (`provisional`, `final`) |
| updated_at | TEXT | NOT NULL |

Constraints: `UNIQUE (review_id, criterion_index)`. Index: `idx_rubric_review (review_id)`. `justifying_finding_ids` is a JSON array of finding identifiers. A score with an empty array is a build failure at the release gate.

### deliverables

The shipped files and their release state.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| kind | TEXT | NOT NULL, CHECK in (`peer_review_report`, `reviewer_private_notes`, `ledger_export`, `run_archive`) |
| format | TEXT | NOT NULL, CHECK in (`docx`, `md`, `zip`) |
| path | TEXT | NOT NULL |
| checksum | TEXT | NOT NULL, sha256 hex |
| byte_size | INTEGER | NOT NULL |
| released | INTEGER | NOT NULL, DEFAULT 0 |
| released_at | TEXT | NULL |
| created_at | TEXT | NOT NULL |

Constraints: `UNIQUE (review_id, kind, format)`. Index: `idx_deliverables_review (review_id)`. The two report kinds each carry a `docx` and an `md` row, `ledger_export` carries `md` only, and `run_archive` carries `zip` only.

**DATA-10** `released` is set to `1` only after the review-final-critic returns a passing verdict for the run. The download endpoint (Section 8) refuses any deliverable whose `released` value is `0`.

### provider_keys

Encrypted API credentials. Empty when the instance runs in session-only key mode (Section 11).

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| provider | TEXT | NOT NULL, CHECK in (`anthropic`, `openai`, `google`, `local`) |
| label | TEXT | NULL |
| ciphertext | BLOB | NOT NULL |
| iv | BLOB | NOT NULL, 12 bytes |
| auth_tag | BLOB | NOT NULL, 16 bytes |
| wrapped_dek | BLOB | NOT NULL |
| dek_iv | BLOB | NOT NULL, 12 bytes |
| dek_auth_tag | BLOB | NOT NULL, 16 bytes |
| base_url | TEXT | NULL |
| created_at | TEXT | NOT NULL |
| updated_at | TEXT | NOT NULL |

Constraints: `UNIQUE (provider, label)`. `base_url` carries the endpoint for a local or Ollama-compatible provider. The encryption scheme is specified in Section 11.

### settings

Key-value configuration.

| Column | Type | Constraints |
| --- | --- | --- |
| key | TEXT | PRIMARY KEY |
| value_json | TEXT | NOT NULL |
| updated_at | TEXT | NOT NULL |

Holds the provider routing profiles, the egress polite-pool identifiers (Crossref mailto, OpenAlex mailto and optional key, Semantic Scholar key), the instance passphrase hash, the key persistence mode, and the telemetry opt-in flag.

### run_commands (control channel)

The one table the API server writes into during a run. It carries pause, resume, cancel, and retry-phase intents from the user interface to the worker.

| Column | Type | Constraints |
| --- | --- | --- |
| id | TEXT | PRIMARY KEY |
| review_id | TEXT | NOT NULL, REFERENCES reviews(id) ON DELETE CASCADE |
| command | TEXT | NOT NULL, CHECK in (`run`, `pause`, `resume`, `cancel`, `retry_phase`) |
| args_json | TEXT | NOT NULL, DEFAULT `{}` |
| created_at | TEXT | NOT NULL |

Index: `idx_run_commands_review (review_id, created_at)`. Rows are insert-only from the API server. The worker reads unacknowledged commands and records consumption by inserting a `control_ack` row into `review_events`, so no two processes ever write the same table row.

## 7.3 Append-only ledger enforcement

The ledger is protected in two layers, so a bug in the application code cannot corrupt the audit trail.

**DATA-11** The repository layer exposes only an `insertFinding` method for the `findings` table. No update or delete method exists in the findings repository, and the same holds for `review_events` and `dispatches`.

**DATA-12** SQLite triggers enforce the invariant at the database level. Any UPDATE on `findings` raises. Any DELETE on `findings` raises unless a purge is in progress (DATA-19). The same trigger pair guards `review_events` and `dispatches`.

```sql
CREATE TRIGGER findings_no_update
BEFORE UPDATE ON findings
BEGIN
  SELECT RAISE(ABORT, 'findings is append-only');
END;

CREATE TRIGGER findings_no_delete
BEFORE DELETE ON findings
WHEN NOT EXISTS (SELECT 1 FROM temp._mara_purge)
BEGIN
  SELECT RAISE(ABORT, 'findings is append-only');
END;
```

**DATA-13** A required unit test asserts that a direct UPDATE and a direct DELETE against `findings` both raise, and that a correction is expressed only as a new row with `supersedes_id` set (this test is one of the three security suites in Section 11).

## 7.4 Single-writer rule and phase-boundary merge

**DATA-14** The pipeline worker is the only process that writes to the pipeline tables `findings`, `phase_checkpoints`, `review_events`, `dispatches`, `rubric_scores`, and `deliverables`, and it is the only process that issues UPDATEs to `reviews` and `manuscripts` after those rows exist.

**DATA-15** The API server writes only to `settings`, `provider_keys`, and `run_commands`, and it performs the initial INSERT of a `reviews` row on create and a `manuscripts` row on upload. Once a run starts, the API server never writes to those two tables again. It signals the worker through `run_commands` (DATA-14 and the run_commands table above resolve the apparent conflict between "worker is the only writer of pipeline tables" and the existence of write endpoints).

**DATA-16** Agents do not write to the database. Each agent returns finding fragments to the worker. The worker validates them and merges them into `findings` at the phase boundary inside a single transaction, so a phase either commits all of its findings or none of them.

**DATA-17** Sequence allocation for `findings.id` (DATA-09) and `review_events.seq` runs inside the merge transaction, so identifiers stay gapless and ordered even when several agents ran in parallel during the phase.

## 7.5 Entity relationship map

```
reviews ─1──N─ manuscripts        (ON DELETE CASCADE)
reviews ─1──N─ findings           (ON DELETE CASCADE, append-only)
reviews ─1──N─ phase_checkpoints  (ON DELETE CASCADE)
reviews ─1──N─ review_events      (ON DELETE CASCADE, append-only)
reviews ─1──N─ dispatches         (ON DELETE CASCADE, append-only)
reviews ─1──N─ rubric_scores      (ON DELETE CASCADE)
reviews ─1──N─ deliverables       (ON DELETE CASCADE)
reviews ─1──N─ run_commands       (ON DELETE CASCADE)

findings ──supersedes_id──▶ findings   (self reference, corrections)
rubric_scores ──justifying_finding_ids (JSON array)──▶ findings

provider_keys   (standalone)
settings        (standalone)
```

## 7.6 File storage layout

**DATA-18** Large binary and text artefacts live on disk, not in the database. The database stores a relative path and an sha256 checksum for each artefact, and every read verifies the file against its stored checksum before use.

```
data/
  mara.db                          SQLite database
  mara.db-wal
  mara.db-shm
  blobs/
    <review_id>/
      manuscript/original.<ext>    immutable original upload
      manuscript/sanitized.txt     written after Phase 0
      manuscript/structure.tei.xml TEI structure map
  deliverables/
    <review_id>/
      Peer-Review-Report.docx
      Peer-Review-Report.md
      Reviewer-Private-Notes.docx
      Reviewer-Private-Notes.md
      Evidence-Ledger.md
      Review-Archive.zip
```

## 7.7 Retention and deletion

**DATA-19** The user owns their data and a full purge must leave nothing behind. `DELETE /api/reviews/{id}` runs a single guarded transaction. It creates the temporary table `temp._mara_purge`, deletes the `reviews` row (which cascades to every child table, including the append-only tables because the delete trigger's guard is now satisfied), then drops the temporary table. The transaction commits only if every dependent row is gone.

**DATA-20** After the database transaction commits, the purge removes `blobs/<review_id>/` and `deliverables/<review_id>/` from disk. The purge is idempotent, so a re-run after a partial failure completes cleanly and leaves no orphaned rows or files.

**DATA-21** A verification query, run by the purge routine before it reports success, confirms that no row in any table references the deleted review identifier and that neither on-disk directory remains.

## 7.8 Postgres profile delta

**DATA-22** The optional Postgres profile keeps the same logical schema and the same single-writer and append-only rules. Type mappings change: `TEXT` stays `TEXT`, `INTEGER` becomes `INTEGER` or `BIGINT` for counters, `REAL` becomes `DOUBLE PRECISION`, `BLOB` becomes `BYTEA`, boolean integers become `BOOLEAN`, timestamp text becomes `TIMESTAMPTZ`, and JSON text columns become `JSONB`.

**DATA-23** The append-only triggers become `BEFORE UPDATE` and `BEFORE DELETE` PL/pgSQL functions that `RAISE EXCEPTION`. The purge guard uses a transaction-local setting, `SET LOCAL mara.purge = 'on'`, which the delete trigger reads through `current_setting('mara.purge', true)`, replacing the temporary-table guard used in SQLite.

**DATA-24** Drizzle emits the correct dialect for the active profile from one schema definition. The profile is selected at startup by the `MARA_DB` connection string, and the application refuses to run pipeline writes from any connection other than the worker's, regardless of profile.
