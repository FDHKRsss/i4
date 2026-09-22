# Architecture

## Overview

`i4` is the workspace that owns the continued development of the **`i3`**
project (implementation lives in `i3_ref/`, a local clone of
`FDHKRsss/i3`). `i3` is a self-contained, Unix-runnable **mobile
incident-reporting PWA**: open the URL on a phone → capture a **real photo**
and **real GPS** → review a two-column location + map/pin screen → write (or
generate) a description → review three readable tiles → submit. Every report is
stored in **PostgreSQL**, including the photo bytes.

```
Phone browser ──HTTPS──> reverse proxy (TLS, out of scope) ──HTTP──> app (Express, :8080 in container)
                                                                       ├─ POST /api/report                → busboy parse → validate → geocode
                                                                       │                                    → INSERT image+thumbnail+metadata into Postgres
                                                                       ├─ GET  /api/reports               → SELECT metadata (no image bytes)
                                                                       ├─ GET  /api/reports/:id/image     → SELECT image  BYTEA → image/jpeg
                                                                       ├─ GET  /api/reports/:id/thumbnail → SELECT thumbnail BYTEA → image/jpeg
                                                                       ├─ GET  /health                    → SELECT 1
                                                                       └─ /*                              → static files from dist/

                                                                       app ──TCP 5432──> db (postgres:16-alpine)  volume: pgdata
```

Key decisions vs. the earlier seed:

| Concern | Earlier seed | Final design |
|---|---|---|
| Camera | mock canvas only | **real `getUserMedia`** (rear camera), with mock fallback |
| GPS | fixed mock | **real `navigator.geolocation`**, with manual-entry fallback |
| Photos | files on `uploads` volume + path in DB | **`BYTEA` in Postgres** (compressed full + thumbnail), served via API |
| Flow | single capture screen | 6-step wizard + separate reports page (hash-routed) |
| Audio | synthetic WAV | **removed** (not in the requested steps; re-addable module) |
| Map | none | self-contained `MapPin` tile grid derived from lat/lon |

## Implementation status (living)

**All milestones M1–M15 are done and the build is green** (`npm test` → 174
passed, `npm run typecheck` → clean, verified this turn). Per-milestone detail:

- **M8 (Home & navigation shell)** — hash mini-router (`#/`, `#/new`,
  `#/reports`); Home with real Polish copy, the numbered step list and
  "Report issue" / "Zgłoszenia" links; a wizard with a step indicator
  (`aria-current` tracking) and back/next navigation; a Reports page with
  loading / error / empty / ready states.
- **M9 (Camera step)** — real `getUserMedia` camera: rear camera
  (`facingMode: "environment"`, no audio), live `<video>` preview, shutter
  gated on the live state; `compressToImages()` downscales the frame to a JPEG
  **full** (≤1280 px @ 0.85) + **thumbnail** (≤360 px @ 0.72); camera
  unavailable / permission denied falls back to `MockCamera`; tracks stopped on
  unmount.
- **M10 (Location step)** — real `navigator.geolocation` (high accuracy +
  10 s timeout + `maximumAge: 0`), typed `GeolocationError` mapping, retry, and
  a validated manual lat/lon fallback; the required two-column screen — left:
  coordinates + reverse-geocoded address + accuracy, right: `MapPin` (3×3 OSM
  tile grid + centered pin). Reverse-geocoding via `src/capture/geo.ts`
  (deterministic mock default; `nominatim` opt-in).
- **M11 (Description step)** — editable textarea + "Generate" button filled by
  `generateDescription()`, a deterministic A.I.-style generator that always
  starts with the fixed
  `"test default description A.I. generated based on the incident picture"` and
  appends short annotations for the captured photo/coordinates/time; "Dalej"
  gates on a non-empty description. No network / API key.
- **M12 (Review & submit)** — three side-by-side tiles (**photo thumbnail**,
  **map + pin thumbnail**, **summary**: description + coordinates + address);
  "Wyślij" POSTs a multipart payload (`image`, `thumbnail`, `lat`, `lon`,
  `description`) via `src/send.tsx` → `/api/report`; success → `#/reports`;
  4xx/5xx/network → short, non-leaky error with retry.
- **M13 (Reports list)** — every report from `GET /api/reports`, newest-first,
  each showing photo thumbnail (`thumbnailUrl`, falling back to `imageUrl`),
  map + pin tile, summary (description + coordinates + address + timestamp),
  plus loading / error / empty states; degrades gracefully against older rows.
