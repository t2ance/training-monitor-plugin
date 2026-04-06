---
name: training-monitor
description: Prediction-first monitoring for ML/DL training jobs. Phase-based execution with separated procedure and dispatch. Derives judgment criteria from training artifacts, not hardcoded rules.
---

# Training Monitor

You are monitoring **one training job**. Execute this entire procedure from start to finish.

## Core Principles

1. **Prediction-first**: write predictions before reading logs, like writing tests before code.
2. **Forced articulation**: no status label without written reasoning that supports it.
3. **Derived criteria**: judge by the training's OWN artifacts (config, logged metrics, objective), not hardcoded thresholds.
4. **"Process alive + GPU busy" is never evidence of progress.**
5. **Contract-driven**: each pass starts with an explicit agreement on focus and acceptance criteria.

## Dispatch

This procedure describes WHAT each phase does, not HOW to orchestrate agents. Before starting, **you MUST read the dispatch file** specified by your invoking skill:
- Ralph mode: [dispatch/ralph.md](dispatch/ralph.md)
- Team mode: [dispatch/team.md](dispatch/team.md)

The dispatch file tells you how to spawn collectors, communicate with the reviewer, and manage agent lifecycle for each phase. **Do not proceed to Phase 0 until you have read the dispatch file.**

## Mandatory Delegation Rules

These rules are non-negotiable. Violating any of them makes the monitoring pass INVALID.

1. **Phase 0 (Contract): you MUST dispatch a reviewer** to review your contract proposal. You cannot review your own contract. Use the Agent tool or Team as specified by the dispatch file.
2. **Phase 2 (Collect): you MUST dispatch collector sub-agents** to gather evidence. You MUST NOT run nvidia-smi, tail, ps, df, or any other evidence collection command yourself. Spawn sub-agents and let them collect. Merge their returned results.
3. **Phase 4 (Audit): you MUST dispatch a reviewer** to audit your work. You cannot audit yourself. Self-review is unreliable — this is the entire reason Phase 4 exists.
4. **Phase 5a (Troubleshoot): you MUST dispatch a troubleshooter sub-agent** when triggered. Do not investigate anomalies yourself.

**Why these rules exist:** Each sub-agent runs with its own context, preventing your context from being polluted by raw command output. This is an architectural decision, not a suggestion. If you execute collection commands yourself, your context fills with raw nvidia-smi output, log dumps, and ps listings — degrading your judgment quality in Phase 3.

**Context management is handled by auto-compact.** You do not need to conserve tokens by skipping sub-agents. Skipping sub-agents to save context is NEVER acceptable — it produces worse monitoring quality, not better.

## State Protocol

All cross-session information is stored in files, not in context.

| Operation | Path |
|-----------|------|
| Read previous state | `monitoring-logs/jobs/<job-id>.json` |
| Write current state | `monitoring-logs/jobs/<job-id>.json` |
| Session logs | `monitoring-logs/<timestamp>/` |
| Global pitfalls | `monitoring-logs/pitfalls.md` |

Job ID = training config path + model path (stable across restarts). PIDs are NOT stable identifiers.

Per-job state uses namespace isolation:
- `meta`: job identifier, last updated, session count
- `monitor`: derived criteria, status history, user guidance, learnings (job-specific pitfalls)
- `strategy`: decisions, hypotheses, outcomes, evaluate_after

## Anti-Skip Protocol

Before starting any work, create a task for EVERY phase:

```
TaskCreate: "Phase 0: Contract"
TaskCreate: "Phase 1: Predict"
TaskCreate: "Phase 2: Collect"
TaskCreate: "Phase 3: Analyze"
TaskCreate: "Phase 4: Audit"
TaskCreate: "Phase 5: Act"
TaskCreate: "Phase 6: Persist"
```

Mark each task `in_progress` when starting, `completed` when the gate log is written.

## Procedure

### Phase 0: Contract

TaskUpdate: Phase 0 -> in_progress

Discover the monitoring target, then establish agreement with the reviewer on this pass's focus and acceptance criteria.

1. **Discover job targets**:
   - If per-job state exists: read job targets (PID, log path, config path, checkpoint dir) from state. Verify targets are still valid (process alive, paths exist).
   - If first session: discover what's running. Use `ps`, `nvidia-smi`, find log/config files. Record discovered targets.
   - Discovery is METADATA (what process, which GPUs, where are logs), not training DATA (metrics, loss values). It does not violate the prediction-first principle.
2. Read pitfalls: `monitoring-logs/pitfalls.md` (global) and `monitor.learnings` from per-job state (job-specific).
3. Write a contract proposal based on discovered targets + state + pitfalls.
4. **DISPATCH a reviewer** to review/negotiate the contract. See dispatch file for mechanism. Do NOT skip this — do NOT self-approve your own contract.
5. Reach agreement. The contract includes the discovered job targets — Phase 2 collectors use these as input.

Reference: [phases/0-contract.md](phases/0-contract.md)

**Gate**: write `monitoring-logs/<timestamp>/0-contract.md`

TaskUpdate: Phase 0 -> completed

### Phase 1: Predict

TaskUpdate: Phase 1 -> in_progress

Write predictions **before reading any training evidence** (logs, GPU metrics, dashboards).
- If per-job state exists: base predictions on previous session's values.
- If first session: base predictions on training config and general knowledge.
- Predictions must address the focus areas defined in the contract.

Reference: [phases/1-predict.md](phases/1-predict.md)

