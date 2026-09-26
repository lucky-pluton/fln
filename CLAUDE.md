# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

FLN Assessment & Personalized Worksheet Platform — an AI-driven system that assesses each child's foundational **Mathematics** level on a **93-level curriculum running from the Balvatika band (Preschool 1–3, ages 3–6) through Class 4 (ages 9–10)**, generates level-personalized printable worksheets, ingests scanned answers, evaluates them, and rolls data up a 7-role national hierarchy. See [SRS.md](SRS.md) (authoritative spec; §1.8 for the level framework), [PRD.md](PRD.md) (product framing), [AUDIT.md](AUDIT.md) (current code health), [MIGRATION_PLAN.md](MIGRATION_PLAN.md) (target structure), [ARCHITECTURE.md](ARCHITECTURE.md) (system shape), [PIPELINES.md](PIPELINES.md) (the four end-to-end flows), and [AGENT_ARCHITECTURE.md](AGENT_ARCHITECTURE.md) (the AI/LLM surface).

**Stack:** React 19 + Vite + Tailwind (frontend), Node/Express + TypeScript (backend), Google Gemini (LLM), Python (PDF rasterization only — the older `ai-services/` LLM pipeline is retained for reference and is not invoked by the backend). The repo is an **npm-workspaces monorepo**: `frontend/`, `backend/`, `ai-services/` (see Layout).

## ⚠ Critical thing to understand before editing

**The app uses the real backend only. No mock backend should run.**

- The source of truth for all `/api/*` calls is the Express server at `backend/`. It implements the SRS: generation locks, defaulter escalation, Aadhaar masking, role-scoping, real Gemini, real PDF generation. It boots on `:3000` and is verified to answer the API directly via curl. (`ai-services/` contributes only PDF rasterization to the scan path — see [PIPELINES.md](PIPELINES.md).)
- The legacy in-browser mock (`frontend/src/mock/fetchInterceptor.ts`, `frontend/src/mock/dbStore.ts`) and the `setupFetchInterceptor()` call that previously lived at `frontend/src/main.tsx:8` are **deleted**. The frontend must talk to the real backend — no fallback, no parallel store.
- Any leftover `frontend/src/mock/**` files, the `public/mock/*.json` dataset, the `frontend/src/constants.ts` hardcoded seed, and `frontend/src/utils/levelGenerator.ts` (a byte-identical duplicate of `backend/src/levelGenerator.ts`) are slated for deletion — never reference them in new code.
- When asked to change "backend behavior," edit `backend/src/**` only. Do not reintroduce a second copy of business logic in the frontend.

See AUDIT.md for the full cleanup list and MIGRATION_PLAN.md for the deletion sequence.

## Layout

```
fln/                          # npm-workspaces monorepo root (package.json = workspaces)
├── frontend/                 # @fln/frontend — React + Vite app (talks to real backend on :3000 via proxy)
│   ├── index.html  vite.config.ts  tsconfig.json  package.json
│   ├── public/worksheets/    # worksheet HTML templates — ALSO read by the backend (Puppeteer)
│   └── src/
│       ├── main.tsx          # React entry; NO fetch interceptor
│       ├── App.tsx           # top-level views + role switch
│       ├── mock/             # deleted (was: fetchInterceptor.ts, dbStore.ts) — do not recreate
│       ├── constants.ts      # 763 ln of hardcoded seed data — slated for deletion; do not extend
│       ├── utils/levelGenerator.ts   # byte-identical duplicate of backend/src/levelGenerator.ts — slated for deletion
│       └── components/       # 24 components; RoleDashboards.tsx (2702 ln) + PanelViews.tsx (1455 ln) are god-files
├── backend/                  # @fln/backend — REAL Node/Express API (API only; no Vite)
│   ├── package.json  tsconfig.json  .env.example
│   ├── fln-backend/          # standalone 1-59 worksheet renderer (Puppeteer) the API calls over HTTP
│   ├── data/db.json          # the JSON-file "database" (not MongoDB despite comments)
│   └── src/                  # index.ts, db.ts, gemini.ts, paperGenerator.ts, ...
│       ├── config/curriculumMap.ts    # the 93-level ⇄ Concept ID registry (S1.1-S7.18)
│       └── competencyPrerequisites.ts # the typed prerequisite DAG — generated from the research
├── ai-services/              # scripts/pdf_rasterize.py is the ONLY script the backend invokes;
│                             # run_pipeline.py + scripts/0..3 + prompts/ are the legacy
│                             # class/phrase data model — retained for reference, NOT invoked
├── Research/                 # the 93-level framework research — the source of the taxonomy
└── docs/                     # teacher workflow docs — describe the SERVER's behavior
```

