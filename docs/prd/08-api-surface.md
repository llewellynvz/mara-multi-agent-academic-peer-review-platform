# 8. API surface

Product Requirements Document. MARA (Multi-Agent Academic Peer-Review Architecture).
Psynalytics. Author: Prof. Llewellyn van Zyl, PhD. Status: draft for build. Language: British English.

This section defines the local HTTP surface: the REST endpoints the Next.js user interface calls, the Server-Sent Events stream that carries live run progress, the error model, and the idempotency and reconnection rules. The surface is local first. It binds to loopback by default (Section 11) and requires no account. An optional instance passphrase gates every route. Requirements carry the `API-` prefix.

## 8.1 Conventions

**API-01** All endpoints live under `/api` and are served by the Next.js application on `127.0.0.1` by default. Request and response bodies are JSON with `Content-Type: application/json`, except manuscript upload (multipart) and deliverable download (binary or text).

**API-02** When an instance passphrase is set, every route except `GET /api/health` and `POST /api/session` requires an `Authorization: Bearer <token>` header. A missing or expired token returns `401`. When no passphrase is set, routes are open on loopback only.

**API-03** Timestamps in payloads use the same ISO-8601 UTC form as the data model. Identifiers are the UUIDv7 and namespaced strings defined in Section 7.

## 8.2 Session and health

**API-04** `POST /api/session` accepts `{ passphrase: string }` and returns `{ token: string, expiresAt: string }` on success, or `401` on mismatch. The token is a signed, expiring bearer token held only in the browser.

```ts
// POST /api/session
type SessionRequest = { passphrase: string };
type SessionResponse = { token: string; expiresAt: string };
```

**API-05** `GET /api/health` returns `{ status: "ok", version: string, worker: "up" | "down", db: "ok" | "error" }` without authentication, so a local watcher can confirm the worker and database are reachable.

## 8.3 Reviews

**API-06** `POST /api/reviews` creates a review. Request `{ title?: string, providerProfile?: string, options?: ReviewOptions }`. Response is the created `Review`. The row is inserted by the API server (Section 7, DATA-15).

```ts
type ReviewOptions = { pauseAtGates?: boolean; lensOverrides?: string[] };
type Review = {
  id: string;
  slug: string;
  title: string | null;
  status: "created" | "sanitizing" | "running" | "paused"
        | "awaiting_input" | "completed" | "failed" | "cancelled";
  currentPhase: string | null;
  recommendation: string | null;
  recommendationConfidence: number | null;
  createdAt: string;
  updatedAt: string;
};
```

**API-07** `GET /api/reviews` returns `{ reviews: ReviewSummary[] }`, newest first, where each summary carries `id`, `slug`, `title`, `status`, `currentPhase`, `createdAt`, and a `findingsCount`.

**API-08** `GET /api/reviews/{id}` returns the full `Review` plus counts by severity and the phase checkpoint list. A missing identifier returns `404`.

**API-09** `POST /api/reviews/{id}/manuscript` accepts a single file as `multipart/form-data` under the field `file`. The server computes the sha256, stores the original blob (Section 7, DATA-18), inserts the `manuscripts` row, and returns `{ manuscriptId: string, sha256: string, byteSize: number, quarantine: "pending" }`. Uploading a second file to a review that already holds a manuscript returns `409`.

**API-10** `POST /api/reviews/{id}/answers` submits an answer to a clarifying question raised during a run. Request `{ questionId: string, answer: string }`. The server writes the answer to the review's `options_json` merge area and inserts a `run_commands` resume intent if the review was `awaiting_input`. Response `{ accepted: true }`.

**API-11** `DELETE /api/reviews/{id}` purges the review and every artefact (Section 7, DATA-19 through DATA-21). It returns `200` with `{ purged: true }` once the guarded transaction and the on-disk cleanup both complete. There is no soft delete.

## 8.4 Run control

Run control is expressed as intents. The API server inserts a row into `run_commands` and the worker acts on it, so these endpoints return the accepted intent, not a terminal run state.

**API-12** The run-control endpoints are:

| Method and path | Effect | Response |
| --- | --- | --- |
| `POST /api/reviews/{id}/run` | Start the pipeline | `{ accepted: true, command: "run" }` |
| `POST /api/reviews/{id}/pause` | Request a pause at the next safe point | `{ accepted: true, command: "pause" }` |
| `POST /api/reviews/{id}/resume` | Resume a paused run | `{ accepted: true, command: "resume" }` |
| `POST /api/reviews/{id}/cancel` | Cancel the run | `{ accepted: true, command: "cancel" }` |
| `POST /api/reviews/{id}/retry-phase` | Re-run one phase | `{ accepted: true, command: "retry_phase" }` |

**API-13** `POST /api/reviews/{id}/retry-phase` takes `{ phase: string }` and validates that the named phase has a checkpoint in `failed` or `completed` state. An unknown phase returns `422`.

