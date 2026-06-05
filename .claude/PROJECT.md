# PROJECT.md — pulseWebKit (Open Pulse Dashboard template)

Longer-form project context for agents. Companion to root `CLAUDE.md` (which covers conventions & dev workflow) and `.claude/SKILLS.md` (which covers concrete how-tos).

---

## Mission

**pulseWebKit** (working name: *pulseNext*) is a SvelteKit starter, intended to be published as a **GitHub template repository**, so that researchers and developers can build their own dashboards or interactive websites that pull variables from the **Open Pulse** platform.

Two audiences read this codebase:

1. **The SDSC team at ETH Zürich**, who use the three reference views (Graph Explorer, Pipeline Runs, Service Health) day-to-day to operate the live Open Pulse deployment at `openpulse.epfl.ch`.
2. **Downstream users** — anyone who clicks "Use this template" on GitHub to spin up a new project. They will keep, replace, or extend the reference views, and they expect this codebase to demonstrate idiomatic patterns they can copy.

When deciding between "this is a one-off feature" vs. "this is a pattern others will follow", err toward the second. Naming, file layout, abstractions, and design tokens should generalise.

---

## The Open Pulse platform (data layer)

Open Pulse is a research-software-observability stack maintained by SDSC. The dashboard does not own data — it surfaces it from three live stores deployed at `openpulse.epfl.ch`. All three live behind plain HTTP/HTTPS on internal ports; the `.env.example` documents the exact endpoints and the auth conventions (`replace-me` upstream placeholders).

### Neo4j — the property graph

- **Endpoints:** `:7503` (HTTP/REST/browser), `:7504` (Bolt for official drivers)
- **Contents:** Repositories, Contributors/Persons, Commits, Organisations, PullRequests, and the relationships between them
- **Use it for:** "Who contributed to what, when?" questions; graph traversal; visualisations of collaboration over time
- **Skill:** `query-neo4j` (Cypher → JSON rows). Always include `LIMIT` on exploratory queries — the graph has tens of thousands of nodes.

### Oxigraph (SPARQL) — RDF metadata

- **Endpoint:** `:7502`, behind a Caddy proxy that terminates HTTP-Basic auth (`/query` for reads, `/update` for writes)
- **Contents:** ~300k triples across multiple named graphs (e.g. `http://open-pulse/repos`, `http://open-pulse/metadata`)
- **Use it for:** Structured metadata, vocabulary/ontology queries, anything that benefits from `SELECT … WHERE { GRAPH ?g { … } }`
- **Skill:** `query-sparql` (SELECT/ASK/CONSTRUCT/DESCRIBE). Updates are intentionally not supported by the skill — use `curl` explicitly if you need to mutate.

### OpenSearch — search & enriched indices

- **Endpoint:** `:9200`, OpenSearch 3.x with the security plugin and self-signed TLS (clients usually disable verification in dev)
- **Contents:** GrimoireLab-enriched indices — `git`, `git_enriched`, `github`, `github_enriched`, `github_pull_requests_enriched`, `github_repo_enriched`, etc.
- **Use it for:** Full-text search across commits/issues/PRs, aggregations over enriched fields, log-like investigations
- **Skill:** `query-opensearch` (health, indices, count, search DSL). Use `--size 0` for aggregation-only queries; `terms` aggs on strings need `.keyword`.

### Pulling them together

The interesting thing this kit is meant to demonstrate is **pulling variables from across all three stores into a single coherent view**. For example: a per-repository panel that combines Neo4j contributors, SPARQL metadata, and OpenSearch commit-frequency aggregates.

---

## Reference views (what ships in the template)

The three views in `dashboard/src/routes/` are intended both as working tools and as **pattern examples**:

| Route | Pattern it demonstrates |
|---|---|
| `/graph` | Reactive D3 visualisation; temporal animation; full-page layout (no AppShell); typed API client + MSW fixture |
| `/pipeline`, `/pipeline/[id]` | List + detail pattern; status badges; tables; AppShell wrap |
| `/health` | Card grid; mixed-status surfaces (containers, endpoints, smoke tests); AppShell wrap |

Together they cover the three layout archetypes a downstream user is most likely to need (full-page canvas, list/detail, card grid).

---

## How a downstream user is expected to extend this

When suggesting changes, keep these likely user journeys in mind:

1. **Swap MSW fixtures for a real backend** — they'll add a SvelteKit server endpoint that proxies to Neo4j/SPARQL/OpenSearch (so credentials stay server-side), and point `VITE_API_BASE_URL` at it. The MSW mocks remain for local dev without infra.
2. **Add a new domain-specific view** — they'll follow `.claude/SKILLS.md §2` (Add a new page) using AppShell, then add API types + client + MSW handler per `.claude/SKILLS.md §1`.
3. **Re-skin the design** — they'll edit the `@theme` block and `:root` tokens in `dashboard/src/app.css`. The `--op-*` token convention exists so the rename to their own brand is a one-file change.
4. **Add a new data store** — e.g. PostgreSQL, a vector DB. They should follow the skill pattern (`.claude/skills/query-*`) and the `.env.example` pattern.

If a change makes one of these journeys harder (e.g. couples the design system to a specific backend, hardcodes `--op-*` colour names into business logic), flag it.

---

## Non-goals

- This template is **not** an Open Pulse SDK or query-builder library — it demonstrates how to consume the platform, not how to abstract it.
- The mock backend (`src/mock-unified-backend/main.py`) exists for devcontainer / direct-backend testing only. It is **not** a reference implementation of the real backend's API surface — the real backend lives in a separate Open Pulse repo.
- No test runner is wired up; do not add one without explicit ask. Type-checking (`npm run check`) is the current quality gate.

---

## Pointers

- **Conventions, stack, what-not-to-do:** `CLAUDE.md` (repo root)
- **Concrete how-tos** (add page, add endpoint, add token, debug graph): `.claude/SKILLS.md`
- **Visual rules**: `dashboard/DESIGN_SYSTEM.md`
- **Backend endpoints & credentials**: `.env.example`
- **Data store query skills**: `.claude/skills/query-{neo4j,sparql,opensearch}/SKILL.md`
- **Devcontainer / mock backend**: `.devcontainer/`, `src/mock-unified-backend/`, `tools/docker/mock-unified-backend/`
