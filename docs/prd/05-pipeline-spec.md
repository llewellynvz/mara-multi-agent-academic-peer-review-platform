# 5. Pipeline specification

This section specifies the review pipeline: the nine phases (0 to 8), the concurrency schedule that runs them, the dispatch and runtime budgets, the release-gate machinery, the human-in-the-loop points, and the per-phase failure handling. It is the operational contract for the durable workflow. Where a rule about scoring, voice, or evidence handling is named here, the owning knowledge module holds the canonical statement and this section defers to it. Pipeline order and gate mechanics are owned here and in the workflow definition.

## 5.1 Pipeline shape

The pipeline is a single Mastra v1 durable workflow, run inside a dedicated Node worker with libSQL/SQLite storage. The workflow is a directed graph of typed steps. Fan-out uses `.parallel()`, the fix-cycle loop uses `.dountil()` capped at two iterations, and human-in-the-loop pauses use `.suspend()` and `.resume()`. Every step reads and writes structured objects produced by Vercel AI SDK `generateObject` calls validated by Zod. There are no markdown fragment files and no free-text handoffs between steps.

**PIPE-01.** The orchestrator is deterministic code. It routes artifacts, merges the evidence ledger, counts gate cycles, and applies arbitration rules. It never runs an LLM turn to decide routing, to merge findings, or to select an agent. A test asserts that no orchestration step issues a model call.

**PIPE-02.** Every phase boundary persists a checkpoint to durable storage containing the phase identifier, the activation map, gate-cycle counters, per-lens re-dispatch counts, and challenge-round completion flags. On restart the workflow resumes from the last completed phase and re-pays no completed phase. A test kills the worker mid-run and asserts resume from the correct phase.

**PIPE-03.** The ledger is append-only and single-writer. Agents return finding objects, the orchestrator alone appends them to the ledger, and a correction is a new row that supersedes a prior Finding ID by reference. No step other than the orchestrator's merge writes to the ledger. A test asserts that a second concurrent writer is rejected.

**PIPE-04.** Every dispatch carries the shared prefix `[static system frame][sanitised manuscript]` so the hosted provider serves it from prompt cache. The manuscript-specific instruction and the phase artifacts follow the cached prefix. A test asserts the prefix bytes are identical across dispatches in one run.

## 5.2 Phase-by-phase specification

Each phase below states its purpose, inputs, outputs, dispatch group, model tier, parallelism, and the gate condition that must hold before the next phase starts. Model tier is either **frontier** (a hosted frontier provider with prompt caching) or **cheap** (a lower-cost tier for mechanical and ensemble work). Ledger prefixes follow the single-source scheme in knowledge/01 and knowledge/02.

### Phase 0: Intake and sanitisation

- **Purpose.** Screen the raw submission for hidden instructions and tampering before any other step reads it, and produce the sanitised copy every downstream step works from.
- **Inputs.** The manuscript package, the journal, reviewer instructions, and any deadline. The review project is created from the template and BRIEF is populated.
- **Outputs.** `sanitised-manuscript` (structured, with quarantined spans placeholdered in place), a quarantine log, and sanitiser findings under prefix `REV-SAN`.
- **Dispatch group.** One dispatch: manuscript-sanitiser (blocking).
- **Model tier.** Cheap.
- **Parallelism.** None. This is the one strictly serial gate at the head of the pipeline.
- **Gate condition.** A Tier 3 tampering verdict halts the run with an editor-only block note. A Tier 2 verdict proceeds on the sanitised copy with a quarantine flag recorded. Tier 1 items are logged and left in place. No downstream step reads the original.

**PIPE-05.** No phase after 0 may read the original manuscript. Every dispatch is given the sanitised copy only. A test asserts the original file handle is not passed to any step past Phase 0.

### Phase 1: Manuscript analysis

- **Purpose.** Convert the sanitised manuscript into the structured objects every specialist reads instead of re-parsing the paper, and decide which lenses activate.
- **Inputs.** The sanitised copy, the quarantine log, and journal requirements when supplied.
- **Outputs.** Mode A: manuscript map, metadata and declarations report, figure and table inventory. Mode B: claim-evidence matrix, study-design classification, reporting-guideline route, and the specialist activation map. Findings under `REV-MAP`.
- **Dispatch group.** manuscript-analyst mode A, then manuscript-analyst mode B.
- **Model tier.** Mode A cheap (structural extraction), mode B frontier (claim materiality, design classification, activation).
- **Parallelism.** Mode B overlaps the tail of mode A once mode A's map is available. Mode B does not start before the map exists.
- **Gate condition.** The activation map is recorded with a one-line rationale per active lens before Phase 3 dispatches.

