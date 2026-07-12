# 6. Agent and knowledge specification

This section specifies the agents as workflow nodes, the knowledge base as their behavioural contract, the MARA v3 capabilities being restored in the rebuild, the enforcement mechanism for each invariant, and the rules for migrating markdown prompt fragments to structured schemas. Every scoring rule, voice rule, and evidence rule named here has one owning knowledge module, and this section defers to it. Duplicating a rule across modules is a defect, not a safety margin.

## 6.1 Agent roster mapped to workflow nodes

Each agent is one or more typed workflow steps. The input contract lists what the orchestrator passes. The output is a Zod-validated object. Model tier is frontier or cheap per section 5. The prompt source is the current agent definition or the restored MARA v3 prompt, carried into the new prompt as the method layer.

| Agent (node) | Phase | Tier | Input contract | Output object | Prompt source |
|---|---|---|---|---|---|
| manuscript-sanitiser | 0 | cheap | manuscript package, BRIEF | SanitiserResult | `.claude/agents/manuscript-sanitizer.md` |
| manuscript-analyst A | 1 | cheap | sanitised copy, quarantine log, journal reqs | ManuscriptStructure | `.claude/agents/manuscript-analyst.md` |
| manuscript-analyst B | 1 | frontier | mode A outputs | ClaimDesignAnalysis | analyst + study-design classifier prompt |
| field-context-scout | 2 | cheap | sanitised copy, claim matrix, scope | ContextPacket | `.claude/agents/field-context-scout.md` |
| citation-auditor | 2 | cheap | reference list, claim matrix | CitationAudit | `.claude/agents/citation-auditor.md` |
| specialist-reviewer (per lens) | 3 | frontier | lens rubric, map, matrix, ledger | LensFindings | `.claude/agents/specialist-reviewer.md` |
| integrity-screener (per cluster) | 4 | cheap | inventory, matrix, external artifacts | IntegritySignals | `.claude/agents/integrity-screener.md` |
| swarm (7 agents) | 5 / 7 | cheap | seed packet, ledger, design | SwarmEvaluation | phase_5 swarm prompt pack |
| review-report-writer A | 6 | frontier | ledger, findings, swarm summary | FullReportEnvelope | `.claude/agents/review-report-writer.md` |
| review-meta-reviewer | 7 | frontier | ledger, fragments, swarm, scope, calibration | MetaSynthesis | `.claude/agents/review-meta-reviewer.md` |
| review-report-writer B | 7 | frontier | recommendation package, report-critique | ShippedReportEnvelope | `.claude/agents/review-report-writer.md` |
| review-final-critic | 7 | frontier | report, reviewer's private notes, ledger, recommendation | CriticVerdict | `.claude/agents/review-final-critic.md` |
| quality-metrics engine | 8 | cheap | ledger, reports, swarm, gate record | QualityMetricsDashboard | phase_8 quality_metrics prompt |
| journal-scope scorer | 7 / 8 | cheap | journal scope, manuscript card, retrieval | JournalScopeScore | phase_8 journal_scope prompt |
| review-calibrator | 8 | cheap | final report, editor decision, benchmarks | CalibrationReport | phase_8 review_calibrator prompt |

**AGENT-01.** Every agent node produces a Zod-validated object and no free-text file. The orchestrator merges validated objects into the ledger. A test asserts each node's return parses against its schema and a parse failure triggers the PIPE-21 retry.

**AGENT-02.** The shared finding object is one schema used by every specialist and integrity node, so the ledger stays uniform. Its outline:

```ts
const Band = z.enum(["Green", "Yellow", "Red"]);
const Finding = z.object({
  id: z.string().regex(/^REV-[A-Z]{3,4}-\d{4}$/),
  lens: z.string(),
  phase: z.number().int(),
  claim: z.string(),
  anchor: z.string().min(1),                       // section+paragraph or table/figure id
  epistemic: z.enum(["Known", "Inferred", "Assumption"]),
  confidence: z.number().min(0).max(1),
  band: Band,
  severity: z.enum(["none", "minor", "moderate", "major", "fatal"]),
  fixability: z.enum(["easy", "moderate", "hard",
                      "not-fixable-from-current-study", "unclear"]),
  scope: z.enum(["author-facing", "editor-only", "both"]),
  failureScenario: z.string(),
  leanestFix: z.string(),
  supersedes: z.string().nullable(),
});
```

