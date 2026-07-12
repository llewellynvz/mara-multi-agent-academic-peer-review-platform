# MARA: Multi-Agent Academic Peer-Review Architecture

## Product requirements document

| Field | Value |
|---|---|
| Product | Multi-Agent Academic Peer-Review Architecture (MARA) |
| Document | Product requirements document, v1.0 draft |
| Owner | Prof. Llewellyn van Zyl, Psynalytics |
| Status | Draft for review |
| Date | 12 July 2026 |
| Canonical annexes | `docs/prd/05` to `docs/prd/13` (pipeline, agents, data, API, UI, brand, security, observability) |

---

## Executive summary

MARA turns a proven but slow multi-agent peer-review methodology into a self-hosted web application that any researcher can run on their own machine with their own model API key. A researcher uploads a manuscript, confirms a short set of informed questions, and receives a verified developmental peer review in about 40 minutes on the happy path and under 50 minutes through one release-gate cycle: a summary verdict card, a full evidence-grounded report rendered in the browser, and a branded Word document for download.

The review methodology is not new. It is the MARA v3.0 architecture already in production use as an orchestrated reviewing pipeline: nine phases, eleven specialist reviewing lenses, six integrity screens, a simulated reviewer population that stress-tests findings, and an adversarial release gate that blocks any report whose claims do not trace to evidence. What changes is the delivery. The pipeline moves from a conversational agent environment that takes around three hours per review into a deterministic TypeScript workflow engine that runs the same depth in roughly a fifth of the wall-clock time, wrapped in a front end built on the Psynalytics design system.

The product ships as source-available software. Researchers may use it as-is for non-commercial purposes. Modification and redistribution of derivatives are prohibited, and commercial use requires permission. The purpose is public benefit with protected authorship: help researchers strengthen manuscripts before submission, without surrendering the methodology.

---

## 1. Vision and positioning

### 1.1 The problem

Peer review is the quality mechanism of science, and it is scarce. Wait times run months. Reviews vary enormously in depth, tone, and usefulness. Most researchers, especially early-career researchers and those outside well-resourced institutions, have no way to obtain a rigorous, structured critique of their manuscript before they submit it. They submit blind, wait half a year, and receive feedback that is often thin, sometimes destructive, and rarely developmental.

Large language models make a new kind of pre-submission review possible, but a single model prompted to "review this paper" produces exactly the failure modes editors already know: confident, ungrounded, generic commentary that either flatters or savages the work. A trustworthy automated review needs the same machinery human editorial processes use: independent perspectives, evidence discipline, integrity screening kept separate from misconduct accusation, calibrated recommendations, and a gate that stops bad reports from shipping.

### 1.2 The product

MARA is a self-hosted web application that runs a complete editorial review process on one manuscript at a time. It is not a chatbot and not a writing assistant. It is a pipeline with fixed phases, defined gates, and an audit trail:

1. The manuscript is screened for embedded prompt-injection content before any reviewing agent sees it.
2. Analysis agents build a structured map of the manuscript: its claims, evidence, design, figures, and declared metadata.
3. Field-context and citation-audit agents establish benchmarks and verify that cited references exist and support the sentences that cite them.
4. Between six and eleven specialist reviewing lenses run in parallel, blind to each other, then challenge each other in a second round where positions change only on evidence.
5. Six integrity screens produce editorial signals, never verdicts.
6. A simulated reviewer population stress-tests which findings are consensus-robust and which are fragile.
7. A meta-reviewer synthesises everything into a 15-criterion rubric and a recommendation.
8. An adversarial critic attacks the draft report and blocks release until every claim is grounded and the tone is developmental.
9. The system produces the deliverables: summary card, full report, and branded Word documents.

Every substantive claim in the output traces to a numbered finding in an append-only evidence ledger, anchored to a location in the manuscript. That traceability is the product's defining property.

### 1.3 Positioning

MARA is positioned as developmental pre-submission review: a rigorous third reviewer that sits alongside the author, not a journal decision engine. The recommendation categories mirror editorial practice (accept through reject) because researchers need honest calibration, but the report's voice is developmental by requirement. A destructive report fails the release gate no matter how correct its findings.

Three boundaries define the positioning:

