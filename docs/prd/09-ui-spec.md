# 9. Interface specification

This section specifies the MARA web application interface: seven screens, the app-kit component extensions they depend on, the new warn and fail tokens, and the accessibility and responsive rules that bind every surface. It extends the Psynalytics design system rather than restating it. Where a token, primitive, or rule already exists in the brand skill, this section names it and defers to it. Where the product needs something the marketing kits do not provide, it is specified here against the same tokens.

Requirements carry testable IDs (UI-01 onward). A requirement is met only when the stated condition can be observed in the built interface.

## 9.1 Scope and platform baseline

- **UI-01** The front end is built on Next.js 16 App Router. Every screen renders in the dark-first Psynalytics register on the base atmosphere `#032A30` with the three-layer radial mesh and 4% grain overlay from the website kit.
- **UI-02** Live pipeline state reaches the browser over Server-Sent Events (SSE). The application never polls for run progress.
- **UI-03** The primary viewport is desktop at a minimum width of 1024px. The interface degrades gracefully to 768px (see 9.13). Below 768px the application shows a single-column reduced view and a notice that a wider viewport is recommended for a running review.
- **UI-04** Glass cards use bone at 6% fill, border bone at 10%, radius 24px, and blur 12px. Card hover lifts 4px, moves the border to 16%, and raises the shadow to elevated, matching the website kit `.card`.
- **UI-05** All data, identifiers, counts, currency, and durations render in JetBrains Mono. All prose and labels render in Inter. No numeric run value appears in a proportional face.
- **UI-06** Lime `#A7D12B` appears only as an accent: the done check glyph, one confidence state, and active-state marks. It is never a fill, a background, or body text.
- **UI-07** Icons are Lucide, solid filled, `fill: currentColor`, no stroke, at 12 / 16 / 20 / 24px. No emoji and no glyph icons appear anywhere.
- **UI-08** Motion uses the token durations 150ms (fast), 280ms (normal), 500ms (slow) with the two ease-out curves. Every animation is suppressed under `prefers-reduced-motion: reduce` (see 9.12).

## 9.2 Shared foundations

### 9.2.1 New semantic tokens: warn and fail

The marketing kits encode all severity inside teal, lime, and bone. That recipe is safe for a static page but ambiguous in a live operational console, where lime already means "phase done" and teal already means "phase active". Reusing lime for danger and teal-mid for warning, as the marketing pills do, would collide with those running-state meanings on the same screen. The application therefore introduces two additional accents, scoped to operational status only. They are deliberately low-chroma so they sit quietly beside teal and bone rather than shouting.

- **UI-09** The application defines `--accent-warn: #E2A13C` (a muted amber) and `--accent-fail: #E07567` (a soft terracotta red). Contrast ratios are recorded in 9.2.2. These accents are used for the run-progress fail token, the degraded marker, error toasts, and the amber and red confidence and status states. They are not used on marketing surfaces, and they never become card fills or body text.
- **UI-10** Warn and fail status pills follow the unified brand recipe unchanged: 25% accent fill, 30% bone outline, 92% bone text. The new tokens are `--status-warn-bg: rgba(226,161,60,0.25)` and `--status-fail-bg: rgba(224,117,103,0.25)`, both with `--status-*-border: rgba(254,252,245,0.30)` and `--status-*-fg: rgba(254,252,245,0.92)`. Only the fill pigment changes.
- **UI-11** The confidence band on the results screen and the status vocabulary on the run screen share one traffic mapping so the whole app speaks a single language: Green is lime-accent, Yellow is warn amber, Red is fail red. Because lime and amber sit close on the wheel, every use of these states pairs the colour with a text label and a Lucide icon. No state is distinguished by colour alone (SC 1.4.1).

### 9.2.2 Contrast, focus, and motion baseline

- **UI-12** Measured contrast against the base atmosphere `#032A30` and against a bone-6% glass surface (composited to `#12373C`). Warn `#E2A13C` is 6.82:1 on base and 5.74:1 on glass. Fail `#E07567` is 5.02:1 on base and 4.22:1 on glass. Lime `#A7D12B` (the done check) is 8.56:1 on base and 7.21:1 on glass. Bone-92% text on the warn pill measures 7.12:1 and on the fail pill 8.01:1. All clear WCAG 2.2 AA for their role (4.5:1 where used as small text, 3:1 for icon and UI use).
- **UI-13** Keyboard focus uses a solid bone ring, not a teal halo, honouring the brand no-teal-focus-ring rule. `:focus-visible` renders `outline: 2px solid rgba(254,252,245,0.92)` with `outline-offset: 2px` and no blur or glow. The ring measures 12.71:1 on base and 10.70:1 on glass, far above the 3:1 minimum for a focus indicator.
- **UI-14** Teal `#008DA1` (3.86:1 on base) is used only for large text, fills, and non-text marks, never for body-size text, matching its brand role. Body and secondary text use the bone foreground tiers (fg-2 at 92%, fg-3 at 74%), which clear 4.5:1.