**AGENT-03.** The band field is validated against the confidence field. Green requires 0.98 to 1.00, Yellow 0.70 to below 0.98, Red below 0.70, exactly as owned in knowledge/01. A cross-field refinement rejects a mismatch, so a Green band on a 0.80 confidence is a parse failure. A test asserts the refinement fires.

**AGENT-04.** An empty anchor is a parse failure, enforcing the knowledge/01 rule that an unanchored finding never enters the ledger. A test submits a finding with a blank anchor and asserts rejection.

The specialist and meta output objects wrap the finding list with node-specific fields.

```ts
const LensFindings = z.object({
  lens: z.string(),
  coreContributionReading: z.string(),             // one sentence, compared across lenses
  findings: z.array(Finding),
  challengeRound: z.object({
    updated: z.array(z.object({ id: z.string(), evidenceThatChanged: z.string() })),
    dissentPreserved: z.array(z.object({ id: z.string(), whyItHolds: z.string() })),
  }).nullable(),
});

const MetaSynthesis = z.object({
  rubric: z.array(z.object({
    criterion: z.number().int().min(1).max(15),
    score: z.number().int().min(0).max(5),
    supportingIds: z.array(z.string()).min(1),
    opposingIds: z.array(z.string()),
  })).length(15),
  average: z.number(),                             // unweighted mean to one decimal
  bottlenecks: z.array(z.number().int()).length(3),
  recommendation: z.enum(["accept", "minor_revision", "major_revision",
                          "reject_and_resubmit", "reject"]),
  recommendationConfidence: z.number().min(0).max(1),
  scopeFit: z.object({ score: z.number(), factorsUsed: z.array(z.string()) }),
  decisionHinges: z.array(z.object({ findingId: z.string(), hinge: z.string() })),
});
```

**AGENT-05.** The rubric array is fixed at fifteen rows and every row cites at least one supporting Finding ID, enforcing the traceability rule in knowledge/03. A score with an empty supporting list is a parse failure returned to the meta-reviewer. A test asserts a fifteen-row rubric with one unsupported row is rejected.

## 6.2 Knowledge modules as the behavioural contract

The six knowledge modules are carried into `/knowledge` unchanged and retain their ownership map. They are the behavioural contract the agents load. The rebuild reads them as data, it does not paraphrase them into prompts.

| Module | Owns | Loaded by |
|---|---|---|
| 01 governance | constitution, confidentiality and retrieval rules, injection tiers, signals-not-verdicts terminology, epistemic labels and confidence bands, ledger rules | every agent |
| 02 lenses | shared finding format, activation logic, the 11 specialist rubrics, the 6 integrity rubrics, the impossible-results definition, first-pass independence | specialist, integrity, analyst, critic |
| 03 decision | the 15-criterion rubric, recommendation taxonomy and thresholds, decision hinge, swarm spec, revise-specialist routing | meta-reviewer, swarm, writer, critic |
| 04 voice | developmental stance, banned destructive phrasing list, golden thread of a finding | writer, critic, swarm, meta-reviewer |
| 05 templates | every artifact template and the house-style docx look | every agent |
| 06 craft | sentence discipline, per-concern micro-pattern, humanise pass, anonymity in prose | writer, meta-reviewer, critic, swarm |

**AGENT-06.** Each rule has exactly one owning module. The canonical values live once: recommendation thresholds only in 03, confidence bands only in 01, the impossible-results definition only in 02, the banned destructive phrasing list only in 04. A test greps the modules for a duplicated canonical value and fails on a second copy.

**AGENT-07.** An agent loads the modules named in its contract plus 01 always, and any agent that writes author-facing or editor-facing prose also loads 04 and 06. A test asserts the writer and critic nodes load 04 and 06 and the sanitiser does not load 03.

## 6.3 Restored capabilities

The rebuild restores five MARA v3 capabilities that the current single-agent pipeline collapsed or dropped. Each is specified to its exact canonical values.

### 6.3.1 The seven-agent swarm

The swarm replaces the single simulated-population context with the original seven-agent design, run on the cheap tier. The seven roles are seed constructor, population generator, interaction moderator, local reviewer fish, consensus and dissent analyst, bias and herding monitor, and report evaluator.

