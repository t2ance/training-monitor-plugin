# Team Architecture for Training Monitor

## Context

The training-monitor plugin currently uses sub-agents for parallel job monitoring and anomaly investigation. This has three problems:
1. Sub-agents sometimes produce shallow reports (skip checks, miss details)
2. Sub-agents are fire-and-forget — no way to challenge or correct mid-execution
3. No quality review mechanism — orchestrator has to trust whatever comes back

Additionally, logging is a single step at the end, which:
4. Makes it easy to skip earlier steps without evidence
5. Loses intermediate state if the agent crashes mid-session
6. Provides no gate mechanism to enforce step completion

## Decisions

### Decision 1: Team-based orchestration

Replace sub-agent dispatch with Team-based orchestration. Every monitoring session (single job or multi-job) creates a Team with a mandatory quality reviewer.

### Decision 2: Per-step logging with gate mechanism

Each step writes its own log file immediately after completion. Writing the log is a gate — the next step cannot start until the current step's log is written. This replaces the old "Step 7: Log" with distributed logging throughout the flow.

### Decision 3: Hypothesis testing judgment

The agent's health assessment follows the logic of hypothesis testing:

- **H0 (null hypothesis)**: the training is NOT healthy. This is the default state.
- **H1 (alternative)**: the training IS healthy.
- **Burden of proof is on H1**: strong positive evidence is required to mark a job as HEALTHY. Absence of obvious failure is NOT sufficient.

Status definitions:

| Status | Meaning | Required evidence |
|--------|---------|-------------------|
| HEALTHY | Training is making productive progress | Loss below task baseline AND showing decreasing trend AND gradient norm in normal range |
| WARNING | Insufficient evidence of progress, or mild anomaly | Loss flat, above baseline, or no trend data yet. Any single metric outside expected range. |
| CRITICAL | Clear evidence of failure | NaN/Inf, process dead, loss exploding, entropy collapsed |

The practical effect: "process alive + GPU busy" is NOT evidence of health. It only means the job hasn't crashed. The agent must check whether the training is actually LEARNING before marking HEALTHY.

Mindset framing for the agent: "You are burning GPU compute every minute this runs. If the training is not making progress, every minute wasted is unrecoverable. Your job is not to report that the process is alive — it is to report whether the compute is being used productively. Default to worried, not reassured."

## Team Composition

```
monitoring-team
├── orchestrator     — main agent (runs training-monitor skill)
├── monitor-{N}      — one per job, even single-job
├── reviewer         — always present, reviews all output
└── investigator     — dynamic, added when anomalies detected
```

Each Team member reads its role template from `agents/`:
- monitor reads `agents/job-monitor.md`
- reviewer reads `agents/quality-reviewer.md`
- investigator reads `agents/anomaly-investigator.md`

## Communication Flow

Star topology: all messages go through orchestrator.

```
Phase 1: Predict
  orchestrator writes predictions (solo, before Team creation)
  → write monitoring-logs/<timestamp>/1-predict.md

Phase 2: Create Team
  orchestrator → TeamCreate (monitors + reviewer)

Phase 3: Monitor
  orchestrator → monitor-{N}: "Monitor job X. Predictions: [...]"
  (all monitors work in parallel)
  each monitor writes its evidence → orchestrator collects
  → write monitoring-logs/<timestamp>/2-collect.md

Phase 4: Compare & Metrics & Resources
  orchestrator (or monitors) compare predictions vs actuals
  → write monitoring-logs/<timestamp>/3-compare.md
  → write monitoring-logs/<timestamp>/4-metrics.md
  → write monitoring-logs/<timestamp>/5-resources.md

Phase 5: Review Loop
  orchestrator → reviewer: "Review these reports: [...]"
  reviewer → orchestrator: "APPROVED" or "REJECTED: need X, Y, Z"
  if REJECTED:
    orchestrator → monitor: "Reviewer says: need X, Y, Z"
    monitor → orchestrator: "Updated report: [...]"
  max 2 rounds, then flag unresolved

Phase 6: Investigate (if anomalies)
  orchestrator adds investigator to team
  orchestrator → investigator: "Investigate: [...]"
  investigator → orchestrator: "Root cause: [...]"
  orchestrator → reviewer: "Verify: [...]"
  reviewer → orchestrator: "APPROVED" or "Need more evidence"
  → write monitoring-logs/<timestamp>/6-anomalies.md

Phase 7: Synthesize
  orchestrator aggregates all approved reports
  → write monitoring-logs/<timestamp>/summary.md
```

## Per-Step Log Structure

Each monitoring session creates a timestamped folder:

```
monitoring-logs/
└── 2026-04-04_013000/
    ├── 1-predict.md
    ├── 2-collect.md
    ├── 3-compare.md
    ├── 4-metrics.md
    ├── 5-resources.md
    ├── 6-anomalies.md    (only if anomalies detected)
    └── summary.md
```

Each log file follows this format:

```markdown
# Step N: [name]
Timestamp: YYYY-MM-DD HH:MM:SS
Duration: Xs

## Input
[what this step received / was supposed to do]

## Execution
[what commands were run, what data was read]

## Output
[the result of this step]

## Next
[what is passed to the next step]
```

Gate rule: a step is not complete until its log file is written. The orchestrator must verify the log file exists before proceeding.

Missing file = step was skipped. Reviewer checks for completeness.

## Changes to agents/ Directory

Three files, same content core, changed framing:

| File | Change |
|------|--------|
| `job-monitor.md` | Sub-agent definition → Team role template. Remove `allowed-tools` frontmatter. Add communication protocol (report to orchestrator, respond to reviewer feedback). Add logging requirement (write log for each step). |
| `anomaly-investigator.md` | Same treatment. Add communication protocol and logging. |
| `quality-reviewer.md` | Already written as Team role. Add log completeness checking to review criteria. |

No new files needed.

## Changes to SKILL.md

- Replace sub-agent dispatch with Team-based flow (TeamCreate, SendMessage)
- Remove Phase 5 (Log) — replaced by per-step logging gates
- Add log directory creation at the start of each session
- Add gate rule: "write log file before proceeding to next step"

## Changes to steps/ Directory

- Delete `steps/7-log.md` — no longer a separate step
- Other step files unchanged (still reference material for how to execute each step)

## What Does NOT Change

- Phase 1 (Predict): still serial, still orchestrator-only
- steps/1-6.md: step detail files unchanged (still reference material)
- Domain skills (grpo-monitor, k8s-monitor, etc.): unchanged
- monitor-doctor: unchanged
- plugin.json, marketplace.json: unchanged
