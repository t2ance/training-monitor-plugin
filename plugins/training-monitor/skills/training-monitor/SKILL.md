---
name: training-monitor
description: Prediction-first monitoring for ML/DL training jobs. Single-agent execution with reviewer sub-agent. Derives judgment criteria from training artifacts, not hardcoded rules.
---

# Training Monitor

You are monitoring **one training job**. Execute this entire procedure from start to finish.

## Core Principles

1. **Prediction-first**: write predictions before reading logs, like writing tests before code.
2. **Forced articulation**: no status label without written reasoning that supports it.
3. **Derived criteria**: judge by the training's OWN artifacts (config, logged metrics, objective), not hardcoded thresholds.
4. **"Process alive + GPU busy" is never evidence of progress.**

## State Protocol

All cross-session information is stored in files, not in context.

| Operation | Path |
|-----------|------|
| Read previous state | `monitoring-logs/jobs/<job-id>.json` |
| Write current state | `monitoring-logs/jobs/<job-id>.json` |
| Session logs | `monitoring-logs/<timestamp>/` |

Job ID = training config path + model path (stable across restarts). PIDs are NOT stable identifiers.

Per-job state uses namespace isolation:
- `meta`: job identifier, last updated, session count
- `monitor`: derived criteria, status history, user guidance
- `strategy`: decisions, hypotheses, outcomes, evaluate_after

## Anti-Skip Protocol

Before starting any work, create a task for EVERY step:

```
TaskCreate: "Step 1: Setup + Predictions"
TaskCreate: "Step 2: Collect Evidence"
TaskCreate: "Step 3: Compare Predictions vs Actuals"
TaskCreate: "Step 4: Analyze Metrics"
TaskCreate: "Step 5: Check Resources"
TaskCreate: "Step 6: Reviewer Audit"
TaskCreate: "Step 7: Troubleshoot (if needed)"
TaskCreate: "Step 8: Strategize"
TaskCreate: "Step 9: Write State"
```

Mark each task `in_progress` when starting, `completed` when the gate log is written.

## Procedure

### Step 1: Setup + Predictions

TaskUpdate: Step 1 -> in_progress

