# Plan

## Goal(s)

1. **Continue the already-working `i3` project** (cloned from
   `https://github.com/FDHKRsss/i3.git` into `i3_ref/`) and make it a **real,
   phone-friendly incident-reporting PWA**: when it is exposed at some address,
   a person opening it on a phone can take a **real photo** and have their
   **real GPS** captured — like the protoplast (`civil42pwa_ref/`, the read-only
   reference clone of `https://github.com/rzymek/civil42pwa-public.git`), not a
   mocked desktop-only seed.
2. **Persist everything in PostgreSQL.** Photos must be stored in Postgres
   (decided *how* — see ARCHITECTURE "Database schema": a compressed `BYTEA`
   full image + a small `BYTEA` thumbnail, no files-on-disk).
3. Implement the exact **step flow** the user described:
   1. **Start page** — a short description of the app and the steps as a
      numbered list, plus a **"Report issue"** (start) button and a
      **"Zgłoszenia"** button.
   2. **Camera page** — take a photo and save it (into Postgres).
   3. **Location page** — capture GPS; show **coordinates on one side and the
      image on the other** (two-column layout), plus a **map with a pin**.
   4. **Description page** — a text field and a **"Generate"** button that
      fills in the AI-style default description.
   5. **Review page** — a readable preview of the report with **three
      thumbnails side by side**.
   6. **Reports page** — a separate page listing **all reports from Postgres**.
4. Honor the two-pass rule: Pass 1 = the whole flow as stubs/mocks so it runs
   end-to-end; Pass 2 = replace each stub with the real implementation.

Everything below is a means to these goals. The plan is a living document;
items are re-checked against the goals at each review and adjusted
incrementally — never followed blindly.

## Where the work lives

- **`i3_ref/`** is the implementation: a local clone of `FDHKRsss/i3` with its
  own git remote (`git@github.com:FDHKRsss/i3.git`). Milestone-acceptance
  commits for *code* land there, and the i4 repo pins `i3_ref` as a gitlink to
  the accepted commit.
- **`civil42pwa_ref/`** is the **read-only protoplast reference** (clone of
  `rzymek/civil42pwa-public`). It is never configured as a git remote we push
  to, and never pushed to.
- This repo (`i4`) owns the workspace-level `docs/` (this file, ARCHITECTURE,
  CONTEXT, lessons); the `i3_ref/docs/` copies are owned by the i3 build and are
  pinned by `i3_ref/tests/docs.spec.ts` — they must stay consistent with the
  i3 working tree.

## Human notes & how they are handled

- *"konthynuuj to co zostalo juz zaczete, bo obecna wersja jest wersja
  dzialajaca"* — we keep the working Express + Postgres + Vite/React foundation
  and evolve it; M8–M15 replace the mocked capture flow with the real mobile
  flow (the M1–M7 seed is preserved as the baseline).
- *"zrob ja bardziej jak z repo protoplasty … zeby z komorki mogl zrobic zdjecie,
  zeby pobralo jego gps"* — M9 replaces the mock camera with real
  `getUserMedia`; M10 replaces the mock GPS with real `navigator.geolocation`.
  Both keep a permission-denied/headless fallback so the app still runs where
  capture is impossible.
- *"zrzut ekranu … z mapa i np pinezka … zeby to wizualnie mialo sens"* — M10
  adds a self-contained `MapPin` component (OSM tile grid + centered pin)
  rendered from the captured coordinates; it is derived from lat/lon, so no map
  image needs to be persisted.
- *"teraz to trzymamy w postgresie, nie wiem jak tam sie przechowuje zdjecia,
  wymysl cos"* — M14 stores the compressed photo **and** a small thumbnail as
  `BYTEA` columns in Postgres and serves them via `/api/reports/:id/image` and
  `/api/reports/:id/thumbnail`.
- *"przycisk 'generate' ktory bedzie generowal opis"* — M11 implements
  `generateDescription()` producing the deterministic A.I.-style default text;
  no external LLM/API key is required (a real LLM is a marked later swap).
- *"miniaturki 3 obok siebie czytelne"* — M12 renders the review page as three
  side-by-side tiles: **photo**, **map+pin**, **summary** (description +
  coordinates). See ARCHITECTURE "Mobile reporting flow" for the interpretation.
- *"dodatkowy przycisk 'zgloszenia' … listuje wszystkie zgloszenia w postgresie"*
  — M8 adds the button; M13 renders the full list from `GET /api/reports`.