## Commands

Run from the repo root (npm workspaces). One install covers both packages:

```bash
npm install
npm run dev:backend    # tsx backend/src/index.ts — real API on :3000  (REQUIRED; start this first)
npm run dev:frontend   # Vite dev server on :5173, proxies /api -> http://localhost:3000
npm run build          # builds frontend (vite) then backend (esbuild -> backend/dist/server.cjs)
npm run lint           # tsc --noEmit across workspaces (type-check only; there are no unit tests)
```

The app you see is the **frontend on :5173** talking to the **real backend on :3000**. Start the backend first; the frontend's Vite proxy (`vite.config.ts`) forwards `/api/*` to it. There is no in-browser mock — never add one. In production the backend serves `frontend/dist` (`FRONTEND_DIST_DIR`).

Demo login (e.g. `gps-mt-001.t01@fln.org`): **ask the team for the demo password** (do not hardcode or paste it into docs/commits). `python` on PATH is needed only for `ai-services/scripts/pdf_rasterize.py` (the scan path); the backend invokes it from `ai-services/` (`AI_SERVICES_DIR` override).

## Environment

- `GEMINI_API_KEY` — required for real AI calls (`backend/src/gemini.ts:9`). Each AI path has a deterministic non-AI fallback, so the server runs without it.
- `PORT` (default 3000), `CHROME_EXECUTABLE_PATH` (Puppeteer PDF generation).
- `AI_SERVICES_DIR` (defaults to `../ai-services`), `WORKSHEET_ASSETS_DIR` (defaults to `../frontend/public/worksheets`), `FRONTEND_DIST_DIR` (prod static serve).
- Copy `backend/.env.example`. Never commit real keys.

## Conventions & gotchas