- **M14 (Backend & DB)** — BYTEA-backed. `db/init.sql` defines `reports`
  (`description TEXT`, `image BYTEA` required, `thumbnail BYTEA` optional, audio
  columns dropped); `server/db.ts` provides insert/list/getImage/getThumbnail;
  `server/report.ts` parses multipart (`image` required, `thumbnail` optional,
  `lat`/`lon` validated finite + in-range, `description`) and reverse-geocodes;
  `server/index.ts` serves `GET /api/reports/:id/image|thumbnail` as
  `image/jpeg`, keeps `GET /api/reports` metadata-only, and preserves `/health`.
- **M15 (Compose, tests & docs)** — compose drops the `uploads` volume and
  `UPLOAD_DIR` env (BYTEA storage), keeps `pgdata` + `APP_PORT`; dead
  `server/store.ts` + `tests/store.spec.ts` removed; HTTPS reverse-proxy
  requirement documented; RUNBOOK rewritten; `tests/runbook.spec.ts` and
  `tests/docs.spec.ts` re-pinned.

## Source & git

- **`i3_ref/`** is a local clone of `https://github.com/FDHKRsss/i3.git` with
  its own remote (`git@github.com:FDHKRsss/i3.git`). Code milestone commits land
  there; the `i4` repo tracks `i3_ref` as a **gitlink** pinned to the accepted
  `i3` commit.
- **`civil42pwa_ref/`** is a **read-only reference** clone of
  `https://github.com/rzymek/civil42pwa-public.git`. It is **never** configured
  as a remote we push to and **never pushed to**; it is only read for the
  protoplast's approach.
- Neither `README.md` (workspace or `i3_ref/README.md`) is written by agents;
  the goal / human gate owns them.

## Tooling (chosen, and why)

| Concern | Choice | Why (vs. alternatives) |
|---|---|---|
| Runtime | Node.js **22 LTS** | already in use; LTS. |
| Frontend | **Vite 5 + React 18 + TS** (already present) | keep the working app; minimal change. |
| Backend | **Express** + **busboy** + **pg** | already present; busboy parses multipart, `pg` drives Postgres. |
| DB | **PostgreSQL 16** (alpine) | already present; `BYTEA` + `gen_random_uuid()`. |
| Image processing | **none on the server** (no `sharp`) | the browser downscales/compresses via `<canvas>`; keeps images lean and avoids a native dep. |
| Routing | **hash mini-router** (~40 LOC, no dependency) | real URLs + back-button on mobile without adding `react-router`. |
| Map | **self-contained `MapPin`** using OSM raster tiles | no dependency (rejects `leaflet`); tile math is ~30 LOC and the pin is CSS/SVG. |
| AI description | **deterministic template module** | no API key/network; LLM API is a marked later swap. |
| Package manager | **npm** | already present. |

## What will be in the code

```
i3_ref/
├─ src/                        # frontend (Vite + React, TypeScript)
│  ├─ main.tsx                 # entry + error boundary
│  ├─ app.tsx                  # hash router → Home / New wizard / Reports
│  ├─ pages/
│  │  ├─ Home.tsx              # numbered steps + "Report issue" + "Zgłoszenia"
│  │  ├─ NewReport.tsx         # wizard state machine (camera→location→description→review)
│  │  └─ Reports.tsx           # lists all reports from Postgres
│  ├─ capture/
│  │  ├─ Camera.tsx            # real getUserMedia camera + shutter + fallback
│  │  ├─ MockCamera.tsx        # kept as the permission-denied/headless fallback
│  │  ├─ location.ts           # navigator.geolocation + manual fallback
│  │  ├─ geo.ts                # frontend reverse-geocode (mock default; nominatim opt-in)
│  │  └─ image.ts              # canvas downscale/compress → full + thumbnail blobs
│  ├─ map/
│  │  ├─ MapPin.tsx            # OSM tile grid + centered pin (interactive + thumb sizes)
│  │  └─ tiles.ts              # slippy-map tile math (tileCoords / wrapTileX / clampTileY)
│  ├─ description.ts           # generateDescription() (deterministic A.I.-style text)
│  └─ send.tsx                 # multipart POST (image, thumbnail, lat, lon, description)
├─ server/                     # backend (TypeScript, compiled to server-dist/)
│  ├─ index.ts                 # routes incl. /api/reports/:id/image|thumbnail
│  ├─ report.ts                # multipart parse + validation + orchestration
│  ├─ db.ts                    # insert/list/getImage/getThumbnail/ping
│  └─ geo.ts                   # backend reverse-geocode provider (mock only; real provider is a later swap)
├─ db/
│  └─ init.sql                 # reports table (description + image/thumbnail BYTEA)
├─ Dockerfile                  # multi-stage: build frontend+server → runtime
├─ docker-compose.yml          # app + db, pgdata volume, APP_PORT env
├─ .env.example                # APP_PORT, POSTGRES_*, GEO_PROVIDER, MAX_UPLOAD_BYTES
└─ docs/                       # PLAN + ARCHITECTURE + CONTEXT + RUNBOOK
```

