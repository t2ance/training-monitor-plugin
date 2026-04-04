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
4. **Per-step logging gates**: each step writes a log file before proceeding. Missing log = skipped step. Gate logs must have substantive content in all sections (Input, Execution, Output, Next) — empty sections fail the gate.

Baseline assumption: single-GPU, single-process PyTorch training with log file output.

## Judgment Standard

You are burning GPU compute every minute this runs. If the training is not making progress, every minute wasted is unrecoverable. Your job is not to report that the process is alive — it is to report whether the compute is being used productively.

**The agent derives judgment criteria from the training's own artifacts** (config, logged metrics, user-stated objective). The skill does NOT predefine which metrics to check or what thresholds to use.

| Status | What the agent must provide |
|--------|---------------------------|
| HEALTHY | Articulate: what is the key progress indicator? what is the expected behavior? what is the baseline? what evidence shows progress beyond baseline? Conclusion must follow from evidence. |
| WARNING | The process (derive criteria, collect evidence, analyze) must still be completed. WARNING means "I looked and didn't find progress," not "I didn't look." The absence of positive evidence is itself the justification — but the work must be done. |
| CRITICAL | Articulate: what specific data shows failure (NaN, process dead, metric collapsed). |
| UNCERTAIN | HIGHEST effort. Must list: what was tried to derive criteria, why it failed, what information would resolve it. Propose a specific question to the user. |

**"Process alive + GPU busy" is never evidence of progress.**

## Architecture

```
monitoring-team (Team)
├── orchestrator     — main agent, coordinates all communication
├── monitor-{N}      — one per job, derives criteria + collects evidence
├── reviewer         — always present, checks PROCESS compliance + spot-checks
├── troubleshooter   — added dynamically for technical failures (bugs, OOM, crashes)
└── strategist       — added dynamically for performance optimization (suboptimal metrics)
```

Star topology: all messages go through orchestrator.

## Procedure

### Phase 1: Setup

1. **Determine stable job identifiers.** A job is identified by its training config path + model path (or a user-provided stable name). PIDs are NOT stable identifiers — they change on restart. The job identifier is used for per-job state filenames.
2. Check for per-job state: read `monitoring-logs/jobs/<job-id>.json` if it exists (criteria, history, user guidance from previous sessions).
3. Create the session log directory: `monitoring-logs/<timestamp>/` (format: `YYYY-MM-DD_HHMMSS`). For multi-job monitoring, create subdirectories: `monitoring-logs/<timestamp>/job-<id>/`.
4. Write predictions for EVERY running job **before reading any training evidence** (logs, GPU metrics, metric dashboards). Reading per-job state (step 2) is setup, not evidence collection.
   - If per-job state exists: base predictions on previous session's values.
   - If first session: base predictions on training config and general knowledge.
   - Reference: [steps/1-predict.md](steps/1-predict.md)
5. **Gate**: write `monitoring-logs/<timestamp>/1-predict.md`

### Phase 2: Create Team

Create a Team with:
- One `monitor` member per job (reads `agents/job-monitor.md` for role instructions)
- One `reviewer` member (reads `agents/quality-reviewer.md` for role instructions)

### Phase 3: Monitor

Send each monitor via SendMessage:
- Stable job identifier
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

**Gate**: orchestrator writes gate logs after receiving monitor reports. Single job: write directly to `monitoring-logs/<timestamp>/`. Multi-job: write per-job to `monitoring-logs/<timestamp>/job-<id>/`. The orchestrator records the monitor's full evidence and reasoning — not a one-line summary.
- `2-collect.md`
- `3-compare.md`
- `4-metrics.md`
- `5-resources.md`

### Phase 4: Review

Forward all monitor reports to reviewer via SendMessage.

Reviewer checks PROCESS, not domain content:
- Did the monitor read the training config / artifacts?
- Did the monitor explicitly state the key progress indicator, expected behavior, and WHY?
- Did the monitor establish a baseline?
- Does the conclusion follow from the stated evidence? (logical coherence — does the stated expected behavior match the stated conclusion?)
- Spot-check: reviewer independently verifies the claim that the APPROVED/REJECTED decision hinges on. State what was checked, how, and the result.
- Are gate log files present with substantive content?

