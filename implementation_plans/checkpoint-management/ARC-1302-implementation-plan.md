# ARC-1302 — Create Bitbucket PR and display persistent [Resume] card on checkpoint hit

## Scope & Alignment
- Story: ARC-1302
- Approach: Server-side git + direct Bitbucket REST API (no new dependencies), non-blocking PR creation, polling UI hook.

## Assumptions & Dependencies
- Bitbucket credentials must be available as environment variables at server runtime.
- Branch naming collision if same stepId reused across sessions — consider session-scoped prefix.

## Implementation Steps
1. Add `prUrl?: string` to `CheckpointQueueEntry` in `packages/server/src/state/types.ts` + mirror in UI types
2. Create `packages/server/src/git/checkpointGit.ts` with `createCheckpointBranchAndPR()`
3. Wire into `packages/server/src/runner/laneRunner.ts` at both checkpoint pause sites (try/catch, non-blocking)
4. Add `POST /api/checkpoints/:stepId/resume` route in `packages/server/src/routes/checkpoints.ts`
5. Create `packages/ui/src/hooks/useCheckpointQueue.ts` hook (polls, exposes `resumeCheckpoint`)
6. Create `packages/ui/src/components/ResumeCard.tsx` component
7. Integrate ResumeCards into `packages/ui/src/pages/Dashboard.tsx`

## Testing & Validation
- `pnpm --filter @runbook-executor/server test`
- `pnpm --filter @runbook-executor/ui test`
- `pnpm --filter @runbook-executor/server typecheck`
- `pnpm --filter @runbook-executor/ui typecheck`

## Risks & Open Questions
- Bitbucket credentials must be available as environment variables at server runtime (BITBUCKET_URL, BITBUCKET_USER, BITBUCKET_TOKEN, BITBUCKET_PROJECT, BITBUCKET_REPO)
- Branch naming collision if same stepId reused across sessions — consider session-scoped prefix
- `useCheckpointQueue` hook is also needed by ARC-1303; the hook interface must remain stable