### Phase 2: Domain context

- **Purpose.** Build the comparative backdrop and verify the reference apparatus, without letting background literature override the submitted evidence.
- **Inputs.** The sanitised copy, the claim-evidence matrix, and the journal scope material.
- **Outputs.** Retrieval packet, benchmark report, source-availability record (`REV-CTX`), and the citation-audit trail with per-citation verdicts (`REV-REF`).
- **Dispatch group.** field-context-scout and citation-auditor.
- **Model tier.** Cheap, both. The citation-auditor's verification is mechanical lookup against Crossref, OpenAlex, and Semantic Scholar. The scout's judgment is bounded to context mapping.
- **Parallelism.** The two run in parallel with each other, and both run concurrently with the Phase 3 first pass.
- **Gate condition.** A possible-fabrication outcome is routed as an editor-only integrity signal, never a verdict. Context and citation results are held for injection at the Phase 3 challenge round, not the first pass.

**PIPE-06.** Both retrieval steps are bound by the retrieval-confidentiality rule in knowledge/01. Manuscript prose, title, author identity, and unpublished result values never enter an outbound query. Cited published reference metadata is the only manuscript-derived material permitted in a query. A test scans the outbound query log against the manuscript text and fails on any overlap outside the reference allowlist.

### Phase 3: Specialist review

- **Purpose.** Run each activated lens over the manuscript as a domain expert would, first independently, then in an evidence-only challenge round.
- **Inputs.** First pass: the manuscript map, the claim-evidence matrix, the lens rubric, and the current ledger. Challenge round: the anonymised findings of the other lenses plus the Phase 2 context and citation results.
- **Outputs.** Per-lens findings in the shared format, and the one-sentence core-contribution reading per lens. Ledger prefixes per lens (`REV-NOV`, `REV-ARG`, `REV-THEO`, `REV-METH`, `REV-STAT`, `REV-MEAS`, `REV-QUAL`, `REV-MIX`, `REV-CAUS`, `REV-PRAC`, `REV-ETH`).
- **Dispatch group.** One specialist-reviewer per active lens on the first pass, then one re-dispatch per lens for the challenge round.
- **Model tier.** Frontier, both rounds. This is the largest frontier consumer.
- **Parallelism.** First-pass lenses run in parallel via `.parallel()`. Phase 2 and the three Phase 4 clusters run concurrently with the first pass.
- **Gate condition.** First-pass fragments are merged before the challenge round dispatches. The challenge round updates a position only on named new evidence, preserves evidence-based dissent, and marks per-lens completion in the checkpoint.

**PIPE-07.** The first pass is blind. No specialist dispatch contains another specialist's findings, hypotheses, or severity calls. A test asserts that the first-pass input object for any lens contains no field sourced from another lens.

**PIPE-08.** Phase 2 results are injected at the challenge round, not the first pass, so the first-pass blindness invariant holds while context still reaches the lenses before synthesis. A test asserts the context and citation objects are absent from every first-pass input and present in every challenge-round input.

### Phase 4: Integrity screening

- **Purpose.** Run the six integrity rubrics as editorial signals, never verdicts.
- **Inputs.** The manuscript map, the figure and table inventory, the claim-evidence matrix, any supplied external artifact (similarity report, detector output, routed guideline), and the ledger.
- **Outputs.** Per-cluster signal findings classified none, low, moderate, or serious, with explicit not-applicable and not-run outcomes recorded. Prefixes `REV-RPT`, `REV-SIM`, `REV-AIC`, `REV-FIG`, `REV-CON`, `REV-RPX`.
- **Dispatch group.** Three integrity-screener clusters. Cluster 1: reporting standards plus reproducibility. Cluster 2: consistency plus figure integrity. Cluster 3: similarity signals plus AI-content risk.
- **Model tier.** Cheap, all three.
- **Parallelism.** The three clusters run in parallel with each other and concurrently with the Phase 3 first pass.
- **Gate condition.** Serious signals are scoped editor-only in the ledger. A check whose external artifact is missing is marked not-run with the artifact named, never improvised.