## 9.3 Screen 1: Setup wizard

**Purpose.** First-run configuration: connect a provider key, choose a speed and cost tier, set defaults, and decide on telemetry.

**Layout.** A centred glass panel (radius 24px) on the atmosphere, with a horizontal stepper across the top. Steps: Provider and key, Tier preset, Defaults, Telemetry, Review. One step is visible at a time. The stepper marks completed steps with a lime done check and the active step with a teal mark.

**Components.** Stepper (extension, 9.10), glass card, text input (website kit `.field`), status pill, primary and secondary buttons, radio-card group for tier presets.

- **UI-15** The API key field validates against the provider live. While a check is in flight the field shows an inline neutral pill reading "Checking". On success it shows an info pill "Verified" with a Lucide filled check. On failure it shows a fail pill "Not verified" with the provider error mapped to plain language.
- **UI-16** Three tier presets render as radio cards: Fast, Balanced, Thorough. Each card states the model mix, an estimated cost range in mono, and an estimated wall-clock range in mono. Selecting a card updates the review defaults on the next step.
- **UI-17** Telemetry is a single opt-in toggle, defaulted off. The copy states exactly what is and is not sent. No telemetry event fires until the toggle is on and the wizard is completed.
- **UI-18** The Review step summarises every choice in a read-only data table before the user commits. The primary button reads "Save setup".

**States.** Loading: the key check shows the neutral "Checking" pill and disables the Next control. Empty: no key entered leaves Next disabled with a helper line. Error: an invalid key surfaces the fail pill and a one-line remedy, and never advances the stepper.

## 9.4 Screen 2: Library

**Purpose.** The home surface. Past reviews as cards, plus a live card for any running review.

**Layout.** A responsive grid of glass cards (three columns at 1280px, one at 768px), newest first, with a header row holding the "New review" primary button and a search field.

**Components.** Glass card, recommendation pill, status pill, mono rubric figure, data table (list density toggle), empty state.

- **UI-19** Each review card shows the manuscript title, the submission date in mono, a recommendation-category pill, the rubric average as a mono figure to one decimal on a five-point scale, and a status pill (Complete, Running, Blocked, Failed).
- **UI-20** When a review is running, its card is pinned to the top of the grid, shows a teal pulse marker and a live phase label, and links into the run-progress view with the SSE stream already reconnecting.
- **UI-21** The empty state (no reviews yet) shows a single glass panel with a short line and one primary "Start your first review" button. It never shows a fabricated sample row.
- **UI-22** Card status pills use the unified recipe. Failed uses the new fail token, Blocked uses the warn token, Complete uses info, and Running uses a teal active mark.

## 9.5 Screen 3: New review

**Purpose.** Bring a manuscript in and parse it immediately.

**Layout.** A large dropzone card on the left, a detected-metadata sidebar on the right that fills as parsing completes.

**Components.** Dropzone (extension, 9.10), glass card, data table (metadata), status pill, popover.

- **UI-23** The dropzone is a card with a dashed bone-10% border. On drag-over the border moves to bone-16% and the fill lifts to bone-10%. It accepts a single manuscript file and states the accepted formats and size ceiling.
- **UI-24** On drop the file parses immediately. The sidebar populates detected metadata (title, authors count, word count, section count, figure and table counts, reference count) as key and mono-value rows. Each value carries a source anchor that opens an excerpt popover.
- **UI-25** A parse that cannot read the file surfaces a fail pill in the dropzone with a plain-language reason and leaves the previous state intact. The user can retry without reloading.

**States.** Loading: a thin progress meter runs along the dropzone base while parsing. Empty: the sidebar shows muted placeholder rows labelled "Awaiting file". Error: the fail pill replaces the progress meter.

## 9.6 Screen 4: Clarifying questions

**Purpose.** Confirm what the lite parse detected and gather the few inputs that shape depth and weighting, before the full run.

**Layout.** An interstitial first, then a two-block question card.

- **UI-26** After the file is accepted, a "Reading your manuscript" interstitial holds the screen for the lite parse (roughly one to two minutes). It shows a calm indeterminate meter and the current parse step in sanitised one-line form. It never shows raw model output.
- **UI-27** Block A is confirm-or-correct. Detected field and subfield, study design, and manuscript type each render as a selected chip the user can accept or replace from a short set. Chips use the unified pill recipe, and the detected value is pre-selected.
- **UI-28** Block B collects four inputs. Target journal is free text with an explicit "None" option. Review depth is Standard or Thorough, each showing a cost delta and a time delta in mono. An optional feedback-focus multi-select reweights emphasis only and can never exclude a review lens. An optional free-text field captures anything the reviewer should know.
- **UI-29** The whole screen is skippable. A "Skip with defaults" control proceeds using the detected values and the tier defaults. No question is mandatory.
- **UI-30** The feedback-focus control states in one line that it adjusts weighting only and does not turn any lens off. Selecting focuses does not reduce the set of lenses that run.

**States.** Loading: the interstitial. Empty: unanswered optional fields carry muted placeholders and are valid. Error: a lite-parse failure routes to the recovery panel pattern (9.7) rather than blocking the user on this screen.

## 9.7 Screen 5: Run progress

**Purpose.** The live console for an executing review. This is the most SSE-dense screen.

**Layout.** A vertical nine-phase timeline down the left, a detail column on the right that expands the active phase, a cost and ETA strip across the top, and a collapsible activity log at the base.

**Components.** Vertical timeline (extension), status pill, progress and cost meters (extension), log stream (extension), findings ticker, recovery panel, toast.

- **UI-31** The timeline lists the nine pipeline phases (Sanitisation, Manuscript analysis, Field context and citation audit, Specialist review, Integrity screening, Swarm stress test, Report construction, Meta-review and release gate, Memory and close). Each node has four base states: pending (bone-30% mark), active (teal pulse), done (lime check), failed (fail token). Degraded (a skipped optional phase) uses the warn token.
- **UI-32** The active phase expands to show per-lens status chips, each with a live finding count in mono. Chip states mirror the phase states.
- **UI-33** A findings-headline ticker shows each new finding as a mono ID, a bold plain-language label, and a severity pill. Editor-only findings are masked in this stream as "1 confidential signal logged" and never reveal their content on this screen.
- **UI-34** A cost and token meter renders in mono with a per-phase breakdown. It updates on each cost event and never estimates a figure it has not received.
- **UI-35** An ETA reads from bundled medians and is refined by the user's own local run history. It is labelled as an estimate and updates as phases complete.
- **UI-36** Release-gate cycles are framed as expected behaviour, not as errors. A gate revision reads "Release gate requested revisions, cycle 1 of 2" in a neutral pill, never in a fail register.
- **UI-37** A recovery panel offers three actions when a phase fails: retry the phase from its checkpoint, skip an optional phase (which then shows as degraded), or cancel and keep the partial result. A required phase cannot be skipped.
- **UI-38** The run survives a tab close and reconnects on return. On reconnect the SSE stream replays state to the current phase and the timeline reconciles without a full reload.
- **UI-39** The browser tab title shows the current phase and percentage while the run is active. A desktop notification fires once on run done, run failed, or release-gate block, subject to the user's notification permission.
- **UI-40** The activity log streams sanitised one-line events. It never shows chain-of-thought, raw prompts, or unredacted model output. It is collapsed by default and expands on demand.

**SSE events consumed.** The screen consumes the event catalogue defined in section 8 (API-23): `phase_status`, `lens_status`, `finding_headline`, `cost_tick`, `eta_update`, `gate_verdict` (carries the cycle counter for the gate pill), `log_event` (drives the activity log), `run_complete`, `run_failed`, and the comment heartbeat. Section 8 owns the event names and payload shapes. This screen consumes them and adds nothing.

**States.** Loading: before the first event, phases show pending and the detail column shows a connecting notice. Empty: not applicable, a run always has phases. Error: a dropped stream shows a reconnecting pill and retries with backoff, and a hard failure routes to the recovery panel.

## 9.8 Screen 6: Results

**Purpose.** Deliver the finished review: a summary first, then the full report, then the private notes, then downloads.