## Mobile reporting flow (the 6 user steps)

1. **Home** (`#/`) — short app description, the steps as a numbered list
   (1 camera → 2 location/map → 3 description → 4 review → submit), and two
   buttons: **"Report issue"** (`#/new`) and **"Zgłoszenia"** (`#/reports`).
2. **Camera** — live preview from the rear camera; tap shutter → downscale +
   JPEG-compress → keep `full` (≤1280 px) and `thumbnail` (≤360 px) blobs in
   the wizard state.
3. **Location** — `navigator.geolocation` (high accuracy, ~10 s timeout);
   on success show a two-column screen: **left = coordinates + address +
   accuracy**, **right = map with pin**. On failure: retry + manual lat/lon.
4. **Description** — editable textarea + **"Generate"** button → fills the
   default AI-style description (see "Description generation").
5. **Review** — three side-by-side tiles: **photo thumbnail**, **map+pin
   thumbnail**, **summary** (description + coordinates + address). A submit
   button persists and redirects to `#/reports`.
6. **Reports** (`#/reports`) — every report from Postgres, newest first, each
   showing photo thumbnail, map thumbnail, description, coordinates, timestamp.

> Interpretation note: the user wrote *"miniaturki 3 obok siebie czytelne"*.
> The two images the flow produces are the **photo** and the **map/pin**, so the
> third tile is the **summary** (description + coordinates), which keeps all
> captured data readable in a 3-up layout. If a third *image* is later wanted,
> the review page is a single component and the tile set is trivial to change.

## Database schema (`db/init.sql`)

```sql
CREATE TABLE IF NOT EXISTS reports (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  lat         DOUBLE PRECISION,
  lon         DOUBLE PRECISION,
  geo_desc    TEXT,
  description TEXT NOT NULL DEFAULT '',
  image       BYTEA NOT NULL,
  thumbnail   BYTEA
);

CREATE INDEX IF NOT EXISTS reports_created_at_idx
  ON reports (created_at DESC);
```

- **How photos are stored**: the browser uploads the compressed photo and a
  small thumbnail; both are stored as `BYTEA` in Postgres. `image` is required,
  `thumbnail` optional. List queries select **only** metadata + `thumbnail`
  (never the full `image`) so the reports list stays cheap.
- **Why `BYTEA` and not files-on-volume + path**: a single source of truth in
  the one DB (matches "zapisywal w postgresie"), no orphaned files, no separate
  image volume/static route to secure, `pg_dump` backs up everything, and
  PostgreSQL TOAST handles the (already compressed) sizes comfortably.
  Full-resolution originals (~several MB) are **not** stored; the client
  compresses to a bounded size first. A future swap to object storage (S3) is
  noted below.

## API contract

- `POST /api/report` — `multipart/form-data`.
  - `image` (file, **required**, JPEG) — the compressed photo (the client always
    uploads a canvas-compressed JPEG).
  - `thumbnail` (file, optional) — small JPEG.
  - `lat` / `lon` (strings, optional but validated as finite, in-range).
  - `description` (string, optional) — defaults to `""`.
  - `200` → `Report received successfully`; `400` on malformed/missing
    image/invalid coords/oversized upload; `405` non-POST; `500` on server/DB
    failure (message logged, not leaked).
- `GET /api/reports?limit=50` — JSON list, newest first; each item returns
  `id`, `created_at`, `lat`, `lon`, `geo_desc`, `description`, and
  `thumbnailUrl` / `imageUrl` (no inline image bytes).
- `GET /api/reports/:id/image` → `image/jpeg` (or `404` if absent).
- `GET /api/reports/:id/thumbnail` → `image/jpeg` (or `404` if absent).
- `GET /health` → `{ ok: true, db: "up"|"down" }` after `SELECT 1`.

## Capture & permissions

| Capability | Primary (real) | Fallback (still runs headless/denied) |
|---|---|---|
| Camera | `navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' } })` | `MockCamera` canvas placeholder, with a clear "camera unavailable" note |
| GPS | `navigator.geolocation.getCurrentPosition({ enableHighAccuracy: true, timeout: 10000 })` | manual lat/lon entry + retry (no silent fake location) |
| Reverse geocode (frontend) | `VITE_GEO_PROVIDER=nominatim` (opt-in) | deterministic `mock` provider (default) |
| Reverse geocode (backend) | — | deterministic `mock` provider only (`server/geo.ts` is the swap point) |

