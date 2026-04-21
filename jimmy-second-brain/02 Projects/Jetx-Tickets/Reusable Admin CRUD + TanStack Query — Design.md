# Reusable Admin CRUD + TanStack Query — Design

  

## Context

  

The `/admin/users` and `/admin/categories` pages in `jetx-tickets-web` currently render static tables with mock data and no edit experience (the "Add category" button drops a row with default values and no way to edit). We need real Create/Read/Update/Delete for both, and the JetX design handoff calls for additional admin-style resources later (SLA policies, potentially stations, future apps). Building the two pages twice, differently, is wasteful.

  

This spec introduces a small set of composable primitives — a data-layer hook pair plus two UI components — that any admin-resource page can compose into a full CRUD experience in 30–50 lines of page code. The primitives are designed with clean generic interfaces so they can be promoted to a shared `@jetx/svelte-admin` package later, but live in-app for v1.

  

## Goal

  

Ship Categories and Users admin pages with real CRUD (create/read/update/delete, optimistic updates, validation, toasts) using shared primitives that work for any resource matching a `{ id: string; ... }` shape.

  

## Tech Choices

  

| Concern | Choice | Why |

|---|---|---|

| Data layer | `@tanstack/svelte-query` v5 | Caching, optimistic updates, loading/error states, polling — all first-class. Svelte 5 runes compatible. |

| UI primitives | `shadcn-svelte` (init in this PR) | `Dialog`, `AlertDialog`, `Input`, `Select`, etc. — we own the component source (in `src/lib/components/ui/`), fully themeable to JetX tokens. |

| Validation | `zod` | Client-side schema + form errors. Types inferred via `z.infer`. |

| Backend for v1 | In-memory SvelteKit `+server.ts` routes at `/api/*` | Real HTTP round-trip without waiting on the .NET API. Swap via `API_BASE_URL` env var when .NET lands; delete `src/routes/api/` folder. |

| State management | Query cache only (no Svelte stores for list data) | Single source of truth, no invalidation drift. |

  

## Architecture

  

Four thin layers, top-down:

  

```

┌────────────────────────────────────────────────────────────────┐

│  routes/admin/*/+page.svelte                                   │ ← page glue (~30–50 LOC)

│    uses createResourceQuery + <ResourceTable>                  │

│          createResourceMutations + <ResourceDialog>            │

├────────────────────────────────────────────────────────────────┤

│  lib/components/admin/*.svelte                                 │ ← reusable UI primitives

│    ResourceTable, ResourceDialog, ResourceField, DeleteConfirm │

├────────────────────────────────────────────────────────────────┤

│  lib/api/resource.ts                                           │ ← data layer

│    createResourceQuery<T>, createResourceMutations<T>          │

│    (wraps @tanstack/svelte-query)                              │

├────────────────────────────────────────────────────────────────┤

│  routes/api/<name>/+server.ts  (dev-only)                      │ ← mock backend

│    in-memory store, REST verbs                                 │

└────────────────────────────────────────────────────────────────┘

```

  

### Layer responsibilities

  

**`lib/api/resource.ts`** — exports two factory functions:

- `createResourceQuery<T>({ name, endpoint? })` → returns a Svelte-query result object `{ data, isLoading, isError, error, refetch }` that fetches `GET {endpoint}` (defaults to `/api/{name}`) and caches under query key `[name]`.

- `createResourceMutations<T>({ name, endpoint? })` → returns `{ create, update, remove }`. Each is a `createMutation` result. Mutations run optimistically against the cache entry keyed by `[name]`, roll back on error, invalidate on success, and push a toast on both outcomes.

  

Zero imports from app-specific code. No `Category` / `User` references. Fully generic over `T`.

  

**`lib/components/admin/`** — four components:

  

- `ResourceTable<T>` — props: `{ items: T[], columns: Column<T>[], rowActions?: Snippet, loading?: boolean, empty?: Snippet }`. Renders the `.admin-table` class from the JetX design system. **No built-in sort / filter / pagination in this spec's v1** — admin lists are small (<50 rows) and these features are deferred to a follow-up spec when a ticket list or similar large-dataset use-case needs them.

- `ResourceDialog<T>` — props: `{ open: boolean, mode: 'create' | 'edit', schema: ZodSchema<T>, fields: Field[], initialValues?: Partial<T>, onSave: (values: T) => Promise<void>, onClose: () => void }`. Renders fields based on declarative `fields` config. Validates on submit via the zod schema. Catches save errors and either (a) highlights field-specific issues if error payload is `{ field, message }[]`, or (b) shows a general error banner.

