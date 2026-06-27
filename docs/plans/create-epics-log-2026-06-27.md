# Create Epics Log — 2026-06-27

**Project:** ARC
**Planning repo:** pa.aid.runbook-executor
**Requirements doc:** docs/requirements/2026-06-27-runbook-executor.md

## Epics Created

| Epic Key | Summary |
|----------|---------|
| ARC-1284 | ARC: Core Infrastructure |
| ARC-1289 | ARC: Session Setup & Runbook Selection |
| ARC-1290 | ARC: Parallel Lane Execution |
| ARC-1295 | ARC: Checkpoint Management |
| ARC-1296 | ARC: Queue & Scheduling Policy |
| ARC-1300 | ARC: Failure Handling & Escalation |
| ARC-1309 | ARC: Session History |

## Stories Created

| Story Key | Epic | Summary |
|-----------|------|---------|
| ARC-1285 | ARC-1284 | Bootstrap web server and serve local UI |
| ARC-1286 | ARC-1284 | Parse runbook markdown into typed step graph |
| ARC-1287 | ARC-1284 | Implement sidecar state file |
| ARC-1288 | ARC-1284 | Integrate opencode CLI subprocess with JSON event stream |
| ARC-1313 | ARC-1289 | Display runbook dashboard at startup |
| ARC-1314 | ARC-1289 | Expand runbook detail on demand |
| ARC-1315 | ARC-1289 | Select runbooks to include in session |
| ARC-1316 | ARC-1289 | Configure session parameters before start |
| ARC-1291 | ARC-1290 | Execute runbook steps sequentially within a lane |
| ARC-1292 | ARC-1290 | Run multiple lanes concurrently up to concurrency limit |
| ARC-1293 | ARC-1290 | Stream live agent output per lane in the UI |
| ARC-1294 | ARC-1290 | Update runbook checkbox state in real time |
| ARC-1301 | ARC-1295 | Detect checkpoint markers and pause lane execution |
| ARC-1302 | ARC-1295 | Deliver browser notification within 5 seconds of checkpoint hit |
| ARC-1303 | ARC-1295 | Display checkpoint queue with automatic priority scoring |
| ARC-1304 | ARC-1295 | Inline-edit checkpoint artifacts and persist handoff file to disk |
| ARC-1305 | ARC-1295 | Approve or reject checkpoint inline with feedback |
| ARC-1297 | ARC-1296 | Display ordered checkpoint queue with priority scores |
| ARC-1298 | ARC-1296 | Reorder checkpoint queue manually |
| ARC-1299 | ARC-1296 | Select and switch queue policy mid-session |
| ARC-1306 | ARC-1300 | Retry failed agent steps up to configurable limit |
| ARC-1307 | ARC-1300 | Escalate agent level on each retry |
| ARC-1308 | ARC-1300 | Surface exhausted retries as supervisor checkpoint notification |
| ARC-1310 | ARC-1309 | Persist active session state across restarts |
| ARC-1311 | ARC-1309 | Browse past sessions list |
| ARC-1312 | ARC-1309 | View full step output history for any past session |
