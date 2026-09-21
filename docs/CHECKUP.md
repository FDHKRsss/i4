# Environment checkup list

_A living checklist the doctor runs before/while the team works. Each row is a cheap check with a concrete
fix. The doctor APPENDS a new row whenever it solves a new blocker, so the next run catches it automatically._

Project under test: `i3_ref/` (Node 22 LTS + Vite/React TS + Express + `pg`). Docker is NOT available here —
the gate is `npm test` + `npm run typecheck`. Node is not on the default PATH.

| symptom | check | fix (no host privileges) |
|---|---|---|
| `node`/`npm` not found or wrong version | `node --version` → `v22.x`, `npm --version` | prepend `/home/op/.local/node-v22.23.2-linux-x64/bin` to PATH before every npm/node command |
| dependencies missing / stale | `cd i3_ref && npm ls --depth=0` (no `UNMET DEPENDENCY`, no `invalid`) | `cd i3_ref && npm ci` (or `npm install`) |
| tests fail | `cd i3_ref && npm test` (all suites pass; `pretest` rebuilds `dist/`) | fix env/deps; if a test is genuinely broken in code, report it — the coder fixes code |
| type errors | `cd i3_ref && npm run typecheck` (frontend + server + test tsconfigs clean) | fix env/deps; report real type errors in code |
| port left occupied by a prior run | `ss -ltnp` for this project's ports (app default `8080`, Vite dev `5173`) | stop only THIS project's leftover `node server-dist`/`vite` process; never touch other services |

_Report-only (do not fix): missing API keys, git branch state. Never add `civil42pwa-public` as a git remote
and never push to it — it stays a read-only reference at `civil42pwa_ref/`._