- *"nie pushuj nic do tego repo"* (about `civil42pwa-public`) — that repo stays a
  read-only reference at `civil42pwa_ref/`; it is never configured as a git
  remote and never pushed to.

## Constraints (non-negotiable)

- Do **not** push to `civil42pwa-public`, and never add it as a git remote.
- Do **not** write `README.md` (owned by the goal / human gate) — neither the
  workspace one nor `i3_ref/README.md`.
- Host port must be configurable via env with a default; never assume a fixed
  host port is free (`APP_PORT`, default `8080`).
- Real capture is the goal, but the app must still boot and be testable where
  camera/GPS are unavailable (permission denied, headless test): every capture
  module keeps a deterministic fallback.
- Camera + geolocation require a **secure context** (HTTPS, or `localhost`);
  the docs must state this clearly.
- Photos are stored in Postgres (`BYTEA`), not as orphaned files on a volume.
- Stack stays: Node 22 LTS, Vite + React (TS), Express, `pg`, PostgreSQL 16,
  Docker + Compose. Dev-workspace gate = `npm test` + `npm run typecheck`
  (Docker is not available here).

## Milestones

Pass 1 = every milestone as a stub/mock so the whole app runs end-to-end.
Pass 2 = replace each stub with the real implementation.

### Seed (M1–M7) — foundation kept from the working project

- [x] M1 -- stub   Bootstrap: directory skeleton (`src/ server/ db/`) + no-op server + placeholder Dockerfile/compose so the repo boots.
- [x] M1 -- real   Source copied in; Firebase, Snowflake, `.github` CI, surge/PWA asset gen removed; single root `package.json`.

- [x] M2 -- stub   DB layer mocked: in-memory store returning canned rows.
- [x] M2 -- real   PostgreSQL 16 via compose, `db/init.sql` (`reports` table), `pg` pool + insert/list.

- [x] M3 -- stub   Backend API stubbed: `/health`, `POST /api/report`, `GET /api/reports` with canned responses.
- [x] M3 -- real   Backend real: multipart parse (busboy), validation, mock geo, uploads volume, persist + list via `pg`, serve `dist/`.

- [x] M4 -- stub   Frontend stub: App renders and submits a hard-coded Blob.
- [x] M4 -- real   Frontend real: mock camera, mock audio, mock GPS, submit multipart, result label.

- [x] M5 -- stub   Compose stub: minimal compose + Dockerfile placeholder.
- [x] M5 -- real   Compose real: multi-stage build, `app` + `db`, `pgdata`/`uploads` volumes, `APP_PORT`, healthchecks.

- [x] M6 -- stub   Tests stub: vitest/supertest smoke + RUNBOOK placeholder.
- [x] M6 -- real   Tests real: backend validation + geo-mock unit tests, frontend smoke, runbook pins.

- [x] M7 -- real   Fresh-clone robustness: `pretest` builds `dist/`; `typecheck` covers `tests/**`; SPA fallback + non-numeric `limit` + lat-only geocode specs.

### Mobile reporting flow (M8–M15)

- [x] M8 -- stub   **Home & navigation shell.** Hash mini-router (`#/`, `#/new`, `#/reports`); Home renders the numbered step list and the two buttons ("Report issue" → `#/new`, "Zgłoszenia" → `#/reports`); wizard and reports are placeholders.
- [x] M8 -- real   Real copy (PL), wired navigation, step indicator in the wizard, reports page shell with empty/error states. No placeholder text left.

- [x] M9 -- stub   **Camera step.** Canvas mock pushes a placeholder photo + thumbnail Blob into the wizard state; "Retake"/"Continue" buttons.
- [x] M9 -- real   **Camera step.** Real `getUserMedia({video:{facingMode:'environment'}})` live preview + shutter; canvas downscale → compressed JPEG (≤1280 px) + thumbnail (≤360 px); permission/error fallback to the mock; stop tracks on unmount.

- [x] M10 -- stub  **Location step.** Mock GPS + grey placeholder map with a pin; two-column layout (coordinates | map). *(Subsumed by M10 -- real, delivered together: the real module already keeps the headless-safe path — manual lat/lon entry + deterministic mock reverse-geocode — so no separate stub step was needed.)*
- [x] M10 -- real  **Location step.** `navigator.geolocation.getCurrentPosition` (high accuracy, timeout), error + retry + manual lat/lon fallback; `MapPin` (OSM tile grid + centered pin); two columns (left: coordinates + address + accuracy, right: map); reverse-geocode via `geo.ts` (mock default, `nominatim` opt-in).