**API-14** Run-control endpoints are idempotent against the current state. Calling `run` on a review that is already `running` returns `200` with `{ accepted: true, command: "run", noop: true }` and the current status, rather than starting a second run. Calling `pause` on a run that is not active returns the same shape with `noop: true`.

## 8.5 Deliverables

**API-15** `GET /api/reviews/{id}/deliverables` lists deliverables with `kind`, `released`, `byteSize`, and `checksum`. `GET /api/reviews/{id}/deliverables/{kind}` streams the file. If the deliverable's `released` value is `0`, the endpoint returns `409` with the error code `deliverable_not_released` and does not stream the file (Section 7, DATA-10).

## 8.6 Settings and keys

**API-16** `GET /api/settings` returns the non-secret settings. `PUT /api/settings` accepts a partial settings object and merges it. The instance passphrase hash and any secret material are never returned by `GET`.

**API-17** `GET /api/keys` lists stored provider keys as `{ id, provider, label, maskedKey, baseUrl, persist }`, where `maskedKey` reveals only the last four characters. Ciphertext, initialisation vectors, and wrapped data keys are never returned.

**API-18** `POST /api/keys` adds a key. Request `{ provider, label?, apiKey, baseUrl?, persist: "disk" | "session" }`. When `persist` is `disk`, the server encrypts and stores the key (Section 11). When `persist` is `session`, the key is held in worker memory only and no `provider_keys` row is written. `DELETE /api/keys/{id}` removes a stored key.

## 8.7 Statistics

**API-19** `GET /api/reviews/{id}/stats` returns the per-run figures derived from the `dispatches` and `phase_checkpoints` tables: total cost, cost per phase, token totals, retry rate, and time to first review. `GET /api/stats` returns the instance aggregate: cost per run, completion rate, retry rate, and time to first review across all runs (Section 13).

```ts
type RunStats = {
  costUsd: number;
  costByPhase: Record<string, number>;
  tokensIn: number;
  tokensOut: number;
  tokensCached: number;
  retryRate: number;
  timeToFirstReviewMs: number | null;
};
```

## 8.8 Error model

**API-20** Errors return a JSON body `{ error: { code: string, message: string, details?: unknown } }` with a matching HTTP status. The `code` is a stable machine string. The `message` is a short human sentence. The `details` field, when present, carries structured context such as the offending field.

```ts
type ApiError = { error: { code: string; message: string; details?: unknown } };
```

**API-21** The status-to-code mapping is: `400 bad_request`, `401 unauthorized`, `404 not_found`, `409 conflict` (for example `manuscript_already_exists`, `deliverable_not_released`), `422 unprocessable` (validation), `429 rate_limited`, and `500 internal`. Validation failures list each invalid field in `details`.

## 8.9 Server-Sent Events stream

**API-22** `GET /api/reviews/{id}/events` opens a `text/event-stream`. The stream is the live view of a run and is backed by the append-only `review_events` table, so it can be replayed after a disconnect. Each message carries an `id:` line equal to the event's `review_events.seq`, an `event:` name, and a `data:` JSON payload.

**API-23** The event catalogue is:

| Event name | Payload |
| --- | --- |
| `phase_status` | `{ phase: string, status: string, startedAt?: string }` |
| `lens_status` | `{ lens: string, status: string, findingsCount: number }` |
| `finding_headline` | `{ findingId: string, severity: string, scope: string, headline: string }` |
| `cost_tick` | `{ costUsdTotal: number, tokensIn: number, tokensOut: number, phase: string }` |
| `eta_update` | `{ etaSeconds: number, basis: string }` |
| `gate_verdict` | `{ phase: string, verdict: string, cycle: number }` |
| `run_complete` | `{ recommendation: string, deliverables: { kind: string }[], durationMs: number }` |
| `run_failed` | `{ errorClass: string, phase: string, message: string }` |
| `log_event` | `{ ts: string, message: string }` (sanitised one-line activity entries, never model output or manuscript text) |

**API-24** The server sends a comment heartbeat (`: ping`) every 15 seconds so proxies and the browser keep the connection open. Heartbeats carry no `id:` and do not advance the sequence.

**API-25** The `finding_headline` payload contains a short headline and the finding's scope. It never contains manuscript text or editor-only narrative. A client rendering an author-facing view filters on `scope`.

## 8.10 Idempotency and reconnection

**API-26** On reconnect, the client sends the `Last-Event-ID` header with the last `seq` it received. The server replays every `review_events` row for that review with `seq` greater than the supplied value, in order, then resumes the live tail. Because `review_events` is append-only and sequenced per review (Section 7), replay is exact and produces no duplicates or gaps.

**API-27** Create and upload endpoints accept an optional `Idempotency-Key` header. A repeated request with the same key and the same review returns the original result rather than creating a duplicate, which protects against a double-submitted form or a retried upload.

**API-28** A run that has already reached `completed`, `failed`, or `cancelled` returns a final `run_complete` or `run_failed` event immediately on connection, so a client that reconnects after the run finished still receives the terminal state without polling.
