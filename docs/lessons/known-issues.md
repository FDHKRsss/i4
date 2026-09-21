# Known issues

_Recurring walls/gotchas and how to get past them. One bullet each._

- Frontend smoke tests pinned to exact copy break every time a milestone changes the copy (M8 replaced the seed-capture string). Assert behavior/structure — heading, ordered-list length, `href` targets — instead of literal copy strings.
- A plan checkbox reflects committed state, not the working tree: M8 -- stub still read `- [ ]` while it was already implemented and green as uncommitted changes. Before (re)implementing an open item, check `git status` and read the working tree, not just the checkbox.
- PLAN.md records milestone state in two places that drift independently — the `- [x]` checkbox and the `## Current status` narrative. When marking a milestone done, update both, or the critic re-flags whichever is stale.
- `npm test` prints `Error: db down` to stderr — this is EXPECTED, not a failure: the negative-path specs mock `listReports`/`insertReport` rejection (`GET /api/reports` → `503`, `POST /api/report` → `500`). Exit code 0 + all tests passing = green.
- Adding a gate to a wizard step (e.g. "Dalej" disabled until a photo is captured) breaks the existing multi-step navigation tests, which click "Dalej" repeatedly and now stall at the gated step. Fix them in the SAME change: satisfy each new gate first — a `capturePhoto(container)` helper that clicks the step's `<canvas>`, later mock geolocation / fill the description — before clicking "Dalej". This recurs for M10 (location) and M11 (description).
