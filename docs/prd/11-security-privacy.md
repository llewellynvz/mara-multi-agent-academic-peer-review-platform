# 11. Security and privacy

Product Requirements Document. MARA (Multi-Agent Academic Peer-Review Architecture).
Psynalytics. Author: Prof. Llewellyn van Zyl, PhD. Status: draft for build. Language: British English.

MARA handles other people's unpublished manuscripts. The security posture treats every submitted document as both confidential and potentially hostile. A manuscript can carry hidden instructions that try to steer the agents, and it can carry text that must never leave the machine. This section sets out the threat model, the injection quarantine, the confidentiality egress control, key management, transport, the privacy statement the product must show, the supply-chain policy, and the three security test suites that must pass before release. Requirements carry the `SEC-` prefix.

## 11.1 Threat model

| Threat | Vector | Mitigation |
| --- | --- | --- |
| Document-borne prompt injection | Hidden text, white-on-white layers, embedded instructions, injection marker phrases in the manuscript | Three-tier quarantine (11.2): deterministic scan, cheap-tier LLM screen, runtime defence. Tier 3 halts the run |
| API key theft | Keys read from the database file, from logs, from a memory dump, or from an API response | AES-256-GCM envelope encryption (11.4), master key from environment or OS keystore, optional session-only mode, keys never logged, keys masked in every API response |
| Manuscript exfiltration through queries | An agent crafts a web query that carries manuscript sentences or an author's identity out to a public API | Egress control (11.3): no free-text queries, allowlist query builder, signed queries, n-gram check against the protected corpus, published references only, single egress client, every query logged |
| Malicious document parser exploitation | A crafted PDF or DOCX triggers a parser bug during ingestion | Parsing runs in the worker with time and memory limits, on pinned parser versions, in a scratch directory, with embedded scripts and active content stripped and never executed (11.5) |
| Local network exposure | The service is reachable from the local network or the internet without the owner intending it | Bind to `127.0.0.1` by default, instance passphrase option, explicit and warned opt-in before binding to a routable address, HTTPS guidance for any LAN exposure (11.6) |

## 11.2 Injection quarantine

**SEC-01** Every manuscript passes through a three-tier quarantine at Phase 0 before any reviewing agent sees it. No downstream agent receives the document until the quarantine has assigned a tier and recorded its log in `manuscripts.quarantine_tier` and `manuscripts.quarantine_log_json`.

**SEC-02** Tier 1 is a deterministic scan that runs without a model. It detects hidden or non-rendering text, differences between the visible text layer and the extracted text layer of a PDF, off-canvas or zero-opacity content, unusually small or white-on-white text, and known injection marker heuristics such as instruction-shaped phrases addressed to a model. The scan produces a structured list of hits with locations.

**SEC-03** Tier 2 is a cheap-tier LLM screen. It receives the deterministic scan output and a delimited copy of the suspect passages and returns a structured verdict `{ tier: 1 | 2 | 3, reasons: string[] }`. The screen runs on the cheapest configured model and never receives tool access.

**SEC-04** Tier 3 halts the run. When either the deterministic scan or the LLM screen assigns tier 3, the pipeline stops, records the reason in `review_events`, sets the review status to `failed` with an `error_class` of `quarantine_tier_3`, and surfaces the finding to the user. The manuscript is not reviewed until the user resolves it.

**SEC-05** Runtime defence applies for tier 1 and tier 2 documents that proceed. The manuscript is always presented to agents inside a delimited untrusted block that is labelled as data, never as instructions. Agents receive no irreversible tools during a review. The adversarial critic at the release gate acts as an output gate that inspects the deliverables for signs the manuscript steered the review.

**SEC-06** The three-tier decision, its inputs, and its output verdict are covered by unit tests that assert a clean document reaches tier 1, a document with hidden instruction text is raised to at least tier 2, and a document with an explicit model-directed injection is raised to tier 3 and halts the run.

## 11.3 Confidentiality egress control

The egress control exists so that no part of a confidential manuscript can leave the machine through a citation lookup. It is the single most important privacy mechanism in the product.

**SEC-07** Agents cannot issue free-text web queries. There is no code path from an agent to an arbitrary outbound HTTP request. The only outbound requests permitted for citation work are built by the allowlist query builder and sent by the single egress client.

**SEC-08** The allowlist query builder accepts only two shapes of input: a parsed citation-shaped reference string or DOI, and a controlled field-vocabulary term drawn from a fixed vocabulary held in the application. It rejects any input that is free text. Each accepted query is signed with an application key, and the signature travels with the query to the egress client.

**SEC-09** A single egress HTTP client is the only component that makes outbound citation requests. It is also the shared rate-limited, cached, exponential-backoff client for the three citation sources: Crossref through the polite pool, OpenAlex through its polite pool with an optional key, and Semantic Scholar with a key. It rejects any query that does not carry a valid signature from the query builder.

**SEC-10** Before it sends a candidate query, the egress client runs an n-gram check against the protected corpus, defined as the manuscript body with the reference section removed. If any n-gram of the candidate query matches the protected corpus above the configured threshold, the query is blocked and the attempt is logged. Reference strings pass this check because references are excluded from the protected corpus by definition.