- **Seed packet.** The seed constructor builds a compact packet: the manuscript card, the top 20 to 40 decision-relevant findings, integrity items labelled as signals, and quarantined content marked untrusted. Author identity is redacted (names, institutions, affiliations, funder names) while every quantitative and methodological detail is retained. Minority findings carry a minority tag and are never dropped. The packet must not lean toward the candidate recommendation.
- **Population and topology.** The generator builds 24 to 48 profiles stratified by expertise and epistemic style, each declaring exactly one blind spot from the enumerated five: (1) dismissing unconventional methods as underpowered or unvalidated, (2) approving familiar methods despite assumption violations, (3) missing construct drift inside a familiar framework, (4) over-weighting writing quality at the expense of substance, (5) anchor bias on the first finding in the packet. Topology is a choice: ring is the default for decentralised stability, small-world when the manuscript is methodologically heterogeneous, stratified when it spans genuinely distinct disciplines, adversarial for high-stakes stress-testing. Ring shows each profile two to four neighbours per round.
- **Rounds.** Round 0 private judgment, round 1 local exchange, round 2 cross-stratum exchange, round 3 mandatory dissent, round 4 final position. Neighbour summaries are 30 to 50 words carrying the recommendation, the primary Finding ID anchor, and the declared confidence, with no reframing. No aggregate distribution is visible to any profile before round 4.

**AGENT-08.** A position shift is valid only when it names the specific new evidence, the prior position and why it is now weaker, and whether the declared blind spot contributed. A shift missing any component is reverted by the moderator. A test submits a shift citing only neighbour agreement and asserts it is reverted.

**AGENT-09.** Evidence-based dissent held at confidence 0.75 or higher survives into the final round even when every neighbour has converged. A test converges all neighbours and asserts a 0.80-confidence dissent still appears in round 4.

**AGENT-10.** Consensus entropy is Shannon entropy across the five recommendation categories, in bits:

```
H = - Σ (p_i · log2 p_i)   for i in {accept, minor_revision, major_revision, reject_and_resubmit, reject}
```

where `p_i` is the fraction of profiles in category `i` and a category with `p_i = 0` contributes zero. The maximum for five equal categories is log2(5) ≈ 2.32 bits. Above 1.5 bits is genuinely divided opinion, below 0.5 bits is near-consensus. A test computes H on a known distribution and asserts the value and the band.

**AGENT-11.** A finding is stable when supported by 60 percent or more of profiles in both round 0 and the final round. A finding is fragile when supported by 70 percent or more in round 0 and falling below 50 percent by the final round, or below 40 percent in every round. Both are reported with round-by-round percentages. A test drives support curves across the thresholds and asserts the classifications.

**AGENT-12.** Decision stability is the proportion of profiles whose recommendation survived interaction unchanged. High consensus on weak evidence is flagged as herding, never as certainty, and a prestige-driven shift is annotated prestige-biased with its substantive content preserved, never deleted. A test asserts a prestige-attributed shift is annotated rather than discarded.

### 6.3.2 Quality-metrics composite

The quality-metrics engine computes a composite quality score at close-out. It is a calibration and improvement tool across many reviews, never a release gate for one manuscript.

**AGENT-13.** The composite uses these exact weights: evidence-grounding rate 30 percent, actionability index 25 percent, decision stability from the swarm 20 percent, tone-risk score 15 percent, and unsupported-claim count applied as a 10 percent penalty. The weights used are recorded in the dashboard so a single run's score is interpretable in context. A test asserts the weights sum to the documented allocation and appear in the dashboard.

- **Evidence-grounding rate.** The share of report claims that trace to a ledger Finding ID with a manuscript anchor.
- **Actionability index.** The share of major concerns carrying a concrete leanest fix and a decision hinge.
- **Decision stability.** Carried from the swarm output.
- **Tone-risk score.** The count of banned-phrasing and severity-mismatch hits against knowledge/04, inverted so lower risk scores higher.
- **Unsupported-claim penalty.** The count of report claims with no supporting Finding ID, subtracted.

**AGENT-14.** A high composite never overrides a final-critic block. The composite is computed only after the gate has resolved and is stored for cross-review calibration. A test asserts a blocked run still records a dashboard and ships no deliverable.

### 6.3.3 Recommendation engine bands

The recommendation follows the canonical taxonomy owned by knowledge/03. It is a severity-and-fixability judgment constrained by journal scope and evidentiary confidence, never a vote count or a weighted average.

