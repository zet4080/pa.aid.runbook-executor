# ARC-1303 — Display checkpoint queue with automatic priority scoring

## Scope & Alignment
- Story: ARC-1303
- Approach: Client-side priority scoring, new hook + component, polling `/api/state`.

## Assumptions & Dependencies
- Dependency: ARC-1302 must execute before ARC-1303 (ARC-1302 adds `prUrl?` to server-side `CheckpointQueueEntry`).

## Implementation Steps
1. Add `CheckpointQueueEntry` client-side type to `packages/ui/src/types/runbook.ts`
2. Create `packages/ui/src/lib/checkpointScore.ts` scoring utility (pure functions)
3. Create `packages/ui/src/hooks/useCheckpointQueue.ts` hook (polls, sorts, optimistic resume)
4. Create `packages/ui/src/components/CheckpointQueuePanel.tsx` component (ordered list, emoji, wait time, PR link, Resume)
5. Integrate into `packages/ui/src/pages/Dashboard.tsx` (render panel above session config when queue non-empty)

## Testing & Validation
- `pnpm --filter @runbook-executor/ui test`
- `pnpm --filter @runbook-executor/ui typecheck`

## Risks & Open Questions
- `useCheckpointQueue` is introduced in both ARC-1302 and ARC-1303 — ARC-1303 should reuse/extend the hook created in ARC-1302 rather than duplicate it
- Optimistic resume may get out of sync if server rejects the request — add error handling and re-poll
- `laneLabel` utility may be duplicated; consider extracting shared helper before ARC-1303 executes