- `ResourceField` — internal component, not exported from the barrel. Switches on `type` and renders the right shadcn primitive (Input / Textarea / Select / Switch / number input).

- `DeleteConfirm` — thin wrapper around shadcn `AlertDialog`; props `{ open, onConfirm, onCancel, resourceName, itemName }`. Default body: *"Delete {itemName}? This can't be undone."*

  

**`routes/api/<name>/+server.ts`** — mock backend for dev only:

  

```ts

// routes/api/categories/+server.ts

let store = [ /* seed data */ ];

export const GET    = () => json(store);

export const POST   = async ({ request }) => { /* ... */ };

export const PUT    = async ({ request, url }) => { /* ... */ };

export const DELETE = async ({ url }) => { /* ... */ };

```

  

Module-level array — resets on dev-server restart (fine for development). Each resource gets its own `+server.ts`.

  

### Field type support (v1)

  

| `type` | shadcn primitive | Validation hint |

|---|---|---|

| `text` | `Input` | `z.string().min(...).max(...)` |

| `textarea` | `Textarea` | `z.string().max(...)` |

| `number` | `Input type="number"` | `z.coerce.number().int().min()....` |

| `select` | `Select` | `z.enum([...])` or `z.string()` with validated options list |

| `boolean` | `Switch` | `z.boolean()` |

  

Color picker = a `select` over 8 preset swatches per the JetX design handoff. No real color picker component in v1.

  

**Out of scope for v1:** `date`, `datetime`, `multi-select`, `image-upload`, `password`, `email`-as-distinct-type.

  

## Data Flow — Example (Categories page)

  

```svelte

<script lang="ts">

  import { z } from 'zod';

  import { createResourceQuery, createResourceMutations } from '$lib/api/resource';

  import ResourceTable from '$lib/components/admin/ResourceTable.svelte';

  import ResourceDialog from '$lib/components/admin/ResourceDialog.svelte';

  import DeleteConfirm from '$lib/components/admin/DeleteConfirm.svelte';

  

  const COLORS = ['#FF015C', '#1E7CD6', '#F5A623', '#1E9E6A', '#9333EA', '#06B6D4', '#EC4899', '#64748B'] as const;

  

  interface Category {

    id: string; name: string; description: string;

    slaHours: number; color: typeof COLORS[number]; isActive: boolean;

  }

  

  const schema = z.object({

    id: z.string(),

    name: z.string().min(1, 'Required').max(80),

    description: z.string().max(280).default(''),

    slaHours: z.coerce.number().int().min(1).max(720),

    color: z.enum(COLORS),

    isActive: z.boolean().default(true)

  });

  

  const FIELDS = [

    { name: 'name',        type: 'text',     label: 'Name',          required: true },

    { name: 'description', type: 'textarea', label: 'Description' },

    { name: 'slaHours',    type: 'number',   label: 'SLA (hours)',   required: true },

    { name: 'color',       type: 'select',   label: 'Color',         options: COLORS.map(c => ({ value: c, label: c })) },

    { name: 'isActive',    type: 'boolean',  label: 'Active' }

  ] as const;

  

  const columns = [

    { key: 'name',        label: 'Name' },

    { key: 'description', label: 'Description' },

    { key: 'slaHours',    label: 'SLA' },

    { key: 'isActive',    label: 'Active' }

  ];

  

  const q = createResourceQuery<Category>({ name: 'categories' });

  const m = createResourceMutations<Category>({ name: 'categories' });

  

  let dialogOpen = $state(false);

  let editing = $state<Partial<Category> | null>(null);

  

  function openCreate()       { editing = null; dialogOpen = true; }

  function openEdit(c: Category) { editing = c; dialogOpen = true; }

  async function handleSave(values: Category) {

    if (editing?.id) await $m.update.mutateAsync(values);

    else             await $m.create.mutateAsync(values);

    dialogOpen = false;

  }

  async function handleDelete(c: Category) {

    await $m.remove.mutateAsync(c.id);

  }

</script>

  

<ResourceTable items={$q.data ?? []} {columns} loading={$q.isLoading} onEdit={openEdit} onDelete={handleDelete}>

  {#snippet header()}<button class="btn btn-primary" onclick={openCreate}>Add category</button>{/snippet}

</ResourceTable>

  

<ResourceDialog

  bind:open={dialogOpen}

  mode={editing?.id ? 'edit' : 'create'}

  {schema} fields={FIELDS}

  initialValues={editing ?? { isActive: true, slaHours: 48, color: COLORS[0] }}

  onSave={handleSave}

/>

```

  

Page LOC: ~50. Same structural shape for Users.

  

## Error Handling

  

### Validation errors