- **Not a substitute for human peer review.** The disclaimer ships in the interface, the README, and every generated document.
- **Not an integrity tribunal.** Similarity, AI-content, figure, and consistency screens produce signals for the author's own attention, in signal language, never accusations.
- **Not a cloud service.** The manuscript never leaves the researcher's machine except to the model provider they chose and, under strict egress rules, to three public citation registries. Psynalytics hosts nothing and sees nothing.

### 1.4 Why now

The reviewing methodology exists and has produced complete reviews in production use. The constraint has been delivery: a three-hour conversational pipeline run by hand. Deterministic workflow engines, durable execution, structured model outputs, and multi-provider abstractions matured across 2025 and 2026 to the point where the same methodology runs as ordinary software. The cost side also resolved: model tiering and prompt caching bring a full-depth review into a price range an individual researcher will pay for their own manuscript.

### 1.5 Goals and non-goals

**Goals**

- G1. A researcher with a laptop, Docker, and an API key completes their first review within 30 minutes of cloning the repository.
- G2. A full-depth review of a standard empirical manuscript completes in roughly 38 to 48 minutes through one release-gate cycle on a hosted frontier provider.
- G3. Every released report passes the grounding validator: 100 percent of substantive claims resolve to ledger finding IDs with manuscript anchors.
- G4. The full review methodology of MARA v3.0 is preserved, including the capabilities the interim system dropped: the seven-agent swarm design, the quality-metrics composite, the recommendation engine bands, and the reporting-guideline selector.
- G5. The interface and every generated document carry the Psynalytics design system.

**Non-goals**

- N1. Multi-tenant hosting, accounts, or billing. Single researcher first, lab instance later.
- N2. Journal-side workflow integration (editorial manager plugins, reviewer matching).
- N3. Manuscript writing or revision assistance. MARA reviews, it does not rewrite.
- N4. Real-time collaboration.

---

## 2. Personas and use cases

### 2.1 Primary persona: the self-reviewing researcher

A PhD candidate, postdoc, or faculty member in psychology, wellbeing science, or adjacent social sciences. Writes empirical papers, quantitative or mixed methods. Comfortable installing software from a README but not a developer. Wants a brutal-but-fair read of the manuscript before spending a submission cycle on it. Owns an API key or is willing to create one. Cares about confidentiality: unpublished results must not leak into anyone's training data or logs.

Needs: honest severity calibration, specific fixes rather than vague complaints, a document they can work through point by point, and confidence that the critique is grounded in the actual manuscript rather than generic reviewer priors.

### 2.2 Secondary persona: the supervisor

A professor reviewing student drafts at volume. Uses MARA as a first-pass filter so supervision time goes to the judgement calls rather than the mechanical problems. Values the rubric scores and bottleneck analysis for tracking a manuscript across revisions. Runs the same install, higher throughput.

### 2.3 Tertiary persona (v2): the lab or department instance

A methods-support unit or graduate school running a shared instance on a lab server. Needs multi-user separation and usage visibility. Explicitly deferred to v2.

### 2.4 Use cases

| ID | Use case | Persona | Frequency |
|---|---|---|---|
| UC-1 | Pre-submission review of own manuscript | Researcher | Per paper, 1 to 3 runs |
| UC-2 | Re-review after major revision | Researcher | Per paper |
| UC-3 | First-pass screen of a student draft | Supervisor | Weekly |
| UC-4 | Calibrating a revise-and-resubmit response against the report | Researcher | Per decision letter |
| UC-5 | Teaching: showing students what a structured review looks like | Supervisor | Per term |

---

## 3. User stories and journey

### 3.1 Core journey

1. **Install.** Clone the repository, copy the environment template, set a master key and one provider API key, run `docker compose up -d`, open the local address.
2. **Set up.** A wizard verifies the API key live, offers Fast, Balanced, or Thorough presets with cost estimates, and records defaults. Telemetry is presented once, off by default.
3. **Upload.** Drag a PDF or DOCX onto the new-review screen. Parsing starts immediately and detected metadata appears alongside.
4. **Confirm.** After a short lite parse, the clarifying-questions screen shows what the system detected (field, study design, manuscript type) as confirm-or-correct chips, and asks four questions: target journal, review depth, optional feedback focus, and anything the reviewer should know. All skippable with defaults.
5. **Watch or leave.** The run progresses through a nine-phase timeline with live lens status, a findings ticker, a cost meter, and an honest ETA. The run survives a closed tab and a machine that goes to sleep on the server side. A desktop notification fires on completion.
6. **Read.** The results screen leads with the summary card: recommendation, confidence band, rubric average, three bottleneck criteria, and the top five imperative changes. The full report renders beneath it. Editor-register signals sit in a second tab.
7. **Download.** One button produces the branded Word report. An overflow menu offers the private-notes document, markdown, the evidence ledger, and a complete archive.

