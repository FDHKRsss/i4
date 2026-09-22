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
  M11, M12, and M14). Only the **`i3_ref/docs/*`** copies are pinned by `i3_ref/tests/docs.spec.ts` —
  update that spec in the same change when you touch `i3_ref/docs/*`, or the suite breaks.
- Two doc trees coexist and have diverged: **workspace `docs/*`** (the living plan/design source of
  truth, NOT pinned by any test) vs **`i3_ref/docs/*`** (an older per-milestone set owned by the i3
  build, pinned by `i3_ref/tests/docs.spec.ts`). Edit them separately: workspace-doc edits never
  affect the suite; `i3_ref/docs/*` edits belong to the i3 build and must keep that spec green (its
  PLAN.md keeps the literal `M15 (stub + real) — done` the spec pins, while workspace PLAN.md does
  not).
- ARCHITECTURE.md's `## What will be in the code` tree also carries per-file inline comments that
  encode milestone/swap state, but these are NOT pinned by `i3_ref/tests/docs.spec.ts`, so they drift
  silently. Proof: the `server/geo.ts` comment claimed `(seed; swapped under M14)` while the provider
  was still mock-only. When you touch such a comment, verify it against the actual source
  (`server/geo.ts` etc.) and fix the comment — never rewrite the code to match a stale claim.