| Recommendation | Conditions (all must hold) |
|---|---|
| Accept | Rubric average at or above 4.0 AND no open major or fatal findings AND recommendation confidence at or above 0.90. |
| Minor revision | Average at or above 3.5, with 1 to 3 moderate concerns, all fixable without new data, nothing fatal. |
| Major revision | Average 2.0 to 3.4, with 4 to 8 major concerns, none fatal. |
| Reject and resubmit | Average 1.0 to 1.9, the contribution salvageable but the required redesign so fundamental that a revision could not be verified against the current version. |
| Reject | Average below 1.0, OR any fatal issue not fixable from current data, OR fundamental misrepresentation of evidence, OR an unresolved policy violation. Any one suffices. |

**AGENT-15.** Severity and fixability can pull the recommendation down from what the average permits, never up. One unresolved major validity issue blocks accept whatever the average says. A test constructs a 4.2 average with an open major finding and asserts the recommendation is not accept.

**AGENT-16.** Accept requires recommendation confidence at or above 0.90 with no exception. Below that the engine returns minor revision with an explicit list of what would raise confidence to 0.90. A test sets a 4.1 average at 0.85 confidence and asserts minor revision with the list.

### 6.3.4 QA bot boundary

The QA bot performs mechanical quality assurance only. Its boundary is exact and enforced.

**AGENT-17.** Mechanical is in scope: a broken cross-reference, a mismatched Finding ID, a typographical error in a heading, a broken internal link. Substantive is out of scope: a recommendation category inconsistent with the rubric average and severity calibration is escalated to the final critic with the inconsistency documented, never self-corrected. A test feeds a recommendation-average mismatch and asserts escalation rather than a silent edit.

### 6.3.5 Journal scope scorer

The scorer assesses fit between manuscript and journal using legitimate factors only.

**AGENT-18.** Legitimate factors are topic fit with stated journal aims, article-type compatibility, methodological-approach match with the journal's stated preferences, and contribution-type alignment. Prestige proxies are excluded: author institution ranking, the impact factor of cited sources, any author's h-index or citation count, and the topic's public attention. A test asserts a prestige proxy passed in the input does not move the score.

**AGENT-19.** The scorer does not reject for novelty where the journal accepts replication or confirmatory work, and it lowers confidence when the journal scope text is unavailable. A test asserts a confirmatory paper at a replication-friendly journal is not penalised for novelty.

### 6.3.6 Review calibrator

The calibrator compares recommendations against journal standards and historical decisions, kept separate from single-manuscript evidence.

**AGENT-20.** Journal-specific calibration is meaningful only at ten or more completed reviews for the target journal. Below that count the calibrator applies the cross-journal fallback against published peer-review and LLM-review benchmarks and labels the calibration as not journal-specific. A test asserts the label and the fallback below the threshold. (See PIPE-10.)

### 6.3.7 Reporting-guideline selector

The study-design classifier selects the reporting standard that matches the design and verifies the match before any checklist is scored.

**AGENT-21.** The selector routes from this list: CONSORT, PRISMA, STROBE, APA JARS, COREQ, SRQR, TRIPOD, STARD, ARRIVE, SPIRIT, or a journal-specific checklist. For a multi-standard design it produces one merged non-duplicate checklist with each item tagged by origin standard, the more specific version winning on overlap. A test asserts a mixed design yields a merged checklist with no duplicated items.

**AGENT-22.** The routed standard is verified against the design classification before scoring. A mismatch, such as CONSORT on a cohort study or PRISMA on a primary study, is itself a finding, the correct standard is named, and the checklist is rebuilt before a single item is scored. A test feeds a mismatched routing and asserts the rebuild before scoring.

## 6.4 Invariant enforcement

Each invariant maps to a named mechanism and a verification test. An invariant with no test is not enforced.

