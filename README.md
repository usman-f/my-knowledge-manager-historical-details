# my-knowledge-manager-historical-details

Archive of settled/completed/educational docs moved out of `my-knowledge-manager`
to keep that repo lean for future development. Nothing here is required to build or
extend km; it is preserved for reference. Paths mirror the source repo
(`knowledge-manager/...`).

Current, forward-looking docs stay in `my-knowledge-manager`:
`design/{principles,model,decisions,processing-layer,glossary}.md`, all of `roadmap/`,
`next-steps-impact.md`. Consult those first; this repo only for history/rationale detail.

## Contents

### `knowledge-manager/implementation-plan/`
Completed build plan for the core engine (design→code; all phases done, tests pass).
Superseded as a live view by `roadmap/status.md` (built-vs-planned baseline) in the main repo.
(The unbuilt `perspectives-properties-value.md` forward plan stays in the main repo, not here.)

### `knowledge-manager/learning-content/` + learning aids
Step-by-step codebase walkthrough that teaches the code and Rust as it was built.
- `learning-content/step-NN-*.md` — worked answers/explanations per step.
- `code-base-learning.md` — the reading plan those steps answer.
- `first-change-walkthrough.md` — guided first-feature exercise.
- `borrow-checker-guide.md` — Rust borrow-checker primer keyed to the codebase.

### `knowledge-manager/design/` — exploratory notes behind settled decisions
Working notes whose conclusions are now implemented in `km-core` and distilled in the
main repo's `design/decisions.md` / `design/model.md`:
- `positioning.md`, `lane-and-use-case-insights.md` — 2026-06-12 market/lane + SAP-workload studies (conclusions folded into `next-steps-impact.md`).
- `example.md` — worked ISA 5.1 flow-loop example of the model.
- `entity-id-as-string.md`, `name-typing-and-source-provenance.md` — resolved model Q&A.
- `selection-and-derived-sets.md`, `derived-primary-performance.md` — derived/intensional set decision + perf follow-on.
- `intensional-sets-and-edge-features-plan.md` — completed implementation spec (value index, intensional sets, edge reification).
- `inter-relation-norms-and-restructure.md` — model-drift retrospective + guardrails.
