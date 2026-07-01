# Insights: Lane Economics & SAP Use Case

Follow-on to `positioning.md` (2026-06-12). Two topics: why the niche is empty despite first movers; relational (SAP) exports as a target workload.

## Why the lane is open

- **First movers died of business model, not competition.** Kùzu: VC-backed, archived a working product to pivot — the niche couldn't return what the company needed. Cozo: single-maintainer project whose activity stalled.
- **Embedded databases have no natural revenue** — nothing to host or meter. Survivors are funded sideways: SQLite (consortium), DuckDB (MotherDuck cloud bolt-on), Oxigraph (no revenue requirement). The lane empties for the same reason it stays open.
- **Inversion for km**: a no-revenue-requirement OSS project is structurally fitter for this niche than the companies that died in it. Sustainability-as-feature (forkable, boring deps, documented format) is the moat, not a nicety.
- **Demand is the real risk, not competitors.** Most agent products use vector stores, pgvector/SQLite, or markdown files; graph memory layers (Zep/Graphiti, Mem0) are server-side services. Kùzu's archival proved a supply vacancy, not demand. The typed-validated-graph-memory thesis is plausible (long-horizon agents accumulating contradictory unstructured memories) but unproven.
- **No first mover on km's dimension.** Kùzu/Cozo competed on query power and speed; neither shipped validated writes or agent-evolvable schema. That dimension is untested, not lost.
- **Consequence**: MCP server is the cheapest demand experiment — zero-cost for real agents to try validated graph memory; run it before deeper investment.

## Relational exports (SAP) as workload

- **Core distinction: related ≠ similar.** Embeddings encode similarity; joins encode relatedness. A PO row and its vendor master share no text — embeddings put them far apart; the FK between them is the value. Vector-only storage structurally cannot answer "everything connected to vendor X"; it returns similar-looking rows instead.
- **Decision rule**:
  - Known join paths, human-written queries → plain SQL (DuckDB over exports); nothing fancier needed.
  - Entity-centric, open-ended neighborhood queries ("all suppliers/plants/BOMs/orders around this material") → graph; in SQL each hop is a join you must know in advance.
  - LLM/agent is the querier → graph strongly favored: text-to-SQL against SAP schemas (EKKO/EKPO/LIFNR/MARA, hundreds of cryptic tables) is brittle; named typed edges are LLM-legible.
- **Hybrid is the answer for "items related to an entity I describe vaguely"**: vectors resolve fuzzy phrasing to the entity node (find the door); explicit edges give the exact, complete neighborhood (walk the rooms). Neither layer alone works.
- **SAP → km mapping**: table → set; scalar column → literal-target definition; foreign key → entity-target definition; import validation catches broken references (common in real exports).
- **Scale guidance**: masters + documents-of-interest in the graph; bulk line items stay out (km unproven past ~1M entities; consistent with model.md's "don't mint a node per measurement").
- **Gap**: CSV/table importer is roadmap, not shipped — blocks this workload today.
- **Opportunity**: an SAP-style export is a near-ideal dogfooding dataset — fuzzy-entry + typed-neighborhood traversal is exactly the designed workload; candidate for the roadmap §4 demo.