**Layout.** A summary card at the top, a tabbed region below (Report, Reviewer's private notes), a persistent download bar, and a side drawer and popover layer for evidence.

**Components.** Glass card, recommendation and confidence pills, mono stat, tabs (extension), markdown render, side drawer (extension), popover (extension), download menu.

- **UI-41** The summary card shows, in order: a recommendation-category pill, a confidence-band pill (Green, Yellow, or Red per 9.2.1), the rubric average as a large mono figure on a five-point scale, three bottleneck criteria, and the top five imperative changes.
- **UI-42** The Report tab renders the full report as on-brand markdown (typography in 10.5): Inter 18px body, mono finding IDs, and severity pills on concern labels.
- **UI-43** A finding ID in the report is interactive. Activating it opens the evidence-ledger side drawer for that finding, showing the ledger row and its manuscript anchor.
- **UI-44** A manuscript anchor is interactive. Activating it opens an excerpt popover showing the anchored manuscript text, without leaving the report.
- **UI-45** The Reviewer's private notes tab holds editor-only signals. On first open it shows a one-time explainer banner stating that these are editorial signals, not verdicts. This tab never appears in the author-facing report tab or its download.
- **UI-46** Downloads present a primary pill "Download report (.docx)" that is pre-generated and downloads instantly. An overflow menu offers the private notes .docx, the report .md, the evidence ledger .md, and an everything .zip.
- **UI-47** The private notes download and the report download are separate files. The author-facing report file contains no editor-only content and no integrity signal text.

**States.** Loading: the report tab shows a skeleton while the markdown renders and the docx pre-generates. Empty: a results view for a cancelled-with-partial run shows only the phases that completed and marks the rest as not run. Error: a missing artifact shows a fail pill in place of the affected download, with the others still available.

## 9.9 Screen 7: Settings

**Purpose.** Manage providers, model tiers, defaults, data location, and destructive actions.

**Layout.** A left navigation rail of setting groups and a right detail pane, both on glass.

**Components.** Data table, text input, radio-card group, tabs, toast, side drawer for confirmations.

- **UI-48** Providers and keys are managed in a data table with a live verified pill per key, reusing the setup-wizard validation.
- **UI-49** Model tiers are set per phase group, not per phase, so the user tunes cost against depth in a small number of controls.
- **UI-50** Defaults mirror the wizard: tier, review depth, and feedback-focus defaults, each editable here.
- **UI-51** Data location shows where reviews are stored on disk and lets the user change it. A change moves existing reviews or clearly states that it applies to new reviews only.
- **UI-52** A danger zone holds destructive actions (delete a review, delete all local data). Each requires an explicit typed or checkbox confirmation in a side drawer and shows exactly what will be removed before it proceeds.

**States.** Loading: setting groups render with skeleton rows. Empty: no providers configured shows a single "Add a provider" action. Error: a failed save shows a fail toast and leaves the previous value intact.

## 9.10 App-kit extension components

The website kit has no application primitives. The following components extend the existing tokens. Each is specified by anatomy, tokens, states, and motion. None introduces a new colour outside the token set plus the warn and fail accents of 9.2.1.

- **UI-53 Vertical timeline.** Anatomy: a vertical rail with a node per phase and a connector between nodes, where each node holds an icon, a label, and an optional expand region. Tokens: connector bone-10%, pending mark bone-30%, active teal `#008DA1` with a pulsing 25% teal ring, done lime check, failed fail token, degraded warn token. States: pending, active, done, failed, degraded, plus expanded and collapsed. Motion: the active pulse runs at 500ms ease-out and is replaced by a static teal mark under reduced motion, and expand and collapse run at 280ms.
- **UI-54 Data table.** Anatomy: header row, body rows, optional zebra, right-aligned mono numeric cells. Tokens: header text fg-3 uppercase with wide tracking, row divider bone-6%, hover row bone-6% fill, numeric cells JetBrains Mono with tabular figures. States: default, hover, selected (bone-10% fill), empty (single muted row). Motion: row hover fill at 150ms.
- **UI-55 Tabs.** Anatomy: a tab list and a panel region. Tokens: active tab text fg-1 with a 2px lime underline mark, inactive fg-3, panel on glass. States: active, inactive, focus (bone ring per 9.2.2). Motion: the underline slides at 280ms ease-out and the panel cross-fades at 150ms, with no motion under reduced motion.
- **UI-56 Toast.** Anatomy: an icon, a one-line message, an optional action, a dismiss control. Tokens: info uses the teal pill fill, success uses a lime-accent icon on a neutral fill, warn uses the warn token, fail uses the fail token, and all keep bone-92% text and a bone-30% outline. States: enter, rest, exit, and a paused state on hover. Motion: the toast enters at 280ms ease-out from the corner and auto-dismisses after a fixed dwell unless hovered or focused, with instant appear and manual dismiss under reduced motion.
- **UI-57 Dropzone.** Anatomy: a card with a dashed border, an icon, a prompt line, and a format and size note. Tokens: dashed border bone-10%, drag-over border bone-16% and fill bone-10%, reject state uses the fail token border. States: idle, drag-over, parsing (base progress meter), success, reject. Motion: border and fill transitions at 150ms.
- **UI-58 Progress and cost meters.** Anatomy: a track, a fill, and a mono readout. Tokens: track bone-10%, determinate fill teal `#008DA1`, indeterminate fill a teal sweep, readout JetBrains Mono. States: determinate, indeterminate, complete (fill settles, no lime unless the step is a phase done), error (fail token fill). Motion: the fill transitions at 280ms, and the indeterminate sweep is a 500ms loop that becomes a static partial fill under reduced motion.
- **UI-59 Log stream.** Anatomy: a scrolling column of one-line, timestamped, sanitised events, newest at the base, with a collapse control. Tokens: timestamp mono fg-4, message fg-3, and the container is a glass-1 surface. States: collapsed, expanded, auto-scrolling, and paused (when the user scrolls up). Motion: new lines fade in at 150ms, and auto-scroll is disabled under reduced motion and replaced by a "new events" affordance.
- **UI-60 Side drawer.** Anatomy: a right-anchored glass panel over a scrim, with a header, a body, and a close control. Tokens: panel glass-3, border bone-10%, radius 24px on the leading edge, scrim base at 60%. States: closed, opening, open, closing. Motion: the panel slides in at 280ms ease-out and the scrim fades at 150ms, and both are instant under reduced motion. Focus moves into the drawer on open and returns to the trigger on close.
- **UI-61 Popover.** Anatomy: a small glass panel anchored to a trigger, holding an excerpt or a source anchor. Tokens: glass-2, border bone-10%, radius 12px, shadow glass. States: closed, open, and a pinned state for keyboard users. Motion: fades and scales from 0.98 at 150ms, and is instant under reduced motion. It traps nothing and dismisses on escape or outside click.

## 9.11 Iconography and status vocabulary

- **UI-62** Every status and phase state pairs a Lucide filled icon with its pill or mark: check (done), a filled circle or dot (pending), a spinner-free pulse mark (active), a triangle (warn or degraded), and a filled octagon or x (failed). No state relies on colour alone.
- **UI-63** Recommendation-category and confidence-band pills use the unified recipe. The confidence Green, Yellow, Red states map to lime-accent, warn amber, and fail red respectively, each with a label word.

## 9.12 Accessibility (WCAG 2.2 AA)

- **UI-64** All text meets AA contrast: body and secondary text use bone foreground tiers at 92% and 74% (both above 4.5:1 on base and glass), and large text and non-text marks that use teal meet at least 3:1. The warn and fail figures in 9.2.2 are the binding values for those accents.
- **UI-65** Every interactive element is reachable and operable by keyboard. The setup wizard advances with Enter on the primary control and moves between steps without a pointer. The run screen exposes the recovery actions, the log collapse, and the timeline expand to keyboard. Tabs move with arrow keys and Home and End. The side drawer and popover trap focus while open and restore it on close.
- **UI-66** Focus is always visible via the bone ring of 9.2.2. No interactive element removes the outline without an equivalent visible replacement. Focus is never obscured by a sticky header or the download bar (SC 2.4.11).
- **UI-67** `prefers-reduced-motion: reduce` suppresses the active pulse, the indeterminate sweep, auto-scroll, and all slide and fade transitions, replacing each with an instant state change or a static affordance. No essential information is conveyed by motion alone.
- **UI-68** Live regions announce run milestones to assistive technology: phase completion, a gate cycle, a run block, and run completion are announced politely. The findings ticker and cost meter update without stealing focus.
- **UI-69** Every icon that carries meaning has a text label or an accessible name. Decorative marks are hidden from assistive technology.
- **UI-70** Colour is never the sole carrier of state (SC 1.4.1): the lime-versus-amber adjacency is always disambiguated by a label word and a distinct icon.

## 9.13 Responsive behaviour

- **UI-71** The design target is desktop-first at a minimum of 1024px. At 1280px the Library grid is three columns and the run screen holds the timeline and detail column side by side.
- **UI-72** At 768px the interface degrades gracefully: the Library grid collapses to one column, the run screen stacks the timeline above the detail column, and the results side drawer becomes a full-width sheet. All information remains reachable.
- **UI-73** Below 768px the application presents a single-column reduced view and advises a wider viewport for running a review. A completed review remains fully readable at this width.
- **UI-74** No screen scrolls horizontally at any supported width. Wide content (data tables, the report render, the cost breakdown) scrolls inside its own container, never the page body.
