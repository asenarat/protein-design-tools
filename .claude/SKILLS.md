# SKILLS.md — Agent task reference

Concrete how-to guides for the most common tasks agents perform on this repo. Companion to `CLAUDE.md`.

---

## 1. Add a new API endpoint + MSW mock

**Step 1 — type** (`dashboard/src/lib/api/types.ts`):
```typescript
export interface MyNewResponse {
  field: string;
  count: number;
}
```

**Step 2 — client** (`dashboard/src/lib/api/client.ts`):
```typescript
myFeature: {
  list: () => get<MyNewResponse>('/my-feature')
}
```

**Step 3 — mock handler** (`dashboard/src/lib/api/mocks/handlers.ts`):
```typescript
const mockMyFeature: MyNewResponse = { field: 'example', count: 42 };

// add to the handlers array:
http.get(`${BASE}/my-feature`, () => HttpResponse.json(mockMyFeature)),
```

**Step 4 — use** in a Svelte component:
```svelte
<script lang="ts">
  import { api } from '$lib/api/client';
  import type { MyNewResponse } from '$lib/api/types';

  let data = $state<MyNewResponse | null>(null);

  $effect(() => {
    api.myFeature.list().then(r => { data = r; });
  });
</script>
```

---

## 2. Add a new page

1. Create `dashboard/src/routes/<slug>/+page.svelte`
2. Scaffold the file:

```svelte
<script lang="ts">
  import AppShell from '$lib/components/layout/AppShell.svelte';
</script>

<AppShell>
  <div class="mx-auto max-w-4xl space-y-6">
    <h1 class="text-xl font-semibold" style="color:var(--op-text)">Page Title</h1>
    <!-- content here -->
  </div>
</AppShell>
```

3. Add a nav entry in `dashboard/src/lib/components/layout/AppShell.svelte` — follow the existing pattern for the active-link highlight
4. Wire up API calls using the pattern in skill #1 if needed

---

## 3. Add a graph example query

Graph example queries live as `ExampleQuery` objects (inline in `graph/+page.svelte`). Each has:

```typescript
{
  id: 'unique-id',
  title: 'Human title',
  description: 'One-line description shown in sidebar',
  cypher: 'MATCH (n) RETURN n LIMIT 25',
  result: {
    nodes: [ { id: 'n1', label: 'Person', properties: { name: 'Alice' } } ],
    edges: [ { source: 'n1', target: 'n2', type: 'KNOWS', timestamp: '2026-03-01' } ]
  }
}
```

Rules:
- `id` must be unique across all queries
- `label` must be one of: `'Repository' | 'Person' | 'Commit' | 'Organisation' | 'PullRequest'`
- Edges without a `timestamp` are always visible; edges with one appear when `currentDate ≥ timestamp`
- Node colors are determined by `label` — see `DESIGN_SYSTEM.md` node color table

---

## 4. Add a new design token

When the design requires a colour or value that isn't in the token set:

1. **`dashboard/src/app.css` — `@theme` block** (Tailwind utility classes):
```css
--color-op-<name>: <hex>;
```

2. **`dashboard/src/app.css` — `:root` block** (CSS custom property for inline styles):
```css
--op-<name>: <hex>;
```

3. **`dashboard/DESIGN_SYSTEM.md`** — add a row to the token reference table

Do not add tokens to only one location — both must stay in sync.

---

## 5. Write a card / table / badge

Always copy the exact markup from `DESIGN_SYSTEM.md` component patterns rather than inventing new markup. Key patterns:

**Card:**
```svelte
<div class="rounded-xl p-4" style="background:var(--op-surface);border:1px solid var(--op-border)">
```

**Status badge (succeeded):**
```svelte
<span class="rounded-lg px-2.5 py-0.5 text-xs font-semibold"
  style="color:var(--op-green);background:rgba(144,202,66,0.12)">
  succeeded
</span>
```

**Section heading:**
```svelte
<h2 class="mono mb-3 text-[11px] font-medium uppercase tracking-wider"
  style="color:var(--op-text-muted)">
  Section Title
</h2>
```

---

## 6. Fix a TypeScript type error

- All API response shapes are defined in `dashboard/src/lib/api/types.ts` — check there first
- SvelteKit generates `$types` in `.svelte-kit/` — run `npm run check` to regenerate after adding routes
- If `import.meta.env.*` causes a type error, add the variable to `vite.config.ts` under `define` or use `?? fallback` inline
- Svelte 5 rune types: `$state<T>()`, `$derived<T>()` — explicit generic prevents inference failures on null initial values

---

## 7. Debug the D3 force graph

- `ForceGraph.svelte` re-runs the simulation whenever `nodes` or `edges` props change
- Node entry animation: new nodes get a random radial offset and animate to their simulated position via a spring (`alpha` decay)
- If nodes spawn at (0, 0) — the simulation hasn't warmed up yet; ensure `buildGraph()` is called inside `requestAnimationFrame`
- The `currentDate` prop controls visibility; filtering is done before passing to ForceGraph — do not filter inside the component
- D3 color constants live at the top of `ForceGraph.svelte` as `NODE_COLOR`; they must match the hex values in `DESIGN_SYSTEM.md`

---

## 8. Run type checking locally

```bash
cd dashboard
npm run check
```

This runs `svelte-kit sync` (generates `.svelte-kit/` types) then `svelte-check`. Fix all errors before pushing — CI runs the same command and will fail the build.
