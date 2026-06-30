# Common Errors — 故障模式目录

Not a list. A diagnostic tool. Start with the error message, follow the flow, arrive at the fix.

## Error A: Message Tool Failure

**Real error messages you'll see:**
```
Message tool error: channel feishu not configured in isolated session
Message failed: auth token not found
delivery.mode=announce resulted in Message send error
```

**Diagnostic flow:**
```
Is the cron running in isolated session? → YES: sessions_send to main instead
                                       → NO:  Check channel auth configuration
Is sessions_send also failing? → YES: Check that target session is alive (sessions_list)
                               → NO:  Fixed. Update payload template.
```

**Root cause stack** (deepest to shallowest):
1. Isolated sessions don't inherit channel credentials from main
2. The `message` tool requires channel-specific auth that only exists in the main session's provider config
3. `delivery.mode=announce` uses the same message infrastructure

**Permanent fix pattern:**
```json
// BEFORE (broken):
"payload": {
  "kind": "agentTurn",
  "message": "Send daily report via message tool to feishu channel..."
}

// AFTER (fixed):
"payload": {
  "kind": "agentTurn",
  "message": "Send daily report via sessions_send to main session. Do NOT use message tool."
}
```

---

## Error B: Model Provider Failures

**Real error messages you'll see:**
```
LLM request failed: 401 Unauthorized
model 'zai/glm-4-flash' not found in provider list
429 Too Many Requests — rate limit exceeded
503 Service Unavailable
Request timeout after 60000ms
```

**Diagnostic flow:**
```
Is error 401? → API key expired or revoked. Update key in provider config.
Is error 404 (model not found)? → Model deprecated. Switch to current model via provider list.
Is error 429? → Rate limit. Either add stagger or switch providers.
Is error 503? → Provider outage. Switch to fallback model.
Is it a timeout? → Provider slow. Increase timeoutSeconds or switch to faster provider.
```

**Fallback model priority** (free tier):
1. `zai/glm-4-flash` (Zhipu, permanent free)
2. `doubao/doubao-seed-2-0-mini` (ByteDance, free)
3. `baidu/ernie-speed` (Baidu, permanent free)

**Adding fallback to a cron job:**
```json
{
  "payload": {
    "model": "zai/glm-4-flash",
    "fallbacks": ["doubao/doubao-seed-2-0-mini", "baidu/ernie-speed"]
  }
}
```

---

## Error C: File Path Failures

**Real error messages you'll see:**
```
ENOENT: no such file or directory, open 'company/departments/brand/学习笔记_20260625.md'
Write failed: permission denied
Directory not found: {workspace_root_dir}/output/
```

**Diagnostic flow:**
```
Does the path contain template variables? → YES: Replace with absolute/relative paths
Does the directory exist? → NO: Create it or add mkdir to payload instructions
Is it a permission issue? → Check file ownership, especially on first run
```

**Path resolution in isolated sessions:**
Isolated sessions may resolve `{workspace_root_dir}` differently than main. Always test a new cron job's file output manually before trusting it.

---

## Error D: Context Overflow

**Real error messages you'll see:**
```
Context length exceeded. Input too long.
Output truncated at token limit.
Task partially complete — ran out of context.
```

**Diagnostic flow:**
```
Is lightContext set? → NO: Set lightContext: true immediately
Is the task inherently large? → YES: Split into smaller jobs
Does the job need history? → If not, lightContext. If yes, reduce timeoutSeconds and simplify task.
```

**Rule of thumb**: If the job doesn't need to know what happened yesterday, it doesn't need full context.

---

## Error E: Silent "Success"

**Real symptoms:**
```
consecutiveErrors: 0
lastRunStatus: "ok"
But: output file is empty or contains only "HEARTBEAT_OK"
```

**Diagnostic flow:**
```
Open the expected output file → Empty or "HEARTBEAT_OK"? → Payload message needs "禁止回复HEARTBEAT_OK"
File exists but wrong content? → Agent hallucinated. Try more specific instructions in payload.
File doesn't exist at all? → Path resolution issue (see Error C).
```

**The HEARTBEAT_OK plague:**
Many agents, when given an empty or minimal payload, will respond with a polite acknowledgment and exit. The cron framework marks this as success. Your file system is empty. Detection requires Inspector content verification — not just error checking.

---

## Error F: Rate Limit Cascades

**Real symptoms:**
```
3+ jobs fail at exactly 22:00:00
All error messages mention "rate limit" or "429"
Jobs that normally succeed are now failing together
```

**Diagnostic flow:**
```
What's the stagger interval? → 0 minutes → That's the bug
                              → < 2 minutes → Increase to 120 seconds minimum
                              → ≥ 2 minutes → Check if provider rate limit changed (monthly audit)
```

**Hidden cause**: Provider rate limits change without notice. A stagger that worked last month may fail this month.

---

## Error F (Variant): Cron Framework Rate Limit

**Symptom**: Jobs fail during the `cron(action="add")` or `cron(action="update")` phase — not during execution.
**Cause**: The cron management API itself has rate limits.
**Fix**: Add 60-second delays between bulk cron operations. Batch no more than 3 creates/updates per minute.
