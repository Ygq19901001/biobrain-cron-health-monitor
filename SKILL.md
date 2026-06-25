---
name: cron-health-monitor
slug: openclaw-cron-health-monitor
version: 1.0.0
displayName: "Cron Health Monitor"
summary: "Proactive cron job health monitoring, failure detection, and auto-repair delegation."
description: "Proactive cron job health monitoring, failure detection, and auto-repair delegation. Triggers: 'cron failed', 'cron health', 'fix cron', 'consecutive errors'"
tags:
  - cron
  - monitoring
  - health
  - auto-repair
  - devops
license: MIT
---

# Cron Health Monitor

Detect, diagnose, and repair cron job failures before they cascade.

## Core Principle

A cron that fails silently is worse than no cron at all. Monitor `consecutiveErrors` — it's the heartbeat of your automation.

## Health Check Workflow

### Step 1: List All Crons and Check State

```
cron(action="list")
```

For each job, check:
- `state.consecutiveErrors` — 0 is healthy, 1 needs attention, 2+ is critical
- `state.lastRunStatus` — "ok" or "error"
- `state.lastError` — the actual error message
- `state.lastDurationMs` — anomalies in execution time

### Step 2: Classify Failures

| consecutiveErrors | Severity | Action |
|-------------------|----------|--------|
| 0 | Healthy | No action |
| 1 | Watch | Log, monitor next run |
| 2 | Warning | Diagnose root cause |
| 3+ | Critical | CEO intervention, consider disabling |

### Step 3: Common Failure Patterns

**Pattern A: "Message failed"**
- Cause: `message` tool fails in isolated sessions (missing auth/feishu config)
- Fix: Change payload to use `sessions_send` to main instead
- Example fix: `cron(action="update", jobId="xxx", patch={payload: {message: "...use sessions_send to main, NOT message tool..."}})`

**Pattern B: "LLM request failed"**
- Cause: Model provider timeout, rate limit, or invalid model
- Fix: Switch to a different free model (e.g., zai/glm-4-flash) or add fallback
- Check: Is the model name correct? Is the provider's API key valid?

**Pattern C: "Write failed"**
- Cause: File path doesn't exist, permissions issue, or template variable not resolved
- Fix: Ensure parent directories exist; avoid template variables in file paths
- Example: `{workspace_root_dir}/<task-summary>.md` may not resolve in isolated sessions

**Pattern D: Timeout**
- Cause: Task too complex for allocated time
- Fix: Increase `timeoutSeconds` or simplify the task
- Split large tasks into smaller cron jobs

**Pattern E: Silent success but no output**
- Cause: Agent produced no meaningful work (just echoed "HEARTBEAT_OK")
- Fix: Add explicit "Do NOT reply HEARTBEAT_OK" to payload message

## Weekly Audit Pattern

Run weekly (e.g., Friday 22:00):

```json
{
  "name": "Weekly Cron Audit",
  "schedule": {"kind": "cron", "expr": "0 22 * * 5", "tz": "Asia/Shanghai"},
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "Weekly cron audit: List all cron jobs. Flag any with consecutiveErrors > 0. For each failure, diagnose root cause and suggest fix. Output summary to company/shared/alerts/ALERT-WEEKLY-AUDIT-YYYYMMDD.md."
  }
}
```

## Auto-Repair Delegation

When a failure is detected, delegate repair to the appropriate department:

```json
{
  "name": "DataCenter · Fix [failing-cron]",
  "schedule": {"kind": "at", "at": "2026-01-01T10:00:00+08:00"},
  "sessionTarget": "isolated",
  "deleteAfterRun": true,
  "payload": {
    "kind": "agentTurn",
    "message": "You are DataCenter. Fix cron [jobId]: [error description]. Diagnose, fix, verify. Output: ✅ Fixed / ❌ Needs CEO."
  }
}
```

Set `deleteAfterRun: true` for one-shot repair tasks.

## References

- See [references/common-errors.md](references/common-errors.md) for error catalog
- See [references/repair-playbook.md](references/repair-playbook.md) for step-by-step repair guides