### 3.2 User stories (selected, testable)

- US-1. As a researcher, I can complete first-run setup in under 3 minutes with one API key, so that trying MARA costs me almost nothing. Done when: wizard completes with a verified key and defaults persisted in under 3 minutes in a timed test.
- US-2. As a researcher, I can upload a 9,000-word PDF manuscript and reach the clarifying questions in under 2 minutes. Done when: lite parse median under 2 minutes on the reference machine.
- US-3. As a researcher, I can skip every clarifying question and still get a valid full review. Done when: a run started with "use detected values" completes and passes the release gate.
- US-4. As a researcher, I can close the browser during a run and return later to a completed review. Done when: a run with the tab closed at phase 3 completes and appears in the library.
- US-5. As a researcher, I can see why any claim in the report was made. Done when: every finding ID in the rendered report opens the evidence drawer showing anchor, evidence, and confidence.
- US-6. As a researcher, I receive severity-honest but developmental feedback. Done when: the release-gate tone audit passes and the banned destructive-phrasing scan reports zero hits on the released report.
- US-7. As a researcher, I can retry a failed phase without restarting the run. Done when: a forced provider failure at phase 4 recovers from checkpoint with no duplicate ledger rows.
- US-8. As a supervisor, I can run three reviews in sequence without touching configuration. Done when: three consecutive runs complete with defaults from the library screen.
- US-9. As a privacy-conscious researcher, I can run in session-key mode where my API key is never written to disk. Done when: a run completes in session-key mode and the key vault table holds no row for the session key.
- US-10. As a researcher on a slow local model, I get an honest time expectation instead of a broken promise. Done when: selecting an Ollama provider switches the ETA to local-hardware mode with the depth-capped preset applied.

### 3.3 Journey-level requirements

- JRN-01. The time from `docker compose up -d` to first completed review on the reference machine must not exceed 75 minutes, of which setup is under 10.
- JRN-02. Every screen in the core journey must carry inline guidance sufficient to proceed. Verified by a scripted first-run walkthrough that completes the journey without opening documentation.
- JRN-03. Every dead end (parse failure, tier-3 quarantine, gate block, provider outage) must present a recovery path or a plain-language halt explanation on screen.

---

## 4. Functional requirements by module

Requirement language: "must" is binding for v1.0. "Should" is binding unless a documented constraint prevents it. IDs are stable and cited by tests.

### 4.1 Ingestion and parsing

- FR-ING-01. The system must accept manuscript uploads in PDF and DOCX up to 50 MB.
- FR-ING-02. PDF parsing must use the GROBID sidecar when reachable and fall back to pure-Node extraction with a visible parse-quality warning when not.
- FR-ING-03. DOCX parsing must preserve semantic structure (headings, tables, captions) via style mapping.
- FR-ING-04. Parsing must produce the structured manuscript object defined in annex 5: section map, metadata, figure and table inventory, and reference list.
- FR-ING-05. The user must confirm or correct the detected section map before frontier-tier spend begins.
- FR-ING-06. Supplementary files (appendices, data dictionaries) must be attachable to the same review and enter the same sanitization path.

### 4.2 Sanitization and clarifying questions

- FR-SAN-01. The sanitizer must run before any other agent reads manuscript content, applying the three-tier quarantine defined in annex 11.
- FR-SAN-02. Tier-3 findings must halt the run with a plain-language explanation and no partial review output.
- FR-CLQ-01. Clarifying questions must fire only after the lite parse, so every question is informed by detected values.
- FR-CLQ-02. The question set is fixed at two blocks: confirm-or-correct (field, design, type) and four choices (journal, depth, focus weighting, reviewer note). No free-form mid-run questions.
- FR-CLQ-03. Answers must write into the run manifest consumed by lens activation and the field-context agent. Focus answers adjust weighting only and must never deactivate a lens.
- FR-CLQ-04. A skip control must start the run with detected values and defaults in one click.

### 4.3 Orchestration

- FR-ORC-01. The pipeline must execute as a deterministic code-controlled workflow. No model output may decide routing, merge the ledger, or count gate cycles.
- FR-ORC-02. The workflow must checkpoint at every phase boundary and resume from the last checkpoint after a process restart.
- FR-ORC-03. Phase overlap must follow the schedule in annex 5 (context and integrity concurrent with the specialist first pass, context injected at the challenge round).
- FR-ORC-04. The release gate must implement pass, revise, revise-specialist, and block verdicts with a hard cycle cap of 2, exiting to logged deterministic arbitration.
- FR-ORC-05. A failed dispatch must retry once with the defect named in the retry prompt, then halt the phase with options (retry, skip if optional, cancel).
- FR-ORC-06. Optional pause-at-gates mode must suspend the run at each gate for user confirmation.

### 4.4 Provider layer

- FR-PRV-01. The system must support Anthropic, OpenAI, and Google providers plus any OpenAI-compatible base URL (Ollama and equivalents) at v1.0.
- FR-PRV-02. Model tiering must be configurable per phase group with sane defaults (frontier for lenses, synthesis, and the gate, cheap tier for mechanical passes).
- FR-PRV-03. All agent outputs must be schema-validated structured objects with automatic repair retries on validation failure.
- FR-PRV-04. Prompt caching must be applied wherever the active provider supports it, with cached-token share recorded per dispatch.
- FR-PRV-05. Per-provider concurrency limits and rate-limit backoff must be enforced centrally.
- FR-PRV-06. A pre-run estimate must show expected cost range and duration for the chosen preset and provider, and the run must display actual spend live.

### 4.5 Review outputs

- FR-OUT-01. The released deliverables are: summary card data, the seven-part peer-review report (markdown), the reviewer's private notes (markdown), the branded Word versions of both, and the evidence ledger export.
- FR-OUT-02. No deliverable may be marked released before a critic pass verdict, enforced at the database layer.
- FR-OUT-03. The Word documents must follow the brand respec in annex 10 and open without warnings in Microsoft Word and LibreOffice.
- FR-OUT-04. Editor-register content must never appear in the author-facing report, enforced by the scope field on every finding and verified at the gate.
- FR-OUT-05. A complete archive export (report, notes, ledger, run audit) must be downloadable as one zip.

### 4.6 Library and run management

- FR-LIB-01. The library must list all reviews with status, recommendation, rubric average, and date, and reconnect to any running review's live view.
- FR-LIB-02. Deleting a review must purge its manuscript, findings, deliverables, and blobs completely.
- FR-LIB-03. Re-running a review must create a new run keyed to the same manuscript record, never overwrite a prior run.

### 4.7 Settings

- FR-SET-01. Settings must cover providers and keys, per-phase-group tiers, default preset, data location, telemetry, and a danger zone (purge all data).
- FR-SET-02. Key storage must follow annex 11 (AES-256-GCM envelope encryption, optional session-only mode).
- FR-SET-03. Changing settings must never affect an in-flight run.

---

## 5. Pipeline specification

Full specification in `docs/prd/05-pipeline-spec.md`. Normative summary: nine phases (0 to 8) preserved from MARA v3.0 with a re-ordered execution schedule that runs field context, citation audit, and integrity screening concurrently with the specialist first pass, injects context at the challenge round, and parallelises the phase-7 synthesis inputs. Dispatch budget roughly 40 to 45 model calls per run, of which 20 to 22 on the frontier tier. Runtime 38 to 40 minutes happy path, 45 to 48 with one gate cycle, 53 to 56 in the rare two-cycle case, always disclosed by the live ETA. Preconditions: hosted frontier provider with prompt caching. Local providers receive their own ETA model and a depth-capped default preset.

## 6. Agent and knowledge specification

Full specification in `docs/prd/06-agent-knowledge-spec.md`. Normative summary: the ten current agents map to workflow nodes with typed output schemas. The knowledge modules 01 to 06 carry into `/knowledge` verbatim as the behavioural contract, with the ownership map preserved. Four MARA v3.0 capabilities are restored: the seven-agent swarm design with Shannon consensus entropy and dissent preservation, the quality-metrics composite (30/25/20/15/10 weights), the recommendation engine's explicit rubric-average bands, and the reporting-guideline selector. Every one of the twelve invariants maps to a named enforcement mechanism and a verification test.

## 7. Data model and storage

