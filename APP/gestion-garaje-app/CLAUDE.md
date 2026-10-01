# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # Next.js dev server on localhost:3000
npm run build    # production build (ESLint errors are ignored during builds, see next.config.ts)
npm run start    # serve the production build
npm run lint     # eslint . --ext .js,.ts,.jsx,.tsx
npm run lpd      # npm install --legacy-peer-deps — use this instead of plain `npm install`
```

There is no test setup in this repo (no test runner, no test files).

Requires a `.env` with `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` (gitignored). Both are read directly from `process.env` in `utils/supabase/*`.

## Architecture

Next.js 15 App Router + React 19 + Supabase (auth + Postgres) + shadcn/ui (new-york style, zinc base) + Tailwind. UI copy, domain vocabulary, and most comments are in Spanish — keep it that way. The domain is a garage/parking manager: **clientes** (customers) each own **coches** (cars), and a car may occupy a `numero_plaza` (parking spot, out of 36 total, hardcoded as `TOTAL_SPACES` in `services/dashboard.service.ts`).

### Three Supabase clients, chosen by execution context

- `utils/supabase/client.ts` — browser client. This is what almost every page uses, because nearly all pages are `"use client"` and query Supabase directly from `useEffect`. There is no API/data layer between components and the database.
- `utils/supabase/server.ts` — RSC/server-action client (async, cookie-backed). Used by `app/actions.ts`, `app/(protected)/layout.tsx`, and `app/auth/callback/route.ts`.
- `utils/supabase/middleware.ts` — `updateSession()`, called from `middleware.ts` on every non-static request. It refreshes the session and enforces the allowlist in `publicRoutes` (`/sign-in`, `/sign-up`, `/auth/callback`, `/forgot-password`); anything else redirects to `/sign-in` when there's no user. **Adding a new public page means adding it to that array**, not just creating the route.

Auth is enforced twice: once in middleware, again in `app/(protected)/layout.tsx` (session check + redirect). The protected layout is also where `GlobalContextProvider`, `ThemeProvider`, and the sidebar shell are mounted, so any page under `(protected)` can call `useGlobalContext()` for the current `user`/`username`.

### Route groups

- `app/(auth-pages)/` — centered, chrome-less layout; sign-in submits to the server actions in `app/actions.ts` via plain `<form action={...}>`. Errors/successes are propagated by `encodedRedirect()` (`utils/utils.ts`) as `?error=`/`?success=` query params and rendered by `components/form-message.tsx`.
- `app/(protected)/` — the app proper: `home` (dashboard), `clientes`, `coches`, `ajustes`, each CRUD area having `page.tsx` (list), `crear/`, and `editar/[id]/`.

### Per-feature `domain/` folders

Each CRUD area keeps its table definitions and validation next to the routes: `app/(protected)/<area>/domain/columns.tsx` and, for coches, `domain/carSchema.ts`. Columns are produced by `getColumns(supabase, router)` — a factory rather than a constant, because the "acciones" cell closes over the Supabase client and router to do inline edit/delete with `confirm()`. The generic `components/data-table.tsx` (TanStack Table) renders whatever those columns describe.

### Forms

react-hook-form + Zod via `zodResolver`, with shadcn `Form*` primitives. Two conventions coexist:

- **Shared form component** (the direction the code is moving): `features/clientes/components/ClientesForm.tsx` owns the `clientSchema`, the `ClientFormData` type, and the default values, and is driven by props (`title`, `onSubmit`, `submitButtonText`, `initialValues`, …). Both `clientes/crear` and `clientes/editar/[id]` render it. Note `editar/[id]/page.tsx` still contains a duplicated, now-dead copy of the schema — prefer the one exported from `ClientesForm`.
- **Inline form**: `coches/crear` and `coches/editar/[id]` build the form in the page, importing the schema from `domain/carSchema.ts`. The car schema is `.refine()`d at runtime against `spainMatriculaRegex` only when the "Spanish plate" checkbox is on, so the resolver is constructed from component state.

`features/` is the newer home for feature-scoped code (components/types); older code lives in `app/.../domain/`, top-level `components/`, and `services/`.

### Data conventions

- Types come from `utils/types/supabase.ts` (generated Supabase types). Use the helpers rather than hand-written interfaces: `Tables<"clientes">`, `TablesInsert<"coches">`, `TablesUpdate<"clientes">`. List pages typically widen them with a joined-display intersection, e.g. `Tables<"clientes"> & { coche?: string; matricula?: string }`.
- Relations are fetched with PostgREST embedding, e.g. `.select("*, coches (marca, modelo, matricula, numero_plaza)")`, then flattened in a `.map()` before hitting the table.
- Dates: form fields are `Date` objects; before writing to Supabase convert with `formatToLocalTimeZoneString()` (`utils/date-helper.ts`, default export), which formats in `Europe/Madrid`. Display formatting is ad-hoc `toLocaleDateString("es-ES")`.
- A client is "active" when `fecha_salida` is null (after the list mapping, null becomes the string `"-"`, so filters check both). See `ClientesFilter` in `features/clientes/types/types.ts`.
- `clientes/crear` redirects to `/coches/crear?cliente_id=<new id>` on success, and that page reads the param via `useSearchParams()` to preselect the owner.

### Dashboard

`services/dashboard.service.ts` is the only service-layer module: it issues parallel counted queries (`select("*", { count: "exact", head: true })`) plus a merged "recent activity" feed derived from `updated_at` vs `created_at` on both tables. `app/(protected)/home/page.tsx` caches the result in `localStorage` under `dashboardStats` and only re-renders when the fetched JSON differs.

## Notes

- `README.md` is the unmodified Supabase/Next.js starter readme and does not describe this app.
- User feedback is currently `alert()`/`confirm()`; errors are `console.error`'d. There is no toast system wired up.
- Indentation is inconsistent (4 spaces in app-authored files, 2 in files inherited from the starter such as `app/actions.ts` and `utils/supabase/server.ts`). Match the file you're editing. Prettier is installed but there is no config file and no format script.
