# 10. Brand and document design

MARA ships inside the Psynalytics design system. The `psynalytics-brand` skill and its `colors_and_type.css` (v3.0) are the single source of truth for tokens, type, and voice. This section lists only what the application carries in verbatim, what it extends, and the two places the product deliberately departs from the marketing kits. It then specifies the Word document respec and the results-screen markdown render, and it binds the voice rules for all user-facing copy.

Requirements carry testable IDs (BRAND-01 onward).

## 10.1 Carried in verbatim

- **BRAND-01** The application imports `colors_and_type.css` (v3.0) as its token base and uses the CSS variables directly. No colour, radius, spacing, blur, or type value is hardcoded when a token exists. The stale v2 `tokens.css` is not used.
- **BRAND-02** The full palette is carried unchanged: base `#032A30`, base-2 `#052E35`, teal `#008DA1`, teal-dark `#006D7C`, teal-mid `#559FB1`, teal-light `#E8F5F7`, lime `#A7D12B`, lime-light `#F2F9D9`, bone `#FEFCF5`, and the graphite and stone neutrals. Bone replaces pure white at every tier. No off-brand colour enters except the two operational accents of BRAND-06.
- **BRAND-03** The foreground tiers (fg-1 at 100%, fg-2 at 92%, fg-3 at 74%, fg-4 at 54%, disabled at 32%), the border tiers (6 / 10 / 16%), the glass surfaces (4 / 6 / 10%), the shadow set, the blur tiers (4 / 12 / 24px), the radius scale, the 5px spacing ladder, and the motion tokens (150 / 280 / 500ms with the two ease-out curves) are all carried in unchanged.
- **BRAND-04** The unified status-pill recipe is carried in unchanged: 25% brand fill, 30% bone outline, 92% bone text, pigment varies by state. The website kit primitives (`.card`, `.btn` in three variants, `.pill`, `.eyebrow`, `.field`, atmosphere layers) are the base grammar the app-kit components extend.
- **BRAND-05** Voice, iconography, and claims policy are carried in from the brand skill without change: British English, sentence case for product UI, evidence-first register, Lucide solid-filled icons only, and the banned-terms and claims rules.

## 10.2 Extended, not replaced

- **BRAND-06** The application adds two operational accents defined in the interface spec (section 9.2.1): `--accent-warn: #E2A13C` and `--accent-fail: #E07567`, with their pill fills at 25% and the standard bone outline and text. These exist because the run console overloads lime as "done" and teal as "active", so the marketing habit of encoding danger as lime and warning as teal-mid would collide with running-state meaning. The new accents are scoped to operational status (run-progress fail token, degraded marker, error toasts, and the amber and red confidence states). They never appear on marketing surfaces and never become fills or body text. Contrast values are recorded in section 9.2.2.
- **BRAND-07** The application adds the app-kit component set specified in section 9.10 (vertical timeline, data table, tabs, toast, dropzone, progress and cost meters, log stream, side drawer, popover). Each is built on the tokens above and introduces no new colour.
- **BRAND-08** Keyboard focus uses the brand-consistent bone ring, not a teal halo, per section 9.2.2. This honours the brand no-teal-focus-ring rule while giving keyboard users a clear indicator.

## 10.3 Font self-hosting

- **BRAND-09** Both `colors_and_type.css` and the whitepaper kit load JetBrains Mono through a Google Fonts `@import`. The application must not carry that import. It self-hosts JetBrains Mono locally via `@font-face` and removes the remote `@import` entirely. This is the one required edit to the token file when it is vendored into the app.
- **BRAND-10** Inter is self-hosted from the existing brand variable font `Inter-VariableFont_opsz_wght.ttf` (OFL). JetBrains Mono is added as self-hosted files (OFL). No screen makes a request to `fonts.googleapis.com` or any external font host.
- **BRAND-11** The font stack resolves Inter first for all prose and labels, and JetBrains Mono first for all data, identifiers, counts, currency, and durations, with the token fallback stacks behind each. No numeric run value renders in a proportional face.

## 10.4 Word document respec

The current generator (`house-docx-generator.js`) renders the branded deliverables in the clean-letterhead house style but relies on the proprietary Gadugi and Calibri Light faces. The respec ports it to a typed pure-Node `docx` module on the brand v3 type system, while keeping the letterhead, the teal register, and the anonymous reviewer identity.

- **BRAND-12** The generator is ported to a typed pure-Node module built on the `docx` library. It takes a typed job configuration and emits the deliverables without any browser or template dependency.
- **BRAND-13** Typography moves to brand v3 per this table:

| Role | Old | New |
|---|---|---|
| Headings and banners | Gadugi bold | Inter bold (embedded, OFL) |
| Body | Calibri Light | Inter regular (embedded, OFL) |
| Metadata, IDs, counts, numbers | Gadugi / Calibri | JetBrains Mono (embedded, OFL) |
| Fallback only | Gadugi, Calibri Light | named in the fallback stack, never embedded |