- The fallbacks keep the module interfaces identical to the originals so each
  real implementation can be swapped in/out one file at a time.
- **Secure context**: `getUserMedia` and `geolocation` only work over **HTTPS**
  (or `localhost`). The app logs a clear in-page error otherwise. Deployment
  must put a TLS-terminating reverse proxy in front of the container (see
  "Port & resource handling").

## Map & pin

- `MapPin` renders a small, non-interactive-by-default map from a `{lat,lon}`:
  it computes the OSM slippy-map tile for a fixed zoom (~16), lays out an N×N
  grid of `https://tile.openstreetmap.org/{z}/{x}/{y}.png` `<img>`s around the
  center tile, and overlays a centered pin (SVG/CSS) using the fractional pixel
  offset of the coordinate inside the center tile. A tiny "© OpenStreetMap"
  attribution is included.
- This gives the *"zrzut ekranu z mapą i pinezką"* look without persisting any
  map image: it is always derived from the stored coordinates, so the same
  component powers the location step, the review tile, and each reports-list
  row. Tile requests are made by the phone browser directly to OSM.

## Description generation

- `generateDescription({ hasImage, lat, lon, at })` returns a deterministic,
  AI-style string (the requested default
  `"test default description A.I. generated based on the incident picture"`,
  optionally annotated with the captured time/coordinates). The "Generate"
  button fills the textarea; the user can edit before submitting.
- A real LLM call is deliberately out of scope (needs an API key + network);
  the module is the single swap point for a real provider later.

## Port & resource handling

- **Host port is configurable**: `APP_PORT` env (default `8080`) maps to the
  container's fixed `8080` (`docker-compose.yml`: `"${APP_PORT:-8080}:8080"`).
  Never assume the host port is free.
- Postgres is **internal only** (no host port published) to avoid `5432`
  conflicts.
- `DATABASE_URL` is injected (default
  `postgres://civil42:civil42@db:5432/civil42`).
- Upload size is capped (`MAX_UPLOAD_BYTES`, default 15 MB) and `lat`/`lon` are
  validated finite + in-range. Images are expected to be small because the
  client pre-compresses them; a too-large image is a `400`.
- **HTTPS for phones**: the container serves plain HTTP on `APP_PORT`; expose it
  through a TLS reverse proxy (e.g. Caddy/Traefik/nginx + Let's Encrypt) so the
  phone browser grants camera/GPS. `localhost` needs no TLS.

## Failure modes & error handling

- DB down at app start → app still boots; `/health` reports `db: "down"`;
  report POST returns `500` (details logged, not leaked).
- Missing/empty image, malformed multipart, bad coordinates → `400` with a
  short message.
- Camera denied / insecure context / no camera → in-page error + fallback to
  mock capture; the flow still completes.
- GPS denied / timeout → in-page error + retry + manual lat/lon entry.
- File insert or DB write fails → `500`; no partial record is left visible as
  success (the single-row insert is atomic).
- Port already bound inside the container → server fails fast with a clear
  message; host-side conflicts are handled by changing `APP_PORT`.

## Verification

- Dev-workspace gate: `npm test` (Vitest: frontend structure/behavior smoke,
  capture module unit tests, backend HTTP + report validation specs) and
  `npm run typecheck` (frontend + server + test tsconfigs). Both are green
  (174 passed). Docker is **not** available in this workspace — `docker compose
  up` is verified on the target box per `i3_ref/docs/RUNBOOK.md`.
- The workspace-level docs here are the plan/design source of truth; the
  `i3_ref/docs/*` copies are pinned by `i3_ref/tests/docs.spec.ts`, so any
  milestone state change must update PLAN + ARCHITECTURE (and that pin) in the
  same change.

## Out of scope & future swaps (modular, minimal now)

- **PWA installability / offline** — not required by the steps; re-addable via
  `vite-plugin-pwa` later.
- **Real AI description** — `src/description.ts` swap point (needs keys/network).
- **Object storage (S3/MinIO)** — `server/db.ts`/`server/report.ts` swap point
  if `BYTEA` ever outgrows the use case.
- **Real backend reverse-geocoding** — `server/geo.ts` is mock-only; swap point
  for a `nominatim`/other provider later.
- **Audio capture** — removed to match the requested photo+location+description
  flow; the capture-module boundary makes it easy to re-add.
- **Interactive/draggable map** — `MapPin` is intentionally static; a full
  `leaflet`/`maplibre` swap point is isolated in `src/map/`.
