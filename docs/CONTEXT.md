# Project context (durable directives all agents must always honor)

- Turn the working `i3` project (`i3_ref/`) into a real mobile incident-reporting PWA: on a phone, take a **real photo** and capture **real GPS**, add a description, and submit — like the protoplast, not a mocked seed.
- Step flow: **Home** (numbered steps + "Report issue" + "Zgłoszenia") → **Camera** → **Location** (two columns: coordinates | map+pin) → **Description** ("Generate" default) → **Review** (3 side-by-side tiles) → **Reports** (all rows from Postgres). Each wizard step gates **"Dalej"** until its input is captured (photo → GPS → description).
- Photos are compressed **client-side via `<canvas>`** (no server image lib like `sharp`) and stored **in PostgreSQL** as `BYTEA` (full + thumbnail), served via `/api/reports/:id/image` and `/api/reports/:id/thumbnail`.
- **M13 is frontend-first against the M14 backend contract** (M12 review/submit already shipped). The seed backend still expects the old `voice`/`audio_path` row shape, so a real `POST /api/report` or `GET /api/reports` will fail until M14 lands. Build M13 with mocked-`fetch` tests; do **not** rewrite the backend as part of it.
- Real capture is the goal, but every capture module keeps a permission-denied/headless fallback so the app and tests still run without camera/GPS. Camera + geolocation require **HTTPS** (or `localhost`).
- **Never push to `civil42pwa-public` and never add it as a git remote** — read-only reference at `civil42pwa_ref/`. Do **not** write `README.md`.
- Keep the stack: Node 22 LTS, Vite + React (TS), Express, `pg`, PostgreSQL 16, Docker + Compose.
- Frontend smoke tests assert DOM **structure/behavior** (heading, step/list length, `href`, `aria-current`) — never exact copy strings, which change every milestone and break.
- Dev env here: **Docker is NOT available** — validate with `npm test` + `npm run typecheck`. Node 22 is not on the default PATH — prepend `/home/op/.local/node-v22.23.2-linux-x64/bin` before npm commands.