### Phase 5: Swarm evaluation

- **Purpose.** Stress-test the merged findings against a stratified reviewer ensemble to separate findings that survive many perspectives from fragile ones and to force dissent to surface.
- **Inputs.** The merged ledger, the swarm seed packet, and the study-design classification.
- **Outputs.** The swarm summary: recommendation distribution round 0 versus final, consensus entropy, decision stability, stable and fragile findings with round-by-round support, the strongest minority report, and herding risk. Swarm-surfaced findings under `REV-SWM`.
- **Dispatch group.** The seven-agent swarm (mode A): seed constructor, population generator, interaction moderator, local reviewer fish, consensus and dissent analyst, bias and herding monitor, report evaluator. Specified in section 6.3.1.
- **Model tier.** Cheap, all seven.
- **Parallelism.** Internal to the swarm. The fish population runs its rounds under the moderator's information control.
- **Gate condition.** Fragile findings and the strongest minority report feed Phases 6 and 7. They never delete a ledger row. The swarm modulates recommendation confidence and never sets the recommendation.

### Phase 6: Report construction

- **Purpose.** Build the full internal report from the ledger and swarm summary, ordered by severity and fixability, with provisional rubric scores.
- **Inputs.** The merged ledger, the findings, and the swarm summary.
- **Outputs.** The internal `full-report`, provisional 15-criterion scores citing Finding IDs, and the three lowest criteria named as bottlenecks.
- **Dispatch group.** review-report-writer mode A.
- **Model tier.** Frontier.
- **Parallelism.** None. Mode A consumes the whole merged evidence base.
- **Gate condition.** Every provisional score cites at least one Finding ID. The report is the audit trail from which the shipped documents derive.

### Phase 7: Synthesis and the release gate

- **Purpose.** Turn the evidence into one editorial judgment, derive the shipped documents, and pass or fail them at the adversarial gate.
- **Inputs.** The full report, the ledger, the swarm summary, the journal scope material, and the calibration memory.
- **Outputs.** The meta-synthesis with final rubric scores and the rubric-mapping table, the recommendation package with confidence, the swarm report-critique, the shipped seven-part peer-review report, the editor summary, and the final-critic verdict.
- **Dispatch group.** review-meta-reviewer and swarm report-evaluator (mode B) in parallel, then review-report-writer mode B, then review-final-critic.
- **Model tier.** Meta-reviewer, writer mode B, and final critic frontier. Swarm mode B cheap.
- **Parallelism.** The meta-reviewer runs concurrently with swarm mode B. The writer mode B waits on both.
- **Gate condition.** The release gate. See 5.6.

**PIPE-09.** The report writer never certifies its own output. Release requires the review-final-critic's structured verdict on a separate node. A test asserts the verdict object is produced by a step distinct from the writer step.

### Phase 8: Production and close-out

- **Purpose.** Render the branded deliverables, compute the run analytics, and persist the cross-review lessons.
- **Inputs.** The passed peer-review report, the editor summary, the ledger, the swarm summary, and the final-critic gate record.
- **Outputs.** `Peer-Review-Report.docx` and `Editor-Summary.docx` in the Psynalytics house style, the quality-metrics dashboard, the journal scope record, and the calibration append.
- **Dispatch group.** Deterministic document generation, plus the quality-metrics engine, the journal scope scorer, and the review calibrator as cheap judges. These are analytics, not gates.
- **Model tier.** Cheap for the judges. Document rendering is deterministic code.
- **Parallelism.** The three analytics judges run in parallel after the deliverables render.
- **Gate condition.** A high composite metric never overrides a gate outcome. Analytics inform system calibration across reviews, not the release of this one.

**PIPE-10.** Journal-specific calibration is withheld until at least ten completed reviews exist for the target journal. Below that count the calibrator applies the cross-journal fallback and labels the calibration as not journal-specific. A test asserts the calibrator refuses journal-specific output below the threshold.

## 5.3 The overlap schedule

The schedule below is the concurrency contract. A group starts only when its named precondition holds. Everything else about a group runs in parallel with the groups on the same tier.