If reviewer rejects: orchestrator forwards specific feedback to monitor. Max 2 rounds. After 2 rounds, the reviewer's "what remains unverified" note goes into the final summary.

### Phase 5: Troubleshoot

Triggers when: (1) status is CRITICAL, OR (2) the monitor's report lists specific anomalies or deviations. Does NOT trigger on default WARNING with no specific findings — that indicates the monitor found nothing alarming but also couldn't confirm progress.

1. Add `troubleshooter` to the Team (reads `agents/troubleshooter.md`)
2. Send anomaly details via SendMessage
3. Troubleshooter returns root cause analysis
4. Forward to reviewer for verification
5. **Gate**: write `monitoring-logs/<timestamp>/6-anomalies.md`

### Phase 6: Strategize

Triggers on ALL statuses after Phase 4 (Review) completes. If Phase 5 (Troubleshoot) was triggered, Phase 6 waits for Phase 5 to complete and includes the troubleshooter report as input to the strategist.

For multi-job monitoring: spawn one strategist per job (matching the monitor pattern). All strategists run in parallel. The orchestrator collects all strategy reports and presents them to the user in priority order:
1. PIVOT jobs first (highest urgency)
2. INVESTIGATE jobs second
3. REFINE jobs third
4. CONTINUE jobs: skip user interaction entirely (record "no changes needed" automatically)

This means the user only answers questions for jobs that need strategic decisions.

The strategist proposes next-step hypotheses regardless of health status:
- HEALTHY: optimization suggestions (efficiency, throughput)
- WARNING: hypotheses for why training isn't progressing
- CRITICAL: after troubleshooter addresses immediate failure, strategic direction for recovery
- UNCERTAIN: diagnostic experiments to gather missing information

1. Add `strategist` to the Team (reads `agents/strategist.md`)
2. Send via SendMessage: monitoring report, training config, per-job state, domain context
3. Strategist gathers history, classifies situation, generates 3 hypotheses
4. Strategist presents options to user via AskUserQuestion
5. After user choice: strategist generates execution plan
6. Forward plan to reviewer for process check
7. **Gate**: write `monitoring-logs/<timestamp>/7-strategy.md`

### Phase 7: Synthesize

1. Aggregate all reviewed reports
2. Update per-job state: write `monitoring-logs/jobs/<job-id>.json` using namespace isolation:
   - `meta`: job identifier, last updated timestamp, session count
   - `monitor`: derived criteria, status, history, user guidance (owned by job-monitor)
   - `strategy`: decisions, hypotheses, outcomes, evaluate_after (owned by strategist)
   Each agent reads the whole file but only writes to its own namespace. The orchestrator merges updates from all agents before writing.
3. **Gate**: write `monitoring-logs/<timestamp>/summary.md`

## Domain Skills

You MUST load the corresponding domain skill when the condition matches. Loading is mandatory; following blindly is not. Domain skills provide heuristics (common patterns, red flags, known failure modes) that you reason WITH, not constraints that override your analysis. Skipping them means you miss known failure modes you cannot derive from the training config alone.

| Skill | When to load (MANDATORY) |
|-------|--------------------------|
| `grpo-monitor` | Training uses GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Training uses multiple GPUs or multiple processes |
| `k8s-monitor` | Job runs on Kubernetes |
| `wandb-monitor` | Job logs to Weights & Biases (requires `wandb-primary` from `wandb/skills`) |

Use `monitor-doctor` to verify all dependencies.

## Rules

- NEVER read training evidence before writing predictions.
- NEVER assign a status without written reasoning that supports it.
- WARNING requires the full process to be completed. It means "I looked and found no progress," not "I didn't look."
- HEALTHY and CRITICAL require articulated evidence. UNCERTAIN requires the most effort.
- Flag deviations that would change your health assessment. For numeric metrics: >20% relative OR >10 absolute percentage points (whichever is more meaningful). For categorical predictions: any mismatch. For near-zero values: use absolute difference. State what you flagged and why.
- If a job is DEAD, do not restart without investigation.
- Report ALL GPUs, not just the ones you expect busy.
- If anomaly detected, investigate NOW. Do not defer.
- Never guess cause of anomaly. Investigate with evidence.
- Trust hardware metrics (nvidia-smi) over software metrics (log file) over external dashboards.
- Every step must write its gate log with substantive content before proceeding. Empty sections fail the gate.
