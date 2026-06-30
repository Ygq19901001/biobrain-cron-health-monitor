# Repair Playbook — 修复操作手册

Step-by-step repair procedures for every failure pattern. Follow exactly. Verify before closing.

---

## Playbook A: Fix Message Tool Failure

**Estimated time**: 5 minutes
**Risk**: Low (payload change only, no data loss)

### Step 1: Identify the failing job
```
cron(action="get", jobId="<failing-job-id>")
→ Note: payload.message content, sessionTarget, delivery config
```

### Step 2: Rewrite the payload
Replace any reference to `message` tool with `sessions_send` to main:
```
Old: "Send daily report to Feishu using message tool"
New: "Send daily report via sessions_send to main session. 禁止使用message工具。"
```

### Step 3: Remove or fix delivery config
If `delivery.mode` is `announce` and it's failing, either:
- Set `delivery.mode` to `none` (let the agent handle delivery via sessions_send)
- Or fix the delivery channel auth

### Step 4: Update the cron
```
cron(action="update", jobId="<job-id>", patch={payload: {message: "<new-message>"}})
```

### Step 5: Test
```
cron(action="run", jobId="<job-id>", runMode="force")
→ Wait 2 minutes → cron(action="get", jobId="<job-id>")
→ Verify: lastRunStatus="ok", consecutiveErrors reset to 0
```

### Verification checklist:
- [ ] Job ran successfully
- [ ] Main session received the message (check sessions_send receipt)
- [ ] No new errors in next scheduled run
- [ ] Update the payload template in your skill/cron-templates to prevent recurrence

---

## Playbook B: Fix Model Provider Failure

**Estimated time**: 10 minutes
**Risk**: Low to Medium (model switch may affect output quality)

### Step 1: Diagnose the exact error
```
cron(action="get", jobId="<job-id>")
→ Note: state.lastError content
```

### Step 2: Check provider status
For the failing model, verify:
- API key is still valid (check provider config)
- Model name hasn't changed (check provider's current model list)
- Provider isn't rate-limiting or down

### Step 3: Select fallback model
Priority order for free-tier internal ops:
1. `zai/glm-4-flash` — most reliable free option
2. `doubao/doubao-seed-2-0-mini` — ByteDance free tier
3. `baidu/ernie-speed` — Baidu permanent free

### Step 4: Apply the fix
Option A — Switch model:
```
cron(action="update", jobId="<job-id>", patch={payload: {model: "zai/glm-4-flash"}})
```

Option B — Add fallback chain:
```
cron(action="update", jobId="<job-id>", patch={payload: {fallbacks: ["doubao/doubao-seed-2-0-mini", "baidu/ernie-speed"]}})
```

### Step 5: Test
```
cron(action="run", jobId="<job-id>", runMode="force")
→ Verify: job completes, output is correct model, no errors
```

### If switch also fails:
Escalate to CEO with: original model, error, attempted fallback, result. Do not continue switching.

---

## Playbook C: Fix File Path Failure

**Estimated time**: 5 minutes
**Risk**: None (filesystem operations only)

### Step 1: Identify the failing path
```
cron(action="get", jobId="<job-id>")
→ Note: state.lastError — the ENOENT or permission error contains the exact path
```

### Step 2: Diagnose
Does the path use template variables? (`{workspace_root_dir}`, `{date}`, etc.)
- YES → Replace with concrete relative path
- NO → Check if parent directory exists

### Step 3a: Create missing directories
```
exec: mkdir -p <parent-directory-path>
```
Or update payload message to instruct the agent to create the directory before writing.

### Step 3b: Fix template variable
Replace `{workspace_root_dir}/company/departments/brand/` with `company/departments/brand/` (relative to workspace root).

### Step 4: Update and test
```
cron(action="update", jobId="<job-id>", patch={payload: {message: "<fixed-message>"}})
cron(action="run", jobId="<job-id>", runMode="force")
→ Verify: output file exists at expected path with expected content
```

---

## Playbook D: Fix Context Overflow

**Estimated time**: 10 minutes
**Risk**: Low (configuration change)

### Step 1: Confirm it's context overflow
- Error mentions "token limit" or "context length" or output is truncated
- `lightContext` is `false` or absent
- Task is complex (report generation, multi-step processing)

### Step 2: Apply lightContext
```
cron(action="update", jobId="<job-id>", patch={payload: {lightContext: true}})
```

### Step 3: If still failing with lightContext
The task itself is too large. Options:
- Split into 2+ smaller cron jobs (sequential, staggered by 5 minutes)
- Reduce output requirements (e.g., 500 words → 300 words)
- Increase `timeoutSeconds`

### Step 4: Test and verify
Run the job and check output completeness.

---

## Playbook E: Fix Rate Limit Cascade

**Estimated time**: 15 minutes
**Risk**: Medium (rescheduling affects all staggered jobs)

### Step 1: Identify the collision
List all jobs with the same cron minute:
```
cron(action="list")
→ Filter: jobs with same schedule minute and same model provider
```

### Step 2: Redistribute
Minimum 2-minute gap between jobs using the same provider. 1-minute gap if different providers.

### Step 3: Update each job
```
cron(action="update", jobId="<job-id>", patch={schedule: {expr: "<new-expr>"}})
```

### Step 4: Document the new stagger
Update your stagger table reference so the next person doesn't re-collapse it.

---

## Playbook F: Fix Silent "Success"

**Estimated time**: 10 minutes
**Risk**: None (payload instruction change)

### Step 1: Confirm it's silent success
- `consecutiveErrors: 0`, `lastRunStatus: "ok"`
- Output file missing, empty, or contains only "HEARTBEAT_OK"

### Step 2: Rewrite payload to be explicit
Add to the agentTurn message:
- "禁止回复HEARTBEAT_OK" (forbid the heartbeat acknowledgment)
- "必须写入文件后退出" (must write file before exit)
- "如果无法完成任务，写入错误原因到文件而不是静默退出"

### Step 3: Simplify the task
Silent success often means the agent was confused. More specific instructions:
```
BAD:  "Search for industry trends and write a report"
GOOD: "搜索2026年6月AI行业3条最新动态。写入3个要点到company/departments/brand/trends_20260626.md。每条≤100字。完成后退出。"
```

### Step 4: Update and test
```
cron(action="update", jobId="<job-id>", patch={payload: {message: "<new-explicit-message>"}})
cron(action="run", jobId="<job-id>", runMode="force")
→ Manually open output file → Verify content exists and is on-topic
```

---

## When All Playbooks Fail

If you've followed the playbook and the job still fails:
1. Disable the job: `cron(action="update", jobId="<job-id>", patch={enabled: false})`
2. Write incident report to `company/shared/alerts/INCIDENT_<jobId>_YYYYMMDD.md`
3. Include: job config, error history, what you tried, why it didn't work
4. Escalate to CEO with the incident report path
5. Do NOT keep retrying. A disabled job is honest. A failing job is lying.