| Group | Starts after | Runs concurrently with | Waits on |
|---|---|---|---|
| P0 sanitiser | run start | nothing | manuscript package present |
| P1 analyst A | P0 sanitised copy | none | Tier 3 clear |
| P1 analyst B | P1 analyst A map | analyst A tail | manuscript map |
| P2 field-context ∥ citation audit | P1 analyst B matrix | P3 first pass, P4 clusters | claim-evidence matrix |
| P3 specialist first pass | P1 analyst B activation map | P2, P4 clusters | activation map recorded |
| P4 integrity clusters (×3) | P1 analyst B outputs | P3 first pass, P2 | figure inventory, matrix |
| P3 challenge round | P3 first pass merged | none | context and citation results injected |
| P5 swarm (7 agents, mode A) | P3 challenge + P4 merged | none | full merged ledger |
| P6 report writer A | P5 swarm summary | none | swarm summary |
| P7 meta-reviewer | P6 full report | P7 swarm mode B | full report, ledger |
| P7 swarm mode B (report evaluator) | P6 full report | P7 meta-reviewer | full report |
| P7 writer mode B | P7 meta-reviewer + swarm mode B | none | recommendation package, report-critique |
| P7 final critic (release gate) | P7 writer mode B | none | shipped documents |
| P8 production + analytics | gate verdict pass | none | pass verdict |

**PIPE-11.** The three concurrency clusters (P2, P3 first pass, P4) join at a single barrier before the challenge round. The challenge round does not start until all first-pass lens fragments and both Phase 2 results are merged. A test asserts the barrier blocks on any outstanding first-pass or Phase 2 step.

## 5.4 Dispatch and tier budget

The budget below is the target envelope for a typical quantitative manuscript with roughly eight to nine active lenses. The exact count scales with the activation map.

| Tier | Dispatches | Steps |
|---|---:|---|
| Frontier | ~20 to 22 | specialist lenses ×2 rounds, analyst mode B, writer mode A, writer mode B, meta-reviewer, final critic (plus one critic per fix cycle) |
| Cheap | ~18 to 23 | sanitiser, analyst mode A, field-context-scout, citation-auditor, three integrity clusters, seven swarm agents, swarm mode B, three Phase 8 analytics judges |
| **Total** | **~40 to 45** | |

**PIPE-12.** Frontier dispatches are held to the ~20 to 22 envelope by construction: only the lens passes, analyst mode B, both writer modes, the meta-reviewer, and the critic use the frontier tier. Any new frontier consumer requires an explicit budget change. A test totals frontier dispatches on a reference manuscript and asserts the ceiling.

## 5.5 Runtime budget and preconditions

Runtimes assume the stated precondition. They are disclosed to the user as a live ETA and are never trimmed to look faster.

| Path | Fix cycles | Target runtime |
|---|---:|---|
| Happy path | 0 | ~38 to 40 min |
| One fix cycle | 1 | ~45 to 48 min |
| Two fix cycles | 2 | ~53 to 56 min |

- **Precondition.** A hosted frontier provider with prompt caching. The shared cached prefix (PIPE-04) is load-bearing for these figures. Without prompt caching the frontier cost and latency rise and the figures do not hold.
- **Local-provider caveat.** A local or Ollama deployment runs on its own measured ETA and a depth-capped preset (reduced swarm population and reduced conditional-lens depth). Its runtime is disclosed from its own measurement, never quoted from the hosted figures above.

**PIPE-13.** The workflow computes and surfaces a live ETA at run start and updates it at each fix-cycle entry. The ETA reflects the actual provider profile in use, hosted or local. A test asserts the local profile never reports the hosted figures.

## 5.6 Gate machinery

The release gate lives at Phase 7 and is the only quality gate that can send work back. It is a `.dountil()` loop over the block {writer mode B (or a specialist re-dispatch) → meta-reviewer re-run when recommendation inputs changed → final critic}. The loop exits when the critic returns **pass** or when the cycle counter reaches two.

The final critic returns exactly one verdict:

| Verdict | Orchestrator action |
|---|---|
| **pass** | Exit the loop, proceed to Phase 8. |
| **revise** | Re-dispatch the report writer on the named sections, then re-run the critic. Counts one cycle. |
| **revise-specialist [lens]** | Re-dispatch that specialist with the critic's specific objection, merge the revised fragment, re-run the meta-reviewer if the recommendation inputs changed, then the writer, then the critic. Counts one cycle. |
| **block** | Halt. Write the editor-only block summary, record it in the checkpoint, surface to the user. |

