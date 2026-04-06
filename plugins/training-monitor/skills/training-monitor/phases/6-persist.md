# Phase 6: Persist

## Goal

Write all session results to persistent storage. This is the final phase -- after this, the monitoring pass is complete.

## Per-Job State

Update `monitoring-logs/jobs/<job-id>.json`:

```json
{
  "meta": {
    "job_id": "<config_path + model_path>",
    "last_updated": "<ISO timestamp>",
    "session_count": "<N>"
  },
  "monitor": {
    "derived_criteria": {
      "key_metric": "<metric name>",
      "expected_behavior": "<direction/pattern>",
      "baseline": "<value>",
      "rationale": "<why this metric>"
    },
    "last_status": "<HEALTHY/WARNING/CRITICAL/UNCERTAIN>",
    "status_history": [
      {"timestamp": "...", "status": "...", "key_value": "..."}
    ],
    "user_guidance": "<any user-specified monitoring instructions>",
    "learnings": [
      "<job-specific pitfall 1>",
      "<job-specific pitfall 2>"
    ]
  },
  "strategy": {
    "chosen_hypothesis": "<description or null>",
    "execution_plan": "<plan or null>",
    "success_criterion": "<criterion>",
    "failure_criterion": "<criterion>",
    "evaluate_after": "<ISO timestamp>",
    "history": [
      {"timestamp": "...", "hypothesis": "...", "outcome": "CONFIRMED/REFUTED/INCONCLUSIVE/PENDING"}
    ]
  }
}
```

## Pitfalls

Write pitfalls collected from the Phase 4 reviewer:

### Global pitfalls

Append to `monitoring-logs/pitfalls.md` (create if doesn't exist):

```
[2026-04-04 01:30:00] Always check ALL GPUs, not just GPU 0
[2026-04-04 02:00:00] Verify eval ratio before reporting HEALTHY -- high eval time can mask slow training
```

Before appending, check if a semantically equivalent pitfall already exists. Skip duplicates.

### Job-specific pitfalls

Append to `monitor.learnings` array in per-job state file.

## Summary Gate Log

Record in `monitoring-logs/<timestamp>/6-summary.md`:

```
MONITORING SUMMARY: [job name] -- [timestamp]
---
Status: [HEALTHY / WARNING / CRITICAL / UNCERTAIN]
Key indicator: [metric] = [value] (expected: [expected behavior])
Contract compliance: [all focus areas addressed? Y/N]

Phases completed: [0-6 or partial]
Reviewer verdict: [APPROVED / REJECTED (resolved) / REJECTED (unresolved)]
Anomalies: [list or "none"]
Strategy: [chosen hypothesis or "no changes" or "CONTINUE"]

New pitfalls recorded: [count or "none"]
Next evaluation: [evaluate_after timestamp or "N/A"]
---
```