Full specification in `docs/prd/07-data-model.md`. Normative summary: SQLite via Drizzle as default with an optional Postgres profile. The findings table is an append-only event table enforced by triggers, written only by the worker process at phase boundaries. Checkpoints and a review-events timeline replace STATE.md and drive both resume and the progress interface. Deliverables carry a released flag settable only on a critic pass.

## 8. API surface

Full specification in `docs/prd/08-api-surface.md`. Normative summary: a local REST API for reviews, uploads, answers, run control, deliverables, settings, and stats, plus one SSE stream per run carrying phase status, lens status, finding headlines, cost ticks, ETA updates, and gate verdicts, with Last-Event-ID reconnect.

## 9. Interface specification

Full specification in `docs/prd/09-ui-spec.md`. Normative summary: seven screens (setup, library, new review, clarifying questions, run progress, results, settings) built on the Psynalytics website kit plus a product-app kit extension (timeline, tabs, table, toast, dropzone, meters, log stream, drawer, popover) specified on the same tokens, with two new desaturated warn and fail tokens. WCAG 2.2 AA.

## 10. Brand and document design

Full specification in `docs/prd/10-brand-spec.md`. Normative summary: the psynalytics-brand skill v3 tokens are the source of truth. Both brand fonts self-hosted (Inter and JetBrains Mono, both OFL), no external font imports. The Word documents move to Inter-based typography with the previous Gadugi and Calibri Light demoted to named fallbacks, never embedded. Teal house identity retained.

## 11. Security and privacy

Full specification in `docs/prd/11-security-privacy.md`. Normative summary: three-tier injection quarantine ahead of all agents. Confidentiality enforced at a single egress client: only signed, citation-shaped or controlled-vocabulary queries leave the machine, unpublished and identifier-less references are excluded, and every outbound query is logged. BYO keys under AES-256-GCM envelope encryption with an optional never-persisted session mode. Three mandatory security test suites.

## 12. Non-functional requirements

- NFR-01 (runtime). Median full review on the Balanced preset with a hosted frontier provider: 48 minutes or less through one gate cycle. Two-cycle runs may reach 56 minutes and must be disclosed by the live ETA as soon as cycle 2 begins.
- NFR-02 (cost). The Balanced preset must complete a 9,000-word manuscript for 12 US dollars or less in model spend at July 2026 frontier pricing, with the pre-run estimate accurate within 40 percent.
- NFR-03 (resumability). Any single process crash must lose at most one phase of work. Resume must not duplicate ledger rows or dispatch spend for completed phases.
- NFR-04 (integrity of outputs). The grounding validator must pass on 100 percent of released reports. This is a release condition, not a quality target.
- NFR-05 (portability). Supported hosts: Windows 11, macOS 14+, Ubuntu 22.04+, each via Docker Compose. The reference machine for all timing targets: 4 vCPU, 16 GB RAM, SSD.
- NFR-06 (concurrency). v1.0 supports one active run per instance. Queued runs start automatically in order.
- NFR-07 (availability of externals). Citation registries degrade gracefully: if Crossref, OpenAlex, and Semantic Scholar are all unreachable, the citation audit reports not-run status rather than fabricating verification.
- NFR-08 (data locality). All review data lives under one user-chosen directory, movable and back-up-able as a unit.

## 13. Observability

Full specification in `docs/prd/13-observability.md`. Normative summary: a local stats panel driven by the local run tables (cost, duration, completion and retry rates). Structured logs with no manuscript content above debug level. Optional Langfuse wiring, env-gated, mapping one session per run with the quality-metrics composite emitted as scores. Opt-in telemetry, off by default, carrying operational fields only.

## 14. Packaging and distribution

### 14.1 Repository layout

```
mara/
  app/          Next.js 16 front end
  server/       worker, workflow, providers, egress client
  knowledge/    modules 01-06, verbatim behavioural contract
  agents/       prompt definitions + output schemas
  docs/         PRD, install, configuration, FAQ
  docker-compose.yml
  .env.example
  LICENSE
  README.md
```

### 14.2 Install story

- PKG-01. The supported install is: clone, copy `.env.example` to `.env`, set the master key and one provider key, `docker compose up -d`. Nothing else.
- PKG-02. Compose profiles: default (single app container with SQLite volume), `grobid` (adds the parsing sidecar, recommended), `full` (adds Postgres and Langfuse for users who want them).
- PKG-03. The README leads with what the tool is in three bullets, a screenshot, the three-command quickstart, the licence banner, and the disclaimer: developmental pre-submission support, not affiliated with any journal, never a substitute for human review.
- PKG-04. Versioning is semver through GitHub releases. Any change to knowledge modules or pipeline behaviour bumps minor at minimum and is flagged "review behaviour changed" in the changelog.
- PKG-05. Issue templates must warn users never to attach manuscripts to public issues.

