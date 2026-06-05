# CLAUDE.md — pulseWebKit (Open Pulse Dashboard)

Agent instructions for this repository. Read this first, then `.claude/PROJECT.md` for the broader mission, `.claude/SKILLS.md` for concrete task recipes, and the `frontend-dev` skill before touching any UI.

---

## What this is

**pulseWebKit** (working name: *pulseNext*) — a SvelteKit starter that will be published as a **GitHub template repo** so researchers and developers can fork it to build their own dashboards or interactive sites on top of the **Open Pulse** platform.

The Open Pulse platform (Neo4j + Oxigraph + OpenSearch) is the data layer; this kit demonstrates how to pull, type, mock, and visualise variables from it. See `.claude/PROJECT.md` for the full mission, data-source overview, and template-extension guidance.

---

## Reference view

- **Graph Explorer** (`/graph`) — D3 force-directed graph of Neo4j data (repos, contributors, commits, orgs, PRs) with temporal animation

All API calls are mocked in-browser using MSW during local development. No backend or Docker required.

---

## Stack

| Layer | Technology |
|---|---|
| Framework | SvelteKit 2, Svelte 5 (runes mode) |
| Build | Vite 8 |
| Styling | Tailwind CSS v4 (`@theme` based) |
| Visualization | D3 v7, layerchart |
| API mocking | Mock Service Worker (MSW) 2 |
| Types | TypeScript 6, strict mode |

---

## Dev workflow

All commands run from the `dashboard/` directory:

```bash
npm install         # install deps
npm run dev         # dev server → http://localhost:5173
npm run build       # production build
npm run preview     # serve production build
npm run check       # TypeScript + Svelte type check (run before every commit)
```

CI runs `npm run check && npm run build` on every push and PR.

---

## UI verification — REQUIRED for frontend work

This repo ships an MCP-server config for **Playwright MCP** (`@playwright/mcp`) in `.mcp.json` at the repo root, enabled via `.claude/settings.json` (`enabledMcpjsonServers: ["playwright"]`).

**Rules for any change that touches Svelte components, routes, CSS tokens, or visual behaviour:**

1. Start the dev server (`cd dashboard && npm run dev`) — the MCP server is locked to `http://localhost:5173` and `http://localhost:4173`.
2. Drive the affected page through the Playwright MCP browser tools — navigate, click, fill, snapshot.
3. Take a screenshot and confirm it visually matches the design intent (see the `frontend-dev` skill).
4. Watch the browser console for runtime errors and network failures.

`npm run check` is a type-correctness gate, **not** a feature-correctness gate. Do not claim UI work is done on type-check alone.

---

## Repository layout

```
open-pulse-sdsc/
├── dashboard/
│   ├── src/
│   │   ├── app.css                        # design tokens + global styles
│   │   ├── fonts.d.ts                     # ambient type declarations for font packages
│   │   ├── routes/
│   │   │   ├── +layout.svelte             # root layout; font imports; MSW init
│   │   │   ├── +page.svelte               # redirects → /graph
│   │   │   └── graph/+page.svelte         # graph explorer (main page)
│   │   └── lib/
│   │       ├── api/
│   │       │   ├── types.ts               # all API types (source of truth)
│   │       │   ├── client.ts              # api.graph / api.metadata / etc.
│   │       │   └── mocks/
│   │       │       ├── handlers.ts        # MSW request handlers + fixture data
│   │       │       └── browser.ts         # MSW browser worker setup
│   │       └── components/
│   │           ├── layout/SiteHeader.svelte  # shared top bar
│   │           ├── layout/SiteFooter.svelte  # shared footer
│   │           ├── graph/ForceGraph.svelte   # D3 force simulation
│   │           └── graph/CommitTimeline.svelte # timeline scrubber
│   ├── static/
│   │   └── mockServiceWorker.js           # MSW service worker (auto-generated, do not edit)
│   └── package.json
└── .github/
    └── workflows/dashboard-ci.yml
```

---

## Architecture

### API layer

`client.ts` exposes a typed `api` object. Environment variable `VITE_API_BASE_URL` sets the base URL (defaults to `http://localhost:8000/api/v1`).

### Data sources (Open Pulse platform)

| Store | What's in it | How to query |
|---|---|---|
| **Neo4j** (`:7503` HTTP, `:7504` Bolt) | Property graph: repositories, contributors, commits, organisations, PRs | Skill `query-neo4j` |
| **Oxigraph / SPARQL** (`:7502`, Caddy basic-auth) | RDF metadata graph (~300k triples) | Skill `query-sparql` |
| **OpenSearch** (`:9200`, self-signed TLS) | GrimoireLab-enriched docs | Skill `query-opensearch` |

Browser code must never hit these stores directly — route requests through a SvelteKit server endpoint.

### MSW mocking

MSW intercepts `fetch` in-browser (dev only). To add a new endpoint:

1. Add the TypeScript interface to `src/lib/api/types.ts`
2. Add a method to `src/lib/api/client.ts`
3. Add fixture data and an `http.get(...)` handler to `src/lib/api/mocks/handlers.ts`

### Svelte 5 runes

This project uses **runes mode** everywhere:

```svelte
let count = $state(0);
let doubled = $derived(count * 2);
$effect(() => { console.log(count); });
```

Do not use `$:` reactive statements or `writable` stores.

### TypeScript

Strict mode is on. All props, function signatures, and API responses must be typed.

---

## Design system

Read the `frontend-dev` skill before writing any UI code. Key rules:

- All colors come from `--op-*` CSS custom properties or `bg-op-*` / `text-op-*` Tailwind classes
- Never hardcode hex in Svelte template markup — D3/SVG drawing code is the only exception
- Fonts: Space Grotesk for headings/wordmark, Switzer for all UI text, JetBrains Mono (`.mono`) for code/IDs
- Sharp corners everywhere (`rounded-none`) — buttons and badges use `rounded` (4px) only
- Brand blues only for interactive chrome; status colors only on badges/toasts

---

## Graph explorer specifics

- `ForceGraph.svelte` receives `nodes`, `edges`, and `currentDate`; shows only nodes/edges with `timestamp ≤ currentDate`
- The playback system in `graph/+page.svelte` advances `currentDate` at 10 simulation-days per real second
- Mock data spans 2026-02-08 → 2026-05-08
- D3 force simulation uses charge, link, center, and collision forces

---

## What not to do

- Do not use `$:` reactive statements or Svelte stores — runes only
- Do not hardcode hex values in Svelte template markup (D3/canvas drawing is exempt)
- Do not bypass `npm run check` — CI will catch type errors
- Do not claim UI work is done from `npm run check` alone — verify via Playwright MCP
- Do not add backend code to this repo — it is frontend-only
- Do not edit `static/mockServiceWorker.js` — it is generated by MSW
