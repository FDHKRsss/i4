# Project context (durable directives all agents must always honor)

- **Build is complete & green**: `i3` (`i3_ref/`) is now a real mobile incident-reporting PWA (real photo → real GPS → description → submit). All milestones M1–M15 are done; `npm test` + `npm run typecheck` pass. Don't re-do or regress shipped steps — verify against the working tree.
- Step flow (accepted design): **Home** (numbered steps + "Report issue" + "Zgłoszenia") → **Camera** → **Location** (two columns: coordinates | map+pin) → **Description** ("Generate" default) → **Review** (3 side-by-side tiles) → **Reports** (all rows from Postgres). Each wizard step gates **"Dalej"** until its input is captured (photo → GPS → description).
- Photos are compressed **client-side via `<canvas>`** (no server image lib like `sharp`) and stored **in PostgreSQL as `BYTEA`** (full + thumbnail), served via `/api/reports/:id/image` and `/api/reports/:id/thumbnail`. `POST /api/report` + `GET /api/reports` speak this contract (`image`/`thumbnail`/`lat`/`lon`/`description` in; `description`, `created_at`, `thumbnailUrl`/`imageUrl` out); the old `voice`/`audio_path` seed contract is gone.
- Capture modules use the real camera/GPS but each keeps a permission-denied/headless fallback so the app and tests still run without camera/GPS. Camera + geolocation require **HTTPS** (or `localhost`).
- **Never push to `civil42pwa-public` and never add it as a git remote** — read-only reference at `civil42pwa_ref/`. Do **not** write `README.md`.
- Keep the stack: Node 22 LTS, Vite + React (TS), Express, `pg`, PostgreSQL 16, Docker + Compose.
- Frontend smoke tests assert DOM **structure/behavior** (heading, step/list length, `href`, `aria-current`) — never exact copy strings, which change every milestone and break.
- Dev env here: **Docker is NOT available** — validate with `npm test` + `npm run typecheck`. Node 22 is not on the default PATH — prepend `/home/op/.local/node-v22.23.2-linux-x64/bin` before npm commands.