**SEC-11** Only published references may leave the machine. A reference is eligible for egress only if it resolves to a stable identifier: a DOI, an ISBN, or a stable venue and year. Any reference marked in preparation, under review, unpublished, or submitted, or lacking a resolvable identifier, is excluded from egress. The reason is that an unpublished or in-preparation self-citation would leak the author's identity and the framing of unpublished results.

**SEC-12** Every outbound query is logged to `review_events` with `kind = "web_query"`, the `egress_target`, and the exact `egress_query` string that was sent. The log is part of the append-only timeline, so the full set of what left the machine during a review is auditable after the fact.

## 11.4 Key management

**SEC-13** Provider API keys are protected with AES-256-GCM envelope encryption. Each key is encrypted with a per-key data encryption key. The data encryption key is itself wrapped by a master key. The wrapped data key, the ciphertext, and the two initialisation vectors and authentication tags are stored in `provider_keys` (Section 7). The plaintext key never touches the database.

**SEC-14** The master key is supplied from the environment or from the operating system keystore. It is never written to the database, never written to a log, and never returned by any API endpoint.

**SEC-15** A session-only mode is available. In session-only mode, keys live in worker memory for the life of the process and are never persisted. No `provider_keys` row is written, and a restart requires re-entry of the keys. The mode is selected per key at creation time through the `persist` field (Section 8, API-18).

**SEC-16** Keys are masked in every API response, revealing at most the last four characters. Keys are excluded from all logs at every level. A key value never appears in an error message, a stack trace, or a telemetry payload.

**SEC-17** A key vault round-trip unit test encrypts a known key, stores it, reads it back, decrypts it, and asserts equality with the original, then asserts that the stored ciphertext does not contain the plaintext and that a wrong master key fails authentication rather than returning corrupt plaintext.

## 11.5 Document parsing safety

**SEC-18** Manuscript parsing runs inside the worker process with a wall-clock timeout and a memory ceiling, on pinned parser library versions. A document that exceeds either limit is rejected with a clear error rather than being allowed to exhaust the host.

**SEC-19** Parsing writes only to a per-review scratch directory. Embedded scripts, macros, and active content in the source document are stripped during extraction and are never executed. The parser is not permitted to shell out to external binaries on untrusted input.

## 11.6 Transport

**SEC-20** The service binds to `127.0.0.1` by default and is reachable only from the local machine. Binding to a routable address requires an explicit configuration change, and the application prints a clear warning at startup describing the exposure when a non-loopback bind is configured.

**SEC-21** When the owner chooses to expose the service on a local network, the documentation instructs them to place it behind an HTTPS terminating reverse proxy and to set an instance passphrase. The product does not offer plain HTTP on a routable address as a supported configuration.

**SEC-22** The instance passphrase, when set, gates every route except health and session (Section 8, API-02). The passphrase is stored only as a salted hash in `settings`, never in plaintext.

## 11.7 Privacy statement

**SEC-23** The product presents a plain privacy statement at setup and keeps it available in the interface. The statement makes the following commitments, each of which the architecture enforces.

**SEC-24** A manuscript never leaves the machine except to the language-model provider the owner has chosen, and to the three citation APIs under the egress rules in 11.3. There is no other outbound path for manuscript content.

**SEC-25** The statement points the owner to each configured provider's data-retention policy, because content sent to a third-party model is governed by that provider's terms. The product does not claim control over how a provider handles a request once it is sent.

**SEC-26** The statement declares that MARA is developmental pre-submission support authored to help an author improve a manuscript before submission. It is not a journal decision, not an editorial acceptance, and not a substitute for formal peer review. This disclaimer appears in the reviewer's private notes and in the interface.

## 11.8 Dependency and supply-chain policy

**SEC-27** Every production dependency is pinned to an exact version and the lockfile is committed. Builds are reproducible from the lockfile. A floating or range version is not permitted for a production dependency.

**SEC-28** Dependency updates are reviewed against release advisories before they are adopted. A dependency with an open advisory relevant to the way the product uses it is not adopted until the advisory is resolved or a mitigation is in place.

**SEC-29** A dependency audit runs in continuous integration and fails the build on a known high-severity advisory in the resolved tree. The parser libraries used for manuscript ingestion are treated as high-sensitivity dependencies and are held to the tightest version discipline.

## 11.9 Required security test suites

**SEC-30** Three security test suites must pass before any release.

The egress guard suite asserts that a query built from a published reference passes, that a query carrying manuscript body text is blocked by the n-gram check, and that a reference marked in preparation is excluded from egress. It also asserts that an unsigned query is rejected by the egress client.

The key vault suite performs the round-trip described in SEC-17 and asserts that plaintext never appears in the stored ciphertext and that a wrong master key fails closed.

The ledger append-only suite asserts that a direct UPDATE and a direct DELETE against `findings` both raise (Section 7, DATA-13), and that a correction is expressible only as a new superseding row.

**SEC-31** These three suites are release-blocking. A failure in any one of them stops the release, and the failure is reported as a headline result, not as a footnote below a passing summary.