- **BRAND-14** Only OFL-licensed fonts are embedded (Inter and JetBrains Mono). Gadugi and Calibri Light are demoted to named fallbacks in the font stack and are never embedded, so no proprietary font ships inside a document. Because Inter is embedded, the fallbacks fire only on a machine that strips embedded fonts.
- **BRAND-15** All metadata-table values that are identifiers, counts, or numbers render in JetBrains Mono. Prose values in the metadata table stay in Inter.
- **BRAND-16** Page geometry is kept: A4 (11906 by 16838 twips), top margin 2350, bottom 1650, left and right 1134 twips. The full-bleed letterhead background (`letterhead-bg.png`, 794 by 1123) is retained behind the content on every page. There is no separate full-bleed cover page.
- **BRAND-17** Colour usage is kept on the brand teal palette: the title kicker in teal `#008DA1`, banners, sub-headings and the metadata first column in teal-dark `#006D7C`, body in graphite `#2B2D2E`, the subtitle and footer in grey `#595959`, bone `#FEFCF5` text on teal fills, teal-tint `#E8F5F7` row stripes, the lime `#A7D12B` rule under the title, and table borders in `#CFE6EA`. The grey `#595959` and the table-border `#CFE6EA` are retained legacy print values from the existing house generator, and no off-brand colour is introduced beyond these two.
- **BRAND-18** Section header treatment is kept: a `#` markdown heading becomes a white-on-teal-dark banner (bone text on a teal-dark fill), and a `##` heading becomes a teal-dark sub-heading. Tables keep the teal header with bone text, the bold teal-dark first column, and the alternating teal-tint and bone row stripes.
- **BRAND-19** The footer is kept: a centred confidential line with a live page number in the grey register. Pagination uses the document page field so it stays correct as content flows.
- **BRAND-20** The document creator and reviewer identity stay anonymous as "The Reviewer". No author name, affiliation, or model identity appears in any document, its metadata, or its properties.
- **BRAND-21** The module emits two documents from the same engine with different job configurations:
  - **Peer-Review-Report.docx** carries the seven-part shipped report structure owned by the review pipeline knowledge base, in the developmental author-facing register. It contains no editor-only content and no integrity signal text. Its title block reads "Peer review report".
  - **Reviewer-Private-Notes.docx** carries the editor-only register, holding integrity signals framed as editorial signals and never as misconduct verdicts. Its title block reads "Reviewer's private notes" and its footer marks it confidential. It shares the letterhead and teal register but is a separate file.
- **BRAND-22** The two documents never merge. The author-facing report file must not contain any private-notes content, and the private-notes file is generated and downloaded separately per section 9.8.

## 10.5 Results-screen markdown render

- **BRAND-23** The report renders as on-brand markdown on the dark register. Body text is Inter at 18px with line-height 1.65 in fg-2. Headings map to the brand heading styles (h2 and h3), and the eyebrow style marks section kickers.
- **BRAND-24** Finding identifiers render in JetBrains Mono and are interactive: activating one opens the evidence-ledger side drawer (section 9.10). Manuscript anchors render as interactive marks that open an excerpt popover.
- **BRAND-25** Severity labels on concern headings render as status pills using the unified recipe and the operational accents where a warn or fail severity applies. Links use teal-light. Tables use the data-table primitive. Inline values and counts render in mono.
- **BRAND-26** The render carries no editor-only content. Integrity signals appear only in the Reviewer's private notes tab and its document, never in the report render or the report download.

## 10.6 Voice rules for product copy

- **BRAND-27** All user-facing copy is British English in sentence case, evidence-first, and calm. It states what is happening and what the user can do, without hype.
- **BRAND-28** No em dashes and no semicolons appear in product prose. No emoji and no exclamation marks appear anywhere. These rules bind labels, helper text, empty states, toasts, notifications, and error messages alike.
- **BRAND-29** Copy never states or implies that the work is machine-produced or machine-assisted. Deliverables and interface text read as the reviewer's own and Psynalytics' work.
- **BRAND-30** Integrity language in any user-facing surface stays a signal, never a verdict. The banned verdict terminology owned by the review pipeline knowledge base is not used in copy, and editor-only signals carry the "signals, not verdicts" framing wherever they appear.
- **BRAND-31** Where a claim is labelled, the claim-label chips follow the brand recipe (Evidence, Inference, Hypothesis) already defined in the whitepaper kit. The full banned-terms list is the one in the brand voice guidelines and is not restated here.
- **BRAND-32** Error and recovery copy names the mechanism in one plain sentence and offers the next action. It never blames the user and never exposes raw model output, prompts, or chain-of-thought.