- **Match the surrounding file's style** — this codebase was AI-generated by non-devs; consistency varies. Don't reformat wholesale.
- **Don't touch `frontend/vite.config.ts` HMR/watch settings** — they're intentionally set for the AI Studio environment (comment in file).
- Magic thresholds recur across many files: max level `59`, certification `currentLevel >= 5`, score bands `80/60`. If you change one, grep for the others (they are NOT centralized). AUDIT §3.3 lists locations.
- **`59` is the renderable cap, not the curriculum size.** The taxonomy is 93 levels (SRS §1.8); the worksheet renderer in `backend/fln-backend/` still implements the retired 1–59 space, so every placement/promotion path caps at 59 pending the 59→93 migration. Do not "fix" a level by raising the cap, and do not treat 59 as the number of levels that exist. See `Research/fln_59_to_93_crosswalk.PROPOSED.md`.
- **The level framework is not a lookup table and the count is not fixed.** Balvatika (Preschool 1–3, levels 1–27) is the anchor band and is reported as ONE band, not Balvatika 1/2/3. Concepts (`S1.1`–`S7.18`) are immutable; level numbers are not. Read counts from `backend/src/config/curriculumMap.ts`, never hardcode 93.
- **Prerequisite edges are typed, and only one type is a dependency.** `backend/src/competencyPrerequisites.ts` reproduces only `prereq` edges from `Research/fln_level_networks.md` Part 2. `sequence` (teaching order) and `parallel` (co-equal) carry no inference and must never be consumed. The graph is validated as a DAG at start-up. Whether a concept's *multiple* prerequisites combine as AND or OR is **undecided** — don't assume a reading. Full rules: SRS §1.8.3.
- **`npm run check:level-notation-drift`** fails if the computed L↔S mapping in `frontend/src/data/skillProgressionMap.ts` drifts from `Research/fln_L_to_S_crosswalk.json`. Run it after touching either.
- The "MongoDB collections" in `backend/src/db.ts` are a single `backend/data/db.json` rewritten in full per mutation — not concurrency-safe.
- **Auth has been hardened — this note used to say otherwise; corrected 2026-08-31.** `backend/src/auth.ts` issues and verifies real signed JWTs (`jsonwebtoken`, `JWT_SECRET`), and its own code comment is explicit: "There is deliberately NO role synthesis from the email/prefix: only real, seeded users with a valid signed token authenticate." Login (`backend/src/routes/auth.ts`) does a real `bcrypt.compare()` against the stored hash, not a length check. `/api/reset` requires an authenticated superadmin and is POST-only (a GET version with no auth was deliberately removed). Every route file imports `getAuthUser`/`canAccessStudent` from this same hardened module — there is no second, older implementation still live anywhere. Still worth normal scrutiny on a per-endpoint basis (e.g. `canAccessStudent`'s own comment notes admin-role scoping is broader than teacher/school scoping, tracked as a separate fix), but the plaintext-Bearer-email bypass this note used to describe no longer exists.
- **`npm run lint` is `tsc --noEmit` — it proves types compile, nothing about behavior.** There is no end-to-end or integration test suite, so a green lint tells you nothing about whether a flow works; verify by running the app (`npm run dev:frontend`) and/or curl against the real backend (`npm run dev:backend`). Narrow unit checks do exist and are worth running after touching the code they cover — `npm run test:answer-matching`, `test:error-classification`, `test:paper-lock`, `test:fingerprint`, `test:search-index`, `test:panel-isolation`, `test:detokenize` (see `backend/package.json`). Baseline note: two **pre-existing** type errors exist (`backend/src/index.ts:665`, `backend/src/paperGenerator.ts:233`) — don't add new ones.
- **`.gitignore` is authored as UTF-8/ASCII.** The original root `.gitignore` was UTF-16, which silently broke `node_modules/` matching. If you edit ignore files on Windows, confirm `file .gitignore` says ASCII/UTF-8, not UTF-16.

## Migration rules

The repo is being restructured per [MIGRATION_PLAN.md](MIGRATION_PLAN.md). While that is in progress, follow these hard rules:

- **Current phase:** _real-backend cutover done (2026-09-02)._ The frontend's `fetch()` interceptor, `src/mock/**`, `public/mock/*.json`, the `constants.ts` seed, and the duplicated `utils/levelGenerator.ts` are all slated for deletion in the cleanup phase. Vite proxies `/api/*` to the Express backend on `:3000`. Do not reintroduce any in-browser mock.
- **Cut over so far:** _all routes._ Every `/api/*` call from the UI reaches the real Express backend. There is no parallel mock store and no fallback path. When you add a new route, add it to `backend/src/**` only.
- **No opportunistic refactors during cleanup.** Do not split the god-files (`RoleDashboards.tsx`, `PanelViews.tsx`) or dedupe the Teacher/Volunteer dashboards as a side effect of removing the mock. Relocation and restructuring are separate phases (see the plan) — keep each change one kind of change.
- Do not re-add the mock interceptor to `frontend/src/main.tsx` under any flag, env var, or "dev-only" mode. If the backend is unreachable, the right fix is to fix the backend, not to fall back to a mock.

## Where new code goes

- **Backend domain logic** → `backend/src/modules/<domain>/` (auth, students, worksheets, evaluation, governance, …). These modules don't exist yet — `backend/src/index.ts` is still one 1580-line file; add to the matching area until the module split phase. One home per domain — don't scatter.
- **Shared thresholds/constants** (`59`, cert `>=5`, score bands, timing windows) → a `shared/` location (not yet created). Never re-inline a new magic number; reference the shared value.
- **Never add to `frontend/src/mock/`** — it is being deleted. New data-fetching goes through the real API (via the planned `apiClient`), not the interceptor.
- **No business logic in React components** — scoring, level assignment, locks, certification, ID generation belong on the backend. This is an existing anti-pattern being unwound, not a pattern to copy.
- Keep answer keys and student PII **out of the frontend bundle** (both currently leak — AUDIT §2.13, §3.2).
