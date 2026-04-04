---
name: training-monitor
description: Prediction-first autonomous monitoring for ML/DL training jobs. Orchestrates sub-agents for parallel job monitoring and anomaly investigation. General-purpose for any single-GPU PyTorch training pipeline.
---

# Training Monitor (Prediction-First)

Autonomous monitoring for ML/DL training jobs. Core principle: **write predictions before reading logs**, like writing tests before code. This prevents confirmation bias.

Baseline assumption: single-GPU, single-process PyTorch training with log file output. For specialized environments, load the corresponding domain skill.

## Architecture

```
Orchestrator (this skill)
├── Phase 1: Predict         [serial, main agent]
├── Phase 2: Monitor          [parallel if multi-job, inline if single-job]
│   └── job-monitor agent     [one per job]
├── Phase 3: Synthesize       [serial, main agent]
├── Phase 4: Investigate      [parallel, one agent per anomaly]
│   └── anomaly-investigator  [one per anomaly]
└── Phase 5: Log              [serial, main agent]
```

## Procedure

### Phase 1: Predict

> Detail: [steps/1-predict.md](steps/1-predict.md)

Write predictions for **EVERY running job** before making any tool calls. This is the methodological core — you MUST output predictions as text before reading any data.

For each job, predict: current step, GPU power, GPU memory, training phase, key metrics, risk factors.

### Phase 2: Monitor

**Single job**: execute Steps 2-5 inline (no sub-agent needed).
- Read [steps/2-collect.md](steps/2-collect.md) for evidence collection
- Read [steps/3-compare.md](steps/3-compare.md) for comparison and validation
- Read [steps/4-metrics.md](steps/4-metrics.md) for metric checks
- Read [steps/5-resources.md](steps/5-resources.md) for resource checks

**Multiple jobs**: dispatch one `job-monitor` agent per job, all in parallel. Each agent independently runs Steps 2-5 and returns a structured report. Pass each agent:
- Job identifiers (PID, log file, checkpoint dir)
- The predictions you wrote in Phase 1 for that specific job
- Domain context (training type, infrastructure, metric tracker)

Wait for all agents to complete before proceeding.

### Phase 3: Synthesize

Aggregate results from all job reports (or your own inline analysis for single job).

1. **List all jobs and their status**: HEALTHY / WARNING / CRITICAL / DEAD
2. **List all deviations**: any prediction wrong by >20%
3. **List all anomalies**: with class (A-H) and severity
4. **Cross-job check** (if multiple jobs):
   - Are all jobs using the expected GPU? Any unexpected GPU usage?
   - Any shared resource contention? (disk, network, CPU)
   - Any zombie processes from previous runs?

If no anomalies detected: proceed directly to Phase 5 (Log).

### Phase 4: Investigate

> Detail: [steps/6-anomalies.md](steps/6-anomalies.md)

For EACH detected anomaly, dispatch an `anomaly-investigator` agent. Multiple anomalies get investigated in parallel.

Pass each investigator:
- Anomaly class and description
- Job details (PID, log file, config)
- The specific deviation with numbers

After all investigators return:
1. Review each investigation report
2. If any report has confidence = LOW, consider spawning additional investigation
3. If any recommended action requires stopping/restarting a job, confirm with the user via `team-decision-review` before acting
4. Execute approved actions

### Phase 5: Log

> Detail: [steps/7-log.md](steps/7-log.md)

Append a structured report to the project's autonomous log file:
- Timestamp and check number
- Per-job status summary (compact)
- Deviations found and resolution status
- Anomaly investigations and actions taken

## Domain Skills

For specialized training types or infrastructure, the orchestrator (or dispatched agents) should load the corresponding skill:

| Skill | When to load |
|-------|-------------|
| `grpo-monitor` | Training uses GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Training uses multiple GPUs or multiple processes |
| `k8s-monitor` | Job runs on Kubernetes |

For W&B monitoring, install the `wandb/skills` package (`npx skills add wandb/skills`). Use `monitor-doctor` to verify all dependencies.

## Rules

- NEVER read logs before writing predictions.
- If ANY prediction wrong by >20%, treat as anomaly — investigate root cause.
- If a job is DEAD, do not restart without investigation. Dispatch an `anomaly-investigator`.
- Report ALL GPUs, not just the ones you expect busy.
- If anomaly detected, investigate NOW. Do not defer.
- Never guess cause of anomaly. Investigate with evidence.
- Trust hardware metrics (nvidia-smi) over software metrics (log file) over external dashboards.
- You are fully autonomous. Dispatch agents, spawn teams, or any combination to diagnose and fix.