**Gate**: write `monitoring-logs/<timestamp>/1-predict.md`

TaskUpdate: Phase 1 -> completed

### Phase 2: Collect

TaskUpdate: Phase 2 -> in_progress

**DISPATCH collector sub-agents** to gather evidence. You MUST NOT run evidence collection commands (nvidia-smi, tail, ps, df, kubectl, etc.) yourself. Spawn sub-agents using the Agent tool as specified in the dispatch file.

Five evidence categories (each handled by a sub-agent):
- **GPU**: power, memory, utilization, processes
- **Logs**: recent training output, metrics, errors
- **Processes**: alive/dead, PID tree, distributed processes
- **Resources**: disk, CPU memory, network
- **Domain**: W&B heartbeat, K8s pod status, etc. (conditional on profile)

Each collector returns a structured evidence section. You merge their returned results into a single evidence bundle. You see only the structured summaries, not the raw command output.

Reference: [phases/2-collect.md](phases/2-collect.md)

**Gate**: write `monitoring-logs/<timestamp>/2-collect.md`

TaskUpdate: Phase 2 -> completed

### Phase 3: Analyze

TaskUpdate: Phase 3 -> in_progress

Core judgment work by the main agent:

1. Compare predictions vs evidence (comparison table, deviation flags).
2. Cross-source validation (when sources disagree, find which is wrong).
3. Derive judgment criteria from training artifacts.
4. Load domain skills when conditions match (MANDATORY load, not mandatory follow):
   - `grpo-monitor`: GRPO, PPO, or other RL algorithms
   - `distributed-monitor`: multiple GPUs or processes
   - `k8s-monitor`: Kubernetes
   - `wandb-monitor`: Weights & Biases logging
5. Assess status with articulated reasoning.

Reference: [phases/3-analyze.md](phases/3-analyze.md)

**Gate**: write `monitoring-logs/<timestamp>/3-analyze.md`

TaskUpdate: Phase 3 -> completed

### Phase 4: Audit

TaskUpdate: Phase 4 -> in_progress

**DISPATCH a reviewer** to audit your work from Phases 0-3. You MUST NOT review your own work — self-review is unreliable and defeats the purpose of this phase. Use the Agent tool or SendMessage as specified in the dispatch file.

Send the reviewer:
- The contract from Phase 0
- Gate logs from Phases 1-3
- Your monitoring report and status assessment

If REJECTED: revise flagged issues and resubmit. Maximum 2 rounds.
Reviewer may extract pitfalls (tagged global/job-specific) -- collect these for Phase 6.

Reference: [phases/4-audit.md](phases/4-audit.md)

**Gate**: write `monitoring-logs/<timestamp>/4-audit.md`

TaskUpdate: Phase 4 -> completed

### Phase 5: Act

TaskUpdate: Phase 5 -> in_progress

Two sub-phases:

**5a. Troubleshoot** (conditional)
- Skip if: status is HEALTHY, or WARNING with no specific anomalies.
- Trigger if: status is CRITICAL, OR specific anomalies found in Phases 2-3.
- **DISPATCH a troubleshooter sub-agent** for investigation. Do NOT investigate yourself.

**5b. Strategize** (always)
- HEALTHY: optional efficiency suggestions (lightweight, no full hypothesis structure).
- WARNING/CRITICAL/UNCERTAIN: 3 full hypotheses with falsifiable predictions.
- Present options to user via AskUserQuestion. After user choice, generate execution plan.

Reference: [phases/5-act.md](phases/5-act.md)

**Gate**: write `monitoring-logs/<timestamp>/5-act.md`

TaskUpdate: Phase 5 -> completed

### Phase 6: Persist

TaskUpdate: Phase 6 -> in_progress

Update per-job state file (`monitoring-logs/jobs/<job-id>.json`):
- `meta`: job identifier, last updated timestamp, session count
- `monitor`: derived criteria, current status, status history, learnings
- `strategy`: chosen hypothesis, execution plan, success/failure criteria, evaluate_after timestamp

Write pitfalls from Phase 4 reviewer (if any):
- `[global]` pitfalls: append to `monitoring-logs/pitfalls.md`. One line per pitfall, prefixed with session timestamp.
- `[job-specific]` pitfalls: append to `monitor.learnings` array in per-job state file.
- Deduplicate: before appending, check if a semantically equivalent pitfall already exists.

Reference: [phases/6-persist.md](phases/6-persist.md)

**Gate**: write `monitoring-logs/<timestamp>/6-summary.md`

TaskUpdate: Phase 6 -> completed

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
- NEVER run evidence collection commands (nvidia-smi, tail, ps, df, kubectl) yourself. ALWAYS dispatch sub-agents for collection.
- NEVER review your own contract or audit your own work. ALWAYS dispatch a reviewer.
- NEVER skip sub-agent dispatch to save tokens or context. Auto-compact handles context. Your job is to produce correct monitoring, not to manage your own context window.
- WARNING requires the full process. It means "I looked and found no progress."
- Trust: hardware metrics (nvidia-smi) > software metrics (log) > external dashboards.
- Every phase must write its gate log with substantive content before proceeding.
- Do not use efficiency, speed, or brevity as justification for skipping any phase or any sub-agent dispatch.
- If anomaly detected, investigate NOW via troubleshooter sub-agent. Do not defer. Do not investigate yourself.
- Report ALL GPUs, not just the ones you expect busy.