**PIPE-14.** revise and revise-specialist iterations count against the same two-cycle cap. The counter increments once per loop iteration regardless of which verbs fired within it. A test drives one revise then one revise-specialist and asserts the loop exits to arbitration on the third would-be iteration.

**PIPE-15.** A block verdict halts immediately and is never counted as a cycle. The block conditions are: a confidentiality violation, an integrity signal stated as a verdict in author-facing text, or a recommendation whose primary rationale cannot be grounded in Finding IDs. A test asserts any one condition forces halt without incrementing the counter.

**Deterministic arbitration.** When the loop exits at cycle two without a pass, control passes to a deterministic arbitration node. The exit is deterministic control flow, not a model decision. The node applies forced-halt rules first: a confidentiality or verdict-term objection, or an ungroundable recommendation, halts with no discretion. Otherwise the node resolves the objection to exactly one of three outcomes, each logged to the checkpoint with its rationale.

| Outcome | Meaning | Evidence requirement |
|---|---|---|
| accept-and-narrow | The objection stands, the recommendation is narrowed to what the evidence supports. | The narrowed category and the Finding IDs that bound it. |
| overrule-with-named-evidence | The objection is overruled. | The specific ledger Finding IDs that make the critic's worse outcome impossible. A bare overrule is invalid. |
| halt | The objection cannot be resolved on the evidence. | A halt report with what was tried and two to three options. |

**PIPE-16.** No arbitration outcome is silent. Every accept-and-narrow, overrule, and halt writes its outcome, the objection it answers, and its named evidence to the checkpoint. A test asserts the arbitration record exists whenever the loop exited at cycle two.

**PIPE-17.** overrule-with-named-evidence is rejected by the orchestrator when it does not cite at least one ledger Finding ID that discharges the objection. The default on insufficient evidence is halt. A test supplies an evidence-free overrule and asserts it is refused.

## 5.7 Human-in-the-loop points

The pipeline is autonomous by default. The release gate replaces the user approval gates. Two HITL points exist, both implemented with workflow suspend and resume.

**PIPE-18.** After the Phase 1 lite parse, if the manuscript file or the journal is missing or ambiguous, the workflow suspends with the smallest resolving question and resumes on the user's answer. It never invents a missing input. A test asserts the workflow suspends rather than proceeds when the journal is absent.

**PIPE-19.** An optional pause-at-gates mode, requested by the user, suspends the workflow at each phase boundary for review before continuing. The default mode does not pause. A test asserts pause-at-gates suspends at every boundary and the default mode suspends at none.

**PIPE-20.** A suspended workflow persists its full state and resumes deterministically from the suspension point with no re-paid phases. A test suspends at a gate, restarts the worker, and asserts clean resume.

## 5.8 Failure handling per phase

The failure contract is uniform: an agent that returns unusable output is re-dispatched once with the specific defect named. A second unusable return halts that phase, records a run-error flag in the checkpoint and in the editor summary run audit, and surfaces to the user with options.

**PIPE-21.** Unusable output is defined by schema validation failure or a contract miss (missing anchor, missing required field, ungrounded claim). The retry dispatch names the exact defect. A test feeds a schema-invalid return and asserts one named-defect retry, then a halt on a second failure.

**PIPE-22.** A hard stop halts cleanly with the checkpoint updated and a halt report carrying what was tried, the evidence, and two to three options. The hard-stop conditions are a Tier 3 tampering verdict, an ungroundable recommendation after cycle two, and manuscript materials too incomplete to review. A clean halt beats an ungrounded review. A test asserts each hard-stop condition produces a halt report and no shipped document.

**PIPE-23.** A missing external artifact never produces an improvised substitute. The dependent check is marked not-run with the artifact named, the phase continues, and the gap is carried to the editor summary. A test asserts a missing similarity report yields a not-run outcome rather than an invented similarity figure.

**PIPE-24.** Every halt, retry, arbitration, and gate cycle is written to the run audit in the editor summary with a checkpoint pointer, so the editor sees the full procedural history of the run. A test asserts the run audit reflects the recorded gate-cycle counters and any arbitration events.