| # | Invariant | Enforcement mechanism | Verification test |
|---|---|---|---|
| 1 | Every report claim traces to a ledger Finding ID with a manuscript anchor, an ungrounded recommendation blocks release | Grounding audit at the release gate, block verdict on an ungroundable recommendation (PIPE-15) | Inject an ungrounded rationale bullet, assert block |
| 2 | Append-only ledger, single-writer merge, supersede-by-new-row | Orchestrator-only merge, `supersedes` field, no agent write path (PIPE-03) | Attempt a second writer, assert rejection |
| 3 | Integrity findings are signals never verdicts, serious is editor-only, banned verdict terms gate-enforced | Signal classification in the finding schema, editor-only scope on serious, banned-term grep at the gate | Set a verdict term in author-facing text, assert block |
| 4 | Developmental voice is a deliverable requirement | knowledge/04 and 06 loaded by writer and critic, tone-risk audit at the gate | Submit a destructive-phrasing letter, assert revise |
| 5 | Manuscript content and author identity never leave in outbound queries, published reference metadata allowlisted, unpublished references excluded | Retrieval-confidentiality rule bound to the two retrieval nodes, outbound-query scan (PIPE-06) | Scan query log against manuscript text, assert no overlap outside the allowlist |
| 6 | Specialist first pass blind, challenge round updates on evidence only, dissent preserved | Blind first-pass input (PIPE-07), Phase 2 injection at challenge (PIPE-08), evidence-only update rule | Feed a cross-lens field into a first pass, assert absence. Hold a fixture lens at dissent confidence 0.75 or above through the challenge round, assert it appears un-averaged in the merged ledger and visible to the swarm seed |
| 7 | No agent passes its own gate | Final critic on a separate node (PIPE-09), arbitration by the orchestrator | Assert the verdict node differs from the writer node |
| 8 | Max 2 fix cycles then logged arbitration | `.dountil()` cap of two, deterministic arbitration node (PIPE-14, PIPE-16) | Drive three would-be iterations, assert exit to logged arbitration |
| 9 | Three-tier injection quarantine, Tier 3 halts | Sanitiser tier classification, Tier 3 halt at Phase 0 | Plant Tier 3 tampering, assert halt with no shipped document |
| 10 | Known/Inferred/Assumption labels and confidence bands | Epistemic and band fields in the finding schema, band-confidence refinement (AGENT-03) | Submit a Green band at 0.80, assert parse failure |
| 11 | Recommendation is a severity-and-fixability judgment, pulls down never up | Recommendation engine bands, the down-only rule (AGENT-15) | Construct a high average with an open major, assert no accept |
| 12 | Single source of truth for every rule | One owning module per rule, ownership map preserved (AGENT-06) | Grep modules for a duplicated canonical value, assert none |

**AGENT-23.** The invariant table is itself a test suite. Each row's test runs in continuous integration and a failing row fails the build. A test asserts every invariant row has a live test bound to it.

## 6.5 Prompt-to-schema migration rules

The rebuild replaces markdown fragment files with structured envelopes. The rules below govern the migration so no contract is lost in translation.

**AGENT-24.** Each markdown fragment becomes a `generateObject` call with a Zod envelope. The prompt body carries the method (how the agent works) and the schema carries the contract (what it must return). The two are separate layers and the schema is the machine-checked one. A test asserts every migrated agent has both a prompt and a schema.

**AGENT-25.** The human-facing report body stays a markdown string field inside a structured envelope. The prose is prose and does not decompose into fields, so the envelope wraps it with the machine-checkable metadata around it:

```ts
const ShippedReportEnvelope = z.object({
  recommendation: z.enum(["accept", "minor_revision", "major_revision",
                          "reject_and_resubmit", "reject"]),
  recommendationConfidence: z.number().min(0).max(1),
  bodyMarkdown: z.string(),                        // the seven-part report, prose
  rubricTable: z.array(z.object({
    criterion: z.number().int(), score: z.number().int(), justification: z.string(),
  })).length(15),
  references: z.array(z.string()),                 // APA 7
  citedFindingIds: z.array(z.string()),            // every id referenced in bodyMarkdown
  editorOnlyLeak: z.literal(false),
});
```

**AGENT-26.** The `citedFindingIds` field lists every Finding ID referenced in the prose body, so the grounding audit checks the prose against the ledger without parsing free text. A test asserts every id in `citedFindingIds` exists in the ledger and every ledger id cited in the body appears in the field.

**AGENT-27.** The `editorOnlyLeak` field is a literal false the writer must assert, and the critic independently re-verifies it by scanning the body for integrity signals, policy content, and confidence caveats meant for the editor. A self-attested clean is not accepted as evidence. A test plants an editor-only signal in the body and asserts the critic overrides the false assertion to a block.

**AGENT-28.** The migration preserves the ledger ID scheme `REV-<LENS>-<seq>` verbatim across every prefix, so audit trails from the markdown era remain parseable. A test asserts the prefix set matches the knowledge/02 assignment exactly.

**AGENT-29.** No structured envelope carries a rule value that duplicates a knowledge module. Thresholds, bands, and taxonomies are read from the loaded modules at run time, not hard-coded into a schema default. A test asserts the recommendation thresholds are absent from the schema source and present only in knowledge/03.
