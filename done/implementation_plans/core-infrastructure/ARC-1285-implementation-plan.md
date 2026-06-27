# ARC-1285 Bootstrap Web Server and Serve Local UI - Implementation Plan

**Issue:** `issues/core-infrastructure/ARC-1285-bootstrap-web-server.md`
**Completion Summary:** `task-completions/ARC-1285-COMPLETION-SUMMARY.md` (TBD)
**Approach:** Node.js/Express backend + React/Vite frontend, monorepo layout
**Owner:** build
**Date:** 2026-06-27

## Scope & Alignment

Covers:
- Monorepo scaffold (root package.json, workspaces)
- Express server (`packages/server`) serving static UI and API skeleton
- React/Vite frontend (`packages/ui`) with a placeholder runbook dashboard route
- Dev tooling: concurrently run server + Vite dev server with API proxy
- Health-check endpoint and configurable port

Maps to AC:
- AC1: `npm start` boots server → `http://localhost:{port}` loads UI without errors
- AC2: root URL renders runbook dashboard component

## Assumptions & Dependencies

- Node.js ≥ 18 available locally
- No existing application code — full greenfield
- Port defaults to 4000; overridable via PORT env var
- `setup.env` sourcing handled by caller — server reads process.env

## Implementation Steps

### Step 1: Scaffold monorepo root

**Files:** `package.json`, `.gitignore`, `README.md`
**Action:** Create root `package.json` with `workspaces: ["packages/*"]`, scripts: `dev`, `build`, `start`. Add `.gitignore` covering `node_modules`, `dist`, `.env`.
**Verification:** `npm install` at root resolves both workspaces without error.

### Step 2: Create Express server package

**Files:** `packages/server/package.json`, `packages/server/src/index.ts`, `packages/server/src/routes/health.ts`, `packages/server/tsconfig.json`
**Action:** Express app listens on `process.env.PORT ?? 4000`. Registers `/api/health` GET route returning `{ status: "ok" }`. In production, serves `packages/ui/dist` as static root with `express.static`. Falls back to `index.html` for SPA routing (catch-all).
**Verification:** `curl http://localhost:4000/api/health` returns `{"status":"ok"}`.

### Step 3: Create React/Vite frontend package

**Files:** `packages/ui/package.json`, `packages/ui/vite.config.ts`, `packages/ui/index.html`, `packages/ui/src/main.tsx`, `packages/ui/src/App.tsx`, `packages/ui/src/pages/Dashboard.tsx`
**Action:** Vite + React + TypeScript scaffold. `App.tsx` renders a `<Dashboard />` component at route `/`. `Dashboard.tsx` renders a placeholder heading `"Runbook Dashboard"` — no data logic yet (that is ARC-1313). Vite dev config proxies `/api` to `http://localhost:4000`.
**Verification:** `npm run dev` in `packages/ui` serves app at `http://localhost:5173`; root URL renders "Runbook Dashboard" heading.

### Step 4: Wire dev mode with concurrently

**Files:** root `package.json` (scripts), `packages/server/package.json` (dev script)
**Action:** Root `dev` script uses `concurrently` to run `packages/server` in watch mode (ts-node-dev or tsx watch) and `packages/ui` Vite dev server simultaneously. Server starts first (delay 500ms) so API is available when UI starts.
**Verification:** `npm run dev` at root starts both processes; browser at `http://localhost:5173` loads dashboard; `http://localhost:4000/api/health` responds.

### Step 5: Production build and static serving

**Files:** root `package.json` (build script), `packages/server/src/index.ts`
**Action:** Root `build` script runs `packages/ui` Vite build then `packages/server` TypeScript compile. Root `start` script runs the compiled server. Server's static middleware path resolves to `packages/ui/dist` relative to server dist output.
**Verification:** `npm run build && npm start` → `http://localhost:4000` serves the built React app; root URL renders "Runbook Dashboard".

### Step 6: Write unit tests for server startup and health route

**Files:** `packages/server/src/__tests__/health.test.ts`
**Action:** Use `supertest` + `vitest` (or Jest) to test: (a) GET `/api/health` returns 200 with `{ status: "ok" }`; (b) server starts and listens on configured port without throwing.
**Verification:** `npm test` in `packages/server` passes with 0 failures.

## Testing & Validation

- Unit: `npm test` in `packages/server` — health route test passes
- Integration (AC1): `npm run build && npm start` → open `http://localhost:4000` → UI loads, no console errors
- Integration (AC2): root URL renders heading "Runbook Dashboard" in browser
- Dev mode smoke: `npm run dev` → both processes start; Vite proxy forwards `/api/health` to Express

## Risks & Open Questions

- `opencode` CLI not involved in this story — no subprocess risk here
- Static file path resolution after TypeScript compile (server dist) vs UI dist must be validated in Step 5; adjust relative path if needed
- Port conflict on 4000: document in README that PORT env var overrides default