1. Read per-job state file if it exists (previous session's criteria, history, decisions).
2. Read pitfalls: `monitoring-logs/pitfalls.md` (global) and `monitor.learnings` from per-job state (job-specific). Keep these in mind throughout the pass — they are mistakes from previous sessions that should not be repeated.
3. Create session log directory: `monitoring-logs/<timestamp>/` (format: `YYYY-MM-DD_HHMMSS`).
4. Write predictions for this job **before reading any training evidence** (logs, GPU metrics, dashboards).
   - If per-job state exists: base predictions on previous session's values.
   - If first session: base predictions on training config and general knowledge.
   - Reference: [steps/1-predict.md](steps/1-predict.md)
5. **Gate**: write `monitoring-logs/<timestamp>/1-predict.md`

TaskUpdate: Step 1 -> completed

### Step 2: Collect Evidence

TaskUpdate: Step 2 -> in_progress

Run ALL evidence collection commands in parallel.
Reference: [steps/2-collect.md](steps/2-collect.md)

**Gate**: write `monitoring-logs/<timestamp>/2-collect.md`

TaskUpdate: Step 2 -> completed

### Step 3: Compare Predictions vs Actuals

TaskUpdate: Step 3 -> in_progress

Build comparison table. Flag deviations that would change your health assessment.
Reference: [steps/3-compare.md](steps/3-compare.md)

**Gate**: write `monitoring-logs/<timestamp>/3-compare.md`

TaskUpdate: Step 3 -> completed

### Step 4: Analyze Metrics

TaskUpdate: Step 4 -> in_progress

Derive judgment criteria from the training's own artifacts. Assess health with articulated reasoning.
Reference: [steps/4-metrics.md](steps/4-metrics.md)

Load domain skills when the condition matches (MANDATORY -- loading required, following blindly not):

| Skill | When to load |
|-------|-------------|
| `grpo-monitor` | GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Multiple GPUs or processes |
| `k8s-monitor` | Kubernetes |
| `wandb-monitor` | Weights & Biases logging |

**Gate**: write `monitoring-logs/<timestamp>/4-metrics.md`

TaskUpdate: Step 4 -> completed

### Step 5: Check Resources

TaskUpdate: Step 5 -> in_progress

Reference: [steps/5-resources.md](steps/5-resources.md)

**Gate**: write `monitoring-logs/<timestamp>/5-resources.md`

TaskUpdate: Step 5 -> completed

### Step 6: Reviewer Audit

TaskUpdate: Step 6 -> in_progress

Spawn a **sub-agent** to adversarially review your work from Steps 1-5. The reviewer checks PROCESS, not domain content.

Send the sub-agent:
- Your gate logs from Steps 1-5
- The reviewer checklist: read `agents/quality-reviewer.md`

If REJECTED: revise the flagged issues and resubmit. Maximum 2 rounds.

Reference: [steps/6-review.md](steps/6-review.md)

**Gate**: write `monitoring-logs/<timestamp>/6-review.md`

TaskUpdate: Step 6 -> completed

### Step 7: Troubleshoot (conditional)

**Skip if**: status is HEALTHY, or WARNING with no specific anomalies.
**Trigger if**: status is CRITICAL, OR specific anomalies or deviations were found in Steps 2-5.

TaskUpdate: Step 7 -> in_progress

Investigate the anomaly systematically: observe with numbers, reproduce, isolate root cause, propose concrete action.
Reference: [steps/7-troubleshoot.md](steps/7-troubleshoot.md)

**Gate**: write `monitoring-logs/<timestamp>/7-troubleshoot.md`

TaskUpdate: Step 7 -> completed

### Step 8: Strategize

TaskUpdate: Step 8 -> in_progress

Propose next-step hypotheses based on monitoring results. This step triggers on ALL statuses:
- HEALTHY: optional efficiency suggestions (lightweight, no full hypothesis structure required)
- WARNING/CRITICAL/UNCERTAIN: 3 full hypotheses with falsifiable predictions

Present options to user via AskUserQuestion. After user choice, generate execution plan.
Reference: [steps/8-strategy.md](steps/8-strategy.md)

**Gate**: write `monitoring-logs/<timestamp>/8-strategy.md`

TaskUpdate: Step 8 -> completed

### Step 9: Write State

TaskUpdate: Step 9 -> in_progress

Update per-job state file (`monitoring-logs/jobs/<job-id>.json`):
- `meta`: job identifier, last updated timestamp, session count
- `monitor`: derived criteria, current status, status history, learnings (job-specific pitfalls from reviewer)
- `strategy`: chosen hypothesis, execution plan, success/failure criteria, evaluate_after timestamp

Write pitfalls from Step 6 reviewer (if any):
- `[global]` pitfalls: append to `monitoring-logs/pitfalls.md` (create if doesn't exist). One line per pitfall, prefixed with session timestamp.
- `[job-specific]` pitfalls: append to `monitor.learnings` array in per-job state file.
- Deduplicate: before appending, check if a semantically equivalent pitfall already exists. Skip if so.

**Gate**: write `monitoring-logs/<timestamp>/9-summary.md`

TaskUpdate: Step 9 -> completed

## Judgment Standard

| Status | What you must provide |
|--------|----------------------|
| HEALTHY | Key progress indicator, expected behavior, baseline, evidence of progress beyond baseline. Conclusion follows from evidence. |
| WARNING | Full process completed. "I looked and found no progress," not "I didn't look." |
| CRITICAL | Specific data showing failure (NaN, process dead, metric collapsed). |
| UNCERTAIN | Highest effort. What was tried, why it failed, what would resolve it. Propose a specific question to the user. |

## Rules

- NEVER read training evidence before writing predictions.
- NEVER assign a status without written reasoning.
- WARNING requires the full process. It means "I looked and found no progress."
- Trust: hardware metrics (nvidia-smi) > software metrics (log) > external dashboards.
- Every step must write its gate log with substantive content before proceeding.
- Do not use efficiency, speed, or brevity as justification for skipping any step.
- If anomaly detected, investigate NOW. Do not defer.
- Report ALL GPUs, not just the ones you expect busy.