Zod validation runs inside `ResourceDialog.onSave()` **before** calling `onSave`. Errors render under each field via a per-field errors map. Submit button is disabled until all required fields pass. No server round-trip is made when local validation fails.

  

### Server errors

`createResourceMutations` catches any non-2xx response and forwards the parsed JSON body to the caller via `onError`. Expected shape:

  

```ts

// success

{ id: 'x', name: '...', ... }

// server error

{ error: 'VALIDATION' | 'NOT_FOUND' | 'INTERNAL', message: string, fields?: Record<string, string> }

```

  

If `fields` is present, `ResourceDialog` highlights each one. Otherwise a single error banner renders above the form.

  

### Optimistic updates + rollback

Every mutation implements TanStack Query's `onMutate` / `onError` / `onSettled` lifecycle:

  

- `onMutate`: snapshot current cache, apply the change locally, return snapshot.

- `onError(err, vars, snapshot)`: restore snapshot, push `error` toast.

- `onSettled`: `invalidateQueries({ queryKey: [name] })` to reconcile with server truth.

- `onSuccess`: push `success` toast.

  

### Network errors

Same path as server errors; `onError` distinguishes via `err.name === 'NetworkError'` and shows a retry-able toast.

  

## Extraction Path (future `@jetx/svelte-admin` package)

  

All code to be extracted lives under two folders with **zero app-specific imports**:

- `src/lib/api/resource.ts`

- `src/lib/components/admin/*.svelte`

  

When the package is extracted:

- Copy those two folders to a new workspace package.

- Peer-depend on `@tanstack/svelte-query`, `zod`, `svelte` (≥5).

- Document via `README.md` + a Storybook-or-equivalent demo app.

- Consuming apps import `{ createResourceQuery, createResourceMutations }` from the package and `<ResourceTable>` / `<ResourceDialog>` likewise.

  

No change required to the API surface when extracting — this is a goal, not a future refactor.

  

## Testing

  

**Unit (Vitest)** — `src/lib/api/resource.test.ts`:

- `createResourceQuery` calls `GET /api/{name}` and caches the result.

- `createResourceMutations.create` PUTs, optimistically adds to cache, rolls back on 500.

- `createResourceMutations.update` optimistic patch, rollback on error.

- `createResourceMutations.remove` optimistic delete, rollback on error.

  

**Component (Vitest + @testing-library/svelte)** — `src/lib/components/admin/ResourceDialog.test.ts`:

- Renders each field type correctly.

- Invalid input shows field-level errors and disables submit.

- Valid input calls `onSave` with parsed values.

- Server `fields` errors re-render as field-level errors after save attempt.

  

**E2E (Playwright)** — `tests/e2e/admin-categories.spec.ts`:

- Load `/admin/categories`.

- Click "Add category" → dialog opens.

- Fill fields, submit → new row appears in table.

- Edit that row → dialog opens with values prefilled, change and save → row updates.

- Delete that row → confirm dialog → row disappears.

  

All three run in CI (`npm test` + `npm run test:e2e`).

  

## Definition of Done

  

- `@tanstack/svelte-query` and `zod` installed; `QueryClient` provisioned in root layout.

- `shadcn-svelte` initialized; `Dialog`, `AlertDialog`, `Button`, `Input`, `Textarea`, `Select`, `Label`, `Switch` added; each one themed to JetX tokens.

- `src/lib/api/resource.ts` implemented and unit-tested.

- `src/lib/components/admin/{ResourceTable,ResourceDialog,ResourceField,DeleteConfirm}.svelte` implemented.

- `src/routes/api/categories/+server.ts` and `src/routes/api/user-roles/+server.ts` implemented (in-memory stores).

- `src/routes/admin/categories/+page.svelte` and `src/routes/admin/users/+page.svelte` re-wired to the new primitives. Mock hand-rolled tables replaced.

- E2E Playwright test for the categories flow green.

- Visual parity with the JetX design handoff — button colors, border radii, spacing match.

- No `console.error` output during any admin flow (including optimistic rollback).

  

## Open Questions (answer before or during implementation)

  

1. `createResourceQuery` defaults the endpoint to `/api/{name}` — do we need a `baseUrl` env var now, or save that for the .NET-swap PR? (Recommend: env var now so the data layer is already configurable.)

2. Do we need an `X-Correlation-Id` header on mutations for debuggability? (Probably yes in prod; defer until observability spec.)

3. Should the `id` of new rows be generated client-side (uuid/ulid) for optimistic inserts, or server-returned? (Recommend server-returned, match the planned `Guid.CreateVersion7()` on the .NET side. This means optimistic "create" shows a pending row with a temp id until the server responds.)