- [x] M11 -- stub  **Description step.** Textarea + "Generate" button that sets the fixed default `"test default description A.I. generated based on the incident picture"`. *(Subsumed by M11 -- real, delivered together: the real module already renders the textarea + "Generate" default and is headless-safe.)*
- [x] M11 -- real  **Description step.** `generateDescription()` builds a deterministic A.I.-style description from the picture/location metadata; editable textarea; non-empty validation before continuing.

- [x] M12 -- stub  **Review & submit.** Three placeholder tiles (photo, map, summary) from wizard state; "Submit" posts to `/api/report` and shows a canned success. *(Subsumed by M12 -- real, delivered directly: the real review step already renders the three tiles and is headless-safe.)*
- [x] M12 -- real  **Review & submit.** Real photo thumbnail, real map thumbnail, summary tile (description + coordinates); multipart POST (`image`, `thumbnail`, `lat`, `lon`, `description`); 4xx/5xx handling; success → `#/reports`.

- [x] M13 -- stub  **Reports list.** `/api/reports` returns canned rows; list renders placeholders. *(Subsumed by M8 -- real's shell + M13 -- real, delivered together: the shell already fetched `/api/reports` and rendered loading/error/empty/ready states.)*
- [x] M13 -- real  **Reports list.** Fetch `/api/reports`; render each report with photo thumbnail, map thumbnail, description, coordinates, timestamp; newest first; empty/error states.

- [x] M14 -- stub  **Backend & DB.** Endpoints `/api/reports`, `/api/reports/:id/image`, `/api/reports/:id/thumbnail` with canned data; in-memory store with the new shape. *(Subsumed by M14 -- real, delivered together: the real `db.ts`/`report.ts`/`index.ts` already implement the BYTEA contract end-to-end and are covered by tests.)*
- [x] M14 -- real  **Backend & DB.** `db/init.sql` new `reports` table (drop audio, add `description`, `image BYTEA`, `thumbnail BYTEA`); `db.ts` insert/list/get image/get thumbnail; `report.ts` multipart parse (`image` required, `thumbnail` optional, `lat`, `lon`, `description`) + validation; image-serving routes; health.

- [x] M15 -- stub  **Compose, tests & docs.** Compose still boots `app`+`db`; smoke tests pass with stubs; RUNBOOK placeholder.
- [x] M15 -- real  **Compose, tests & docs.** Compose drops the `uploads` volume and the `UPLOAD_DIR` env (BYTEA storage), removes the now-dead `server/store.ts` + `tests/store.spec.ts`, keeps `pgdata` + `APP_PORT` env; documents the HTTPS reverse-proxy requirement for mobile camera/GPS; unit + frontend tests for description generator, map tile math, geo, db row mapping, report validation, endpoints, Home/wizard/reports; RUNBOOK rewritten to the real photo/GPS/description flow; `tests/runbook.spec.ts` re-pinned and `tests/docs.spec.ts` updated to the M15 state.

## Current status

- **All milestones M1–M15 are done** (Pass 1 stubs and Pass 2 real
  implementations), and the build is **green**: `npm test` → **174 passed**
  (22 files) and `npm run typecheck` → clean, verified against the working tree
  this turn (Node 22 via
  `/home/op/.local/node-v22.23.2-linux-x64/bin`).
- The flow is end-to-end: **Home** (numbered steps + "Report issue" +
  "Zgłoszenia") → **Camera** (real `getUserMedia` + canvas JPEG/thumbnail) →
  **Location** (real GPS, two columns: coordinates | map+pin) → **Description**
  ("Generate" default) → **Review** (3 side-by-side tiles) → **Reports**
  (all rows from Postgres, newest first). Photos persist as `BYTEA` in Postgres
  and are served via `/api/reports/:id/image|thumbnail`.
- No remaining Pass-1 stubs and no remaining Pass-2 items are open.

## Out of scope / future swaps (modular, minimal now)

- **PWA installability / offline** — not required by the steps; re-addable via
  `vite-plugin-pwa`.
- **Real AI description** — `src/description.ts` is the swap point (needs
  keys/network).
- **Object storage (S3/MinIO)** — `server/db.ts`/`server/report.ts` swap point
  if `BYTEA` ever outgrows the use case.
- **Real backend reverse-geocoding** — `server/geo.ts` is currently mock-only;
  the frontend already supports an opt-in `nominatim` provider, and the backend
  module boundary is the swap point.
- **Audio capture** — removed to match the requested photo+location+description
  flow; the capture-module boundary makes it easy to re-add.
