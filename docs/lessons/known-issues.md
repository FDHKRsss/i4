# Known issues

_Recurring walls/gotchas and how to get past them. One bullet each._

- Node/npm are NOT on the default PATH. Prepend `/home/op/.local/node-v22.23.2-linux-x64/bin`
  (matches `.nvmrc` = `22`) before `npm test` / `npm run typecheck`, else `command not found: node`.
- Docker is unavailable in this workspace (`docker: not found`). Validate backend/API behavior with
  the Node test suite (`npm test` + `npm run typecheck`) instead of relying on `docker compose`.
- `npm test` prints `Error: db down` to stderr; this is EXPECTED — the negative-path specs mock the DB
  calls to reject (GET `/api/reports` → `503`, POST `/api/report` → `500`, and
  GET `/api/reports/:id/image|thumbnail` → `503`). Exit code 0 + all tests passing means the suite is
  green, not broken.
- Reviewers read the **committed** tree (`HEAD`), not the **working** tree — so a doc/test-state
  claim already fixed in the working tree still gets re-flagged as stale. Verify against the working
  tree (the runner's `N passed`, or the actual file), NOT `git show HEAD`: a reviewer read HEAD (`43`)
  as "matching" while the runner had `46`, and at M14 the critic re-flagged `PLAN.md`'s M14 checkboxes
  as unchecked while the working tree already had them `[x]`.
- Critic verdicts are unreliable — even when handed the goal, `docs/PLAN.md`, and the prior actor's
  output, the critic can still emit a non-verdict (`approved:false` with `Could not parse the
  critic's verdict as JSON`, `No task/goal, plan, or actor output`, or `need more steps`). Treat any
  reply that is not a single strict-JSON `{approved, blocking_issues, cosmetic_issues, notes}` object
  as "not reviewed"; always include the goal + plan + last actor output when invoking a reviewer.
- Contract-pinning tests break when the contract they pin changes — they are test-owned, so rewrite
  them to pin the new shape, never the production code. Happened twice: the seed `src/app.spec.tsx`
  pinned the capture-screen copy (broke at M8 → Home), and the backend HTTP specs
  (`tests/app.spec.ts`, `tests/report.spec.ts`) pinned the old `voice`/`audio_path` row shape
  (broke at M14 when the BYTEA contract landed).
- Milestone state lives in THREE places that drift independently — PLAN.md's `- [x]` checkbox,
  PLAN.md's `## Current status`, and ARCHITECTURE.md's `## Implementation status`. Update all three
  in the SAME change when a milestone ships, or the critic re-flags whichever is stale (recurred at
  M11, M12, and M14). Since M14, `tests/docs.spec.ts` also pins these three locations — update it in
  the same change too, or the suite breaks when the next milestone lands.
