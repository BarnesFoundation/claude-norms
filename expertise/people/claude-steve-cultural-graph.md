# Claude (Steve's sessions) — Cultural Graph (CMPVisualDashboard)

What this Claude has learned and implemented through hands-on work on the Cultural Graph PoC. Accumulated implementation knowledge, not general training data.

## Project Context

- **Repo**: `CMPVisualDashboard` (local at `C:\Users\steve\CMPVisualDashboard`)
- **Source-of-truth spec**: `docs/cultural-graph-handoff.md` — a handoff doc Steve produced via a parallel Claude 4.6 conversation on 2026-04-26
- **Goal**: 3D physics-based force-directed graph visualization of the Philadelphia cultural ecosystem (institutions, funders, people, projects, programs), backed by Neo4j and queryable via natural language → Cypher
- **Strategic purpose**: Position Barnes Foundation within the OACCE Cultural Master Plan, Pew/PCAH evolution, Knight, William Penn ILI, and Barnes FY27 strategic plan. Long-term: licensable product for any city/region.
- **Original demo target**: 2026-04-30 (postponed). New target: **2026-05-04**, 1:1 with Thom Collins (ED).
- **Session started**: 2026-05-03

## Implemented
*(this file is updated as work lands; nothing here yet at session start)*

## Knows From Spec (the handoff doc)

### Stack lock-ins
- Neo4j AuraDB Free (graph DB; bolt+s connection)
- Cypher as query language; Claude generates Cypher natively
- AWS Lambda (Node.js) for deployed backend; matches CaptionCue infra; SAM template
- React + Vite + TypeScript frontend
- `react-force-graph-3d` (Three.js under the hood) for the 3D viz
- Anthropic API for natural-language → Cypher translation
- Voice input (Whisper/Deepgram) is roadmap, not PoC

### Why NOT Neptune
Gremlin/SPARQL are uglier and Claude is less fluent in them; no real free tier; smaller ecosystem; Neo4j Browser is invaluable for debugging.

### Licensing
Neo4j Community Edition is GPLv3 + Commons Clause. Building applications ON TOP of Neo4j is fully permitted, including commercial use. Selling Neo4j-as-a-service is the only thing the Commons Clause blocks. Production path: AuraDB Pro (~$65/mo) or self-hosted Community on EC2 (~$15/mo).

### Data model essentials
- Node labels: `:Project`, `:Institution`, `:Funder`, `:Person`, `:Program`, `:Descriptor`, `:Methodology`
- Multi-label nodes are a feature, not a bug. PCAH is `:Institution:Program`. Multiple relationship types between the same two nodes are expected (e.g., `(pcah)-[:OPERATES_UNDER]->(barnes)` AND `(barnes)-[:HOSTS]->(pcah)`).
- Funding edges carry `{amount, year, program}` properties; relationships carry `{role}` for INVOLVED_IN.

## Knows But Hasn't Implemented Yet

### Cypher fluency
Read-only patterns (MATCH, OPTIONAL MATCH, WITH, collect/aggregate, shortest path), seeded data loading, parameterized queries via neo4j-driver. Hasn't yet exercised APOC procedures or full-text indexes here.

### react-force-graph-3d
The library exposes:
- Standard force-directed simulation (`d3-force-3d`) you can override with custom forces via `onEngineInit` (e.g., `engine.d3Force('y', d3.forceY(...))` to lock entities to Y planes)
- `nodeThreeObject` callback for replacing the default sphere with custom Three.js geometry per node — this is how Project (sphere) vs. Program (torus) shape differentiation lands
- `nodeOpacity`/`linkOpacity` callbacks (REACTIVE based on React state, NOT mutating node properties — the snippet style in the handoff doc that mutates `node.opacity` directly does not trigger re-render)
- `cooldownTicks` to freeze layout after settling

### Neo4j AuraDB Free constraints
200K nodes free; bolt+s connection only (no plain bolt); auto-pause after 3 days inactivity; Cypher-only query interface.

### AWS SAM
Single-Lambda + API Gateway is a 30-line `template.yaml`. CORS handled at API Gateway level. Local invoke via `sam local start-api`. Deploy is `sam build && sam deploy --guided` first time.

### Three-layer shared-memory pattern (claude-norms)
This Claude already adopted the pattern at session start: `CLAUDE.md` (Layer 2 boot bridge) + `docs/claude/` (Layer 3 versioned shared memory: STRATEGY.md, DECISIONS.md, contributors/, context/). Has NOT yet written the per-project files; doing so is the next step in this PoC.

## Architectural Decisions Already Made

These will land in `CMPVisualDashboard/docs/claude/DECISIONS.md` as ADRs at session start; recorded here for cross-project visibility:

- **Project + Program share the bottom plane**, distinguished by 3D node geometry (sphere vs. torus) rather than Y-axis position. Reasoning: they are conceptually distinct (one-time vs. ongoing) but a project can seed/become a program — sharing a plane preserves that lineage. Shape differentiation (not color alone) is required for color-blind accessibility.
- **Layout strategy is query-parametric, not hardcoded.** The frontend accepts a `LayoutStrategy` (`{planeMap, nodeShape, clusterAxis}`) per query. Same node set can be top-down funding-flow, person-centric, beneficiary-centric, or arbitrarily restructured by future queries. Hardcoding the plane map would create the wrong abstraction.
- **Dual-mode backend.** Core logic is a pure `cypherToGraph(cypher, params)` function. Two thin adapters: Express for local dev (port 4747), AWS Lambda + API Gateway for deploy. Frontend reads `VITE_API_BASE` env var to switch between them. Demo runs locally; cloud deploy is a deferred non-blocker.
- **Click highlight via React state + library callbacks**, not by mutating node properties. The snippet in the handoff doc that does `n.opacity = …` would silently no-op against `react-force-graph-3d`'s reactivity model.
- **Local dev ports 3737 (Vite) and 4747 (Express)** registered in `shared-memory/port-registry.md` per Focus-3.0's frontend-3000s/backend-4000s convention.

## Lessons Learned on This Project (in progress)

1. **Read claude-norms files via `gh api`, not `WebFetch` summaries.** A WebFetch top-level summary missed the entire `shared-memory/port-registry.md` and conflated the cross-project shared-memory directory with the per-project pattern. Always pull canonical text.
2. **Default branch on `claude-norms` is `master`, not `main`** — the repo predates the GitHub default-branch change. Naive raw URLs against `main` 404.
3. **Treat code in handoff/spec docs as pseudocode for scope.** Steve's note: "You are the coder." Don't quote-critique snippets line-by-line; just write working code and briefly note the intent of any departure.
