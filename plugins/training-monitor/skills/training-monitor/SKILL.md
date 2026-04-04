---
name: training-monitor
description: Prediction-first autonomous monitoring for ML/DL training jobs. Uses Team-based orchestration with mandatory quality review. Agent derives judgment criteria from training artifacts, not from hardcoded rules.
---

# Training Monitor (Prediction-First)

Autonomous monitoring for ML/DL training jobs.

**Core principles:**
1. **Prediction-first**: write predictions before reading logs, like writing tests before code.
2. **Forced articulation**: the agent must explain WHY it believes the training is healthy or not. No status label without written reasoning.
3. **Process review**: the reviewer checks whether the reasoning process is sound, not whether specific metrics hit specific thresholds.
4. **Per-step logging gates**: each step writes a log file before proceeding. Missing log = skipped step.

Baseline assumption: single-GPU, single-process PyTorch training with log file output.

## Judgment Standard

You are burning GPU compute every minute this runs. If the training is not making progress, every minute wasted is unrecoverable. Your job is not to report that the process is alive — it is to report whether the compute is being used productively.

**The agent derives judgment criteria from the training's own artifacts** (config, logged metrics, user-stated objective). The skill does NOT predefine which metrics to check or what thresholds to use.

| Status | What the agent must provide |
|--------|---------------------------|
| HEALTHY | Articulate: what is the key progress indicator? what is the baseline? what evidence shows progress beyond baseline? Conclusion must follow from evidence. |
| WARNING | Default state. "I do not see sufficient evidence of productive progress." No extra justification needed. |
| CRITICAL | Articulate: what specific data shows failure (NaN, process dead, metric collapsed). |
| UNCERTAIN | HIGHEST effort. Must list: what was tried to derive criteria, why it failed, what information would resolve it. Propose a specific question to the user. |

**"Process alive + GPU busy" is never evidence of progress.**

## Architecture

```
monitoring-team (Team)
├── orchestrator     — main agent, coordinates all communication
├── monitor-{N}      — one per job (even single-job), derives criteria + collects evidence
├── reviewer         — always present, checks PROCESS compliance + spot-checks one data point
└── investigator     — added dynamically when anomalies detected
```

Star topology: all messages go through orchestrator.

## Procedure

### Phase 1: Setup

1. Check for per-job state: read `monitoring-logs/jobs/<job-name>.json` if it exists (criteria, history, user guidance from previous sessions).
2. Create the session log directory: `monitoring-logs/<timestamp>/` (format: `YYYY-MM-DD_HHMMSS`)
3. Write predictions for EVERY running job before making any tool calls.
   - If per-job state exists: base predictions on previous session's values.
   - If first session: base predictions on training config and general knowledge.
   - Reference: [steps/1-predict.md](steps/1-predict.md)
4. **Gate**: write `monitoring-logs/<timestamp>/1-predict.md`

### Phase 2: Create Team

Create a Team with:
- One `monitor` member per job (reads `agents/job-monitor.md` for role instructions)
- One `reviewer` member (reads `agents/quality-reviewer.md` for role instructions)

### Phase 3: Monitor

Send each monitor via SendMessage:
- Job identifiers (PID, log file, checkpoint dir)
- Predictions from Phase 1
- Per-job state (if exists) — so the monitor uses consistent criteria
- Domain context (training type, infrastructure)

Each monitor:
1. Derives judgment criteria from training artifacts (or uses per-job state if available)
2. Collects evidence: [steps/2-collect.md](steps/2-collect.md)
3. Compares predictions vs actuals: [steps/3-compare.md](steps/3-compare.md)
4. Checks metrics using derived criteria: [steps/4-metrics.md](steps/4-metrics.md)
5. Checks resources: [steps/5-resources.md](steps/5-resources.md)
6. Writes articulated reasoning for status assessment

All monitors work in parallel.

**Gate**: orchestrator writes after receiving reports:
- `monitoring-logs/<timestamp>/2-collect.md`
- `monitoring-logs/<timestamp>/3-compare.md`
- `monitoring-logs/<timestamp>/4-metrics.md`
- `monitoring-logs/<timestamp>/5-resources.md`

### Phase 4: Review

Forward all monitor reports to reviewer via SendMessage.

Reviewer checks PROCESS, not domain content:
- Did the monitor read the training config / artifacts?
- Did the monitor explicitly state the key progress indicator and WHY?
- Did the monitor establish a baseline?
- Does the conclusion follow from the stated evidence?
- Spot-check: reviewer independently verifies ONE data point (reads one log line or runs one command)

If reviewer rejects: orchestrator forwards specific feedback to monitor. Max 2 rounds.

### Phase 5: Investigate (if WARNING or CRITICAL)

1. Add `investigator` to the Team (reads `agents/anomaly-investigator.md`)
2. Send anomaly details via SendMessage
3. Investigator returns root cause analysis
4. Forward to reviewer for verification
5. **Gate**: write `monitoring-logs/<timestamp>/6-anomalies.md`

### Phase 6: Synthesize

1. Aggregate all reviewed reports
2. Update per-job state: write `monitoring-logs/jobs/<job-name>.json` with derived criteria, status, history, any user guidance
3. **Gate**: write `monitoring-logs/<timestamp>/summary.md`

## Domain Skills

Domain skills provide HEURISTICS (common patterns, red flags, known failure modes) — not rules or checklists. They are reference knowledge that informs the agent's reasoning, not constraints that override it.

| Skill | When to load |
|-------|-------------|
| `grpo-monitor` | Training uses GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Training uses multiple GPUs or multiple processes |
| `k8s-monitor` | Job runs on Kubernetes |
| `wandb-monitor` | Job logs to Weights & Biases (requires `wandb-primary` from `wandb/skills`) |

Use `monitor-doctor` to verify all dependencies.

## Rules

- NEVER read logs before writing predictions.
- NEVER assign a status without written reasoning that supports it.
- WARNING is the default. HEALTHY and CRITICAL require articulated evidence. UNCERTAIN requires the most effort.
- If ANY prediction wrong by >20%, investigate.
- If a job is DEAD, do not restart without investigation.
- Report ALL GPUs, not just the ones you expect busy.
- If anomaly detected, investigate NOW. Do not defer.
- Never guess cause of anomaly. Investigate with evidence.
- Trust hardware metrics (nvidia-smi) over software metrics (log file) over external dashboards.
- Every step must write its log file before proceeding. No exceptions.
