# 13. Observability

Product Requirements Document. MARA (Multi-Agent Academic Peer-Review Architecture).
Psynalytics. Author: Prof. Llewellyn van Zyl, PhD. Status: draft for build. Language: British English.

Observability is local first. Everything an owner needs to understand a run is computed from the local database and shown in a statistics panel, with no external service required. Langfuse is a soft dependency that adds richer tracing only when the owner supplies its environment variables. Anonymous telemetry is off by default and asks once. Requirements carry the `OBS-` prefix.

## 13.1 Local statistics panel

**OBS-01** The statistics panel is computed entirely from the `dispatches` and `phase_checkpoints` tables (Section 7) and requires no external service. It renders per run and in aggregate across all runs.

**OBS-02** The panel reports these metrics, each with its source:

| Metric | Definition | Source |
| --- | --- | --- |
| Cost per run | Sum of `cost_usd` over a review's dispatches | `dispatches` |
| Cost per phase | Sum of `cost_usd` grouped by `phase` | `dispatches` |
| Completion rate | Share of runs that reach `completed` | `reviews.status` |
| Retry rate | Sum of `retries` divided by dispatch count | `dispatches` |
| Time to first review | Interval from run start to the first specialist finding | `phase_checkpoints`, `findings` |

**OBS-03** The panel figures match the definitions above exactly. A figure shown to the owner is never an estimate quoted from a brief. It is derived from the stored rows at read time.

**OBS-04** Statistics retention follows the review. When a review is purged (Section 7, DATA-19), its dispatches and events are deleted, so its contribution to the aggregate figures disappears. Statistics are not retained separately from the data they describe.

## 13.2 Structured logging

**OBS-05** The application writes structured logs. Each log line is a JSON object with a timestamp, a level, a component, and a message, plus structured fields for the event.

**OBS-06** Each model call writes one dispatch log record carrying the review identifier, phase, agent, provider, model, tokens in, tokens out, tokens cached, latency in milliseconds, cost in dollars, retry count, and outcome. This record mirrors the `dispatches` row and is the primary operational log for a run.

**OBS-07** Log levels are `error`, `warn`, `info`, and `debug`. Manuscript content, finding claim text, author identities, and deliverable prose never appear in a log above `debug`. At `debug` level, any manuscript excerpt is truncated and clearly marked, and `debug` is not the default level.

**OBS-08** API keys, master keys, wrapped data keys, and passphrases never appear in a log at any level (Section 11, SEC-16). The logger redacts known secret fields before serialising a record.

## 13.3 Optional Langfuse wiring

**OBS-09** Langfuse is enabled only when the `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, and `LANGFUSE_HOST` environment variables are present. When they are absent, the exporter is a no-op and the product runs with local statistics only. The absence of Langfuse is never an error and never blocks a run.

**OBS-10** When Langfuse is enabled, the tracing maps as follows: a review run is a session, the run has one root trace, each phase is a span under the root trace, and each dispatch is a generation under its phase span.

**OBS-11** Scores are attached to the run trace: the critic verdict, the rubric average, the recommendation category, and the quality-metrics composite (13.4). These four scores let an owner compare runs in Langfuse when the integration is on.

**OBS-12** The Langfuse `trace_id` for each dispatch is stored in `dispatches.langfuse_trace_id` so a local row links to its remote trace. The field is null when Langfuse is off, which keeps the schema stable across both modes.

## 13.4 Quality-metrics composite

**OBS-13** The quality-metrics composite is a single score in the range 0 to 1 that summarises the quality of a completed review. It is the weighted sum of five components, each itself in the range 0 to 1:

```
composite = 0.30 * evidenceGrounding
          + 0.25 * actionability
          + 0.20 * decisionStability
          + 0.15 * (1 - toneRisk)
          + 0.10 * (1 - unsupportedClaimPenalty)
```

**OBS-14** The five components and their sources:

| Component | Weight | Computation and source |
| --- | --- | --- |
| Evidence grounding | 30% | Mechanical share of report claims that resolve to a finding identifier. The report writer tags each claim, and a linter resolves each tag against the `findings` table. The value is resolved claims divided by total claims |
| Actionability | 25% | A cheap-tier LLM judge scores the author letter's recommendations for concreteness, returning a value in 0 to 1 |
| Decision stability | 20% | One minus the normalised Shannon entropy of the recommendation distribution produced by the swarm simulator. A run where the simulated reviewers agree scores near 1 |
| Tone risk | 15% | A cheap-tier LLM judge scores the author letter for tone risk against the developmental-voice standard. The composite uses one minus this risk, so a safer letter raises the score |
| Unsupported-claim penalty | 10% | A severity-weighted share of claims that do not resolve to a finding. The composite uses one minus this penalty. It is severity-weighted so that it measures the harm of unsupported claims rather than duplicating the evidence-grounding count |

**OBS-15** Evidence grounding and the unsupported-claim penalty both derive from the same claim-resolution linter but measure different things: grounding is the plain coverage ratio, and the penalty is the severity-weighted residual. This split is deliberate, so a single unsupported but high-severity claim is penalised more sharply than the coverage ratio alone would show.

**OBS-16** The composite and its five components are computed at Phase 7 after the release gate passes, stored with the run, and, when Langfuse is enabled, written as a score on the run trace (OBS-11).

## 13.5 Opt-in anonymous telemetry

**OBS-17** Anonymous telemetry is off by default. The product asks once, at setup, whether the owner wishes to enable it, and records the choice in `settings`. The choice is reversible at any time in the interface.

**OBS-18** When enabled, telemetry sends only these fields: phase durations in milliseconds, error classes as enumerated strings, the provider used per phase, the application version, the operating-system platform, and a random anonymous install identifier. It never sends manuscript content, finding text, titles, file paths, queries, or keys.

**OBS-19** Telemetry contains no manuscript-derived content of any kind. The field list in OBS-18 is exhaustive, and a field outside that list is never transmitted. When telemetry is off, no telemetry request leaves the machine.