### 14.3 Distribution boundaries

- PKG-06. No hosted instance, no auto-update, no phone-home. Release advisories are the update channel.

## 15. Licensing

- LIC-01. The licence is PolyForm Noncommercial 1.0.0 as the base grant, extended with a clause prohibiting distribution of modified versions. Private local modification remains permitted, so self-hosters can apply local patches lawfully. Redistribution of the software, modified or not, outside the official repository is prohibited.
- LIC-02. All public language says "source-available", never "open source", because the licence does not meet the OSI definition. This is stated plainly in the README to set expectations.
- LIC-03. Commercial use requires written permission from Psynalytics. A contact route ships in the LICENSE file.
- LIC-04. Pull requests are closed by policy. CONTRIBUTING.md states it plainly: the licence prevents accepting code contributions, while bug reports, parsing-failure reports, and documentation suggestions are welcome and credited.
- LIC-05. Third-party components retain their own licences, listed in a NOTICE file. Both shipped fonts are OFL.

## 16. Roadmap

| Version | Scope |
|---|---|
| v1.0 | Full pipeline at near-full depth including the restored swarm, multi-provider (Anthropic, OpenAI, Google, OpenAI-compatible local), all seven screens, both Word deliverables, compose install, the three security test suites, grounding validator as release condition |
| v1.1 | npx single-command installer, desktop notifications hardened cross-platform, mid-phase pause and resume, findings-ticker refinements, cost-estimate learning from local history |
| v2.0 | Revision re-review (diff a resubmission against the prior run's findings and report which concerns were resolved), lab multi-user instance, programmatic API for batch use, journal-guideline import |

Revision re-review is the standout later feature: the append-only ledger and stable finding IDs exist in v1.0 precisely so that v2.0 can diff runs.

## 17. Success metrics

- MET-01. North star: completed reviews per install per month, measured only in the opt-in telemetry cohort.
- MET-02. Local stats shown to every user as a feature: completion rate, retry rate, median duration, median cost.
- MET-03. Quality proxy: a one-tap post-review usefulness rating stored locally, shared only under opt-in.
- MET-04. Public proxies tracked by the maintainer: stars, release downloads, unique cloners, issue engagement.
- MET-05. Release-quality bar: zero released reports failing the grounding validator, zero banned-verdict-term hits in author-facing output, across all verification runs.

## 18. Risks

| # | Risk | Mitigation |
|---|---|---|
| 1 | Runtime target missed under provider rate limits | Central concurrency limiter, timeout plus one retry with tier fallback, honest live ETA, depth presets |
| 2 | Prompt-cache misses on parallel fan-out | Shared-prefix prompt structure, cache-warming pilot call, dispatch batching within cache TTL, cached-share monitoring |
| 3 | Structured outputs degrade nuanced review prose | Hybrid envelopes (markdown body inside typed shell), schema-repair retries, golden-set checks on gate schemas |
| 4 | GROBID misparses non-standard manuscripts | DOCX path, pure-Node fallback with warning, mandatory parse confirmation before frontier spend |
| 5 | Workflow-framework dependency under a no-derivatives licence | Pinned versions in the release image, thin workflow-adapter interface allowing engine swap without rewriting step logic |
| 6 | Security-patch latency | Licence permits private local patching, maintainer ships pinned-dependency releases on a stated cadence with advisories |
| 7 | Cost surprise for users | Pre-run estimate, live meter, per-run ceiling setting that pauses the run at the cap |

## 19. Open questions

- OQ-1. Reference-machine ETA medians must be measured during build. Bundled defaults ship from the maintainer's verification runs.
- OQ-2. Whether the lite parse (pre-questions) uses the cheap tier of the user's provider or a fixed minimal local pass. Decide in slice (a) of the build by measuring quality difference on the fixture set.
- OQ-3. The exact wording of the no-derivatives licence clause needs legal review before publication (not before build).
- OQ-4. Whether Semantic Scholar remains in the default citation-verification chain or becomes optional, given its 1 request-per-second authenticated limit. Decide in slice (e) from measured audit durations.
