---
name: training-monitor
description: Prediction-first autonomous monitoring for ML/DL training jobs. Uses Team-based orchestration with mandatory quality review. Hypothesis testing judgment — default WARNING, must prove HEALTHY.
---

# Training Monitor (Prediction-First)

Autonomous monitoring for ML/DL training jobs.

**Core principles:**
1. **Prediction-first**: write predictions before reading logs, like writing tests before code.
2. **Hypothesis testing**: assume training is NOT healthy (H0). HEALTHY requires strong positive evidence. "Process alive + GPU busy" is not evidence of health.
3. **Per-step logging gates**: each step writes a log file before proceeding. Missing log = skipped step.

Baseline assumption: single-GPU, single-process PyTorch training with log file output.

## Judgment Standard

You are burning GPU compute every minute this runs. If the training is not making progress, every minute wasted is unrecoverable. Your job is not to report that the process is alive — it is to report whether the compute is being used productively. Default to worried, not reassured.

| Status | Required evidence |
|--------|-------------------|
| HEALTHY | Loss below task baseline AND showing decreasing trend AND gradient norm in normal range |
| WARNING | Loss flat, above baseline, or no trend data yet. Any single metric outside expected range. |
| CRITICAL | NaN/Inf, process dead, loss exploding, entropy collapsed |

**"Process alive + GPU busy" = NOT HEALTHY. It only means the job has not crashed.**

## Architecture

```
monitoring-team (Team)
├── orchestrator     — main agent, coordinates all communication
├── monitor-{N}      — one per job (even single-job)
├── reviewer         — always present, reviews all output
└── investigator     — added dynamically when anomalies detected
```

All communication goes through orchestrator (star topology). Each member reads its role template from `agents/`.

## Procedure

### Phase 1: Setup

1. Create the log directory: `monitoring-logs/<timestamp>/` (format: `YYYY-MM-DD_HHMMSS`)
2. Write predictions for EVERY running job before making any tool calls.
   - Reference: [steps/1-predict.md](steps/1-predict.md)
3. **Gate**: write `monitoring-logs/<timestamp>/1-predict.md` with all predictions.

### Phase 2: Create Team

Create a Team with:
- One `monitor` member per job (reads `agents/job-monitor.md` for role instructions)
- One `reviewer` member (reads `agents/quality-reviewer.md` for role instructions)

### Phase 3: Monitor

Send each monitor their job assignment + predictions via SendMessage.

Each monitor executes Steps 2-5:
- [steps/2-collect.md](steps/2-collect.md) — evidence collection
- [steps/3-compare.md](steps/3-compare.md) — comparison and validation
- [steps/4-metrics.md](steps/4-metrics.md) — metric checks (apply hypothesis testing judgment)
- [steps/5-resources.md](steps/5-resources.md) — resource checks

All monitors work in parallel.

**Gate**: after each monitor reports, orchestrator writes:
- `monitoring-logs/<timestamp>/2-collect.md`
- `monitoring-logs/<timestamp>/3-compare.md`
- `monitoring-logs/<timestamp>/4-metrics.md`
- `monitoring-logs/<timestamp>/5-resources.md`

### Phase 4: Review

Forward all monitor reports to reviewer via SendMessage.

Reviewer checks:
- Completeness: all data sources checked?
- Evidence: every claim backed by a specific number?
- Judgment: does the HEALTHY/WARNING/CRITICAL status match the evidence? (applying hypothesis testing standard)
- Log files: are all gate logs written?

If reviewer rejects: orchestrator forwards feedback to monitor, monitor revises. Max 2 rounds.

### Phase 5: Investigate (if anomalies)

If any job has WARNING or CRITICAL status:

1. Add `investigator` to the Team (reads `agents/anomaly-investigator.md`)
2. Send investigator the anomaly details via SendMessage
3. Investigator returns root cause analysis
4. Forward to reviewer for verification
5. **Gate**: write `monitoring-logs/<timestamp>/6-anomalies.md`

### Phase 6: Synthesize

Aggregate all reviewed reports into a final summary.

**Gate**: write `monitoring-logs/<timestamp>/summary.md`

## Domain Skills

Load the corresponding skill when applicable:

| Skill | When to load |
|-------|-------------|
| `grpo-monitor` | Training uses GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Training uses multiple GPUs or multiple processes |
| `k8s-monitor` | Job runs on Kubernetes |
| `wandb-monitor` | Job logs to Weights & Biases (requires `wandb-primary` from `wandb/skills`) |

Use `monitor-doctor` to verify all dependencies.

## Rules

- NEVER read logs before writing predictions.
- NEVER mark HEALTHY without positive evidence of learning (loss trending down, below baseline).
- If ANY prediction wrong by >20%, treat as anomaly — investigate.
- If a job is DEAD, do not restart without investigation.
- Report ALL GPUs, not just the ones you expect busy.
- If anomaly detected, investigate NOW. Do not defer.
- Never guess cause of anomaly. Investigate with evidence.
- Trust hardware metrics (nvidia-smi) over software metrics (log file) over external dashboards.
- Every step must write its log file before proceeding. No exceptions.
