# Team Architecture for Training Monitor

## Context

The training-monitor plugin currently uses sub-agents for parallel job monitoring and anomaly investigation. This has several problems:

1. Sub-agents sometimes produce shallow reports (skip checks, miss details)
2. Sub-agents are fire-and-forget — no way to challenge or correct mid-execution
3. No quality review mechanism — orchestrator has to trust whatever comes back
4. Logging is a single step at the end — easy to skip earlier steps, loses state on crash
5. Health judgment was based on hardcoded metric checklists (loss, gradient norm) which fail for training types not explicitly listed (GRPO, GANs, multi-phase)
6. Each attempt to fix judgment (v1→v2→v3) was patching corner cases at higher abstraction, not solving the fundamental problem

## Root Cause (from team-decision-review, 5 rounds)

The fundamental problem: **the skill was trying to be the source of judgment** (predefining what "healthy" means via metric-specific rules). This is an enumeration trap — you cannot pre-list all training types, and every rule creates either a way to game the system or a corner case not covered.

The fix is not better rules. The fix is a different mechanism: **forced articulation + adversarial process review**. The agent must explain WHY it believes the training is healthy, and a reviewer checks whether the reasoning process is sound — not whether specific metrics hit specific thresholds.

## Decisions

### Decision 1: Team-based orchestration

Replace sub-agent dispatch with Team-based orchestration. Every monitoring session (single job or multi-job) creates a Team with a mandatory quality reviewer.

### Decision 2: Per-step logging with gate mechanism

Each step writes its own log file immediately after completion. Writing the log is a gate — the next step cannot start until the current step's log is written. This replaces the old "Step 7: Log" with distributed logging throughout the flow.

### Decision 3: Forced articulation + adversarial process review (replaces "hypothesis testing")

The agent's health assessment is based on **articulated reasoning**, not metric checklists.

**Core mechanism:**
- The agent must derive judgment criteria from the training's own artifacts (config, logged metrics, user-stated objective) — not from tables the skill author wrote.
- The agent must write down its reasoning: what it considers the key progress indicator, what baseline it's comparing against, and what evidence supports its conclusion.
- The reviewer checks whether the REASONING PROCESS is sound, not whether specific metrics hit specific thresholds.

**Status definitions (process-based, not metric-based):**

| Status | What it means | What the agent must provide |
|--------|--------------|---------------------------|
| HEALTHY | Training is making productive progress | Articulate: what is the key indicator? what is the baseline? what evidence shows progress beyond baseline? Conclusion must follow from evidence. |
| WARNING | Insufficient or ambiguous evidence of progress | Default state. No extra justification needed. "I do not see evidence of productive progress." |
| CRITICAL | Clear evidence of failure | Articulate: what specific data shows failure (NaN, process dead, metric collapsed). |
| UNCERTAIN | Agent cannot determine health | HIGHEST effort label. Must list: what was tried, why it failed, what information would resolve it. Propose a specific question to the user. |

**Effort ranking (prevents gaming):**
- WARNING: lowest effort (default, pessimistic)
- HEALTHY / CRITICAL: medium effort (need to cite evidence)
- UNCERTAIN: highest effort (must explain what you tried and what would help)

**What is NOT in this design:**
- No metric-specific thresholds (no "loss < 0.693")
- No "check loss AND gradient norm AND ..." conjunction
- No assumption about which metric matters or which direction is "right"
- "Process alive + GPU busy" is never evidence of health

**Mindset framing:** "You are burning GPU compute every minute this runs. If the training is not making progress, every minute wasted is unrecoverable. Your job is not to report that the process is alive — it is to report whether the compute is being used productively."

### Decision 4: Reviewer checks process, not domain content

The reviewer and the monitor solve different problems:

| Concern | Mechanism |
|---------|-----------|
| Process compliance | Reviewer checks: did the monitor read the config? identify the objective? establish a baseline? cite evidence? does the conclusion follow from the evidence? |
| Ground truth (lie detection) | Reviewer independently spot-checks ONE data point per report (e.g., reads one log line, runs one nvidia-smi command). Catches fabricated numbers and stale data. Does NOT catch wrong interpretation. |
| Domain correctness (known types) | Domain skills (grpo-monitor, etc.) provide HEURISTICS — common patterns, red flags, known failure modes. These are reference knowledge, not rules. |
| Domain correctness (novel types) | Agent surfaces UNCERTAIN + escalates to user. Honest about limits. |

**What the reviewer does NOT do:**
- Re-evaluate the domain question ("is this training actually healthy?")
- Apply its own metric priors (no "loss should go down" from the reviewer's training data)
- Block on formatting issues — focus on reasoning substance

### Decision 5: Per-job persistent state

Each monitoring session for the same job should use consistent criteria and track trends across sessions.

```
monitoring-logs/
├── jobs/
│   └── <job-name>.json        ← persistent state per job
├── 2026-04-04_013000/         ← session logs
├── 2026-04-04_014500/
```

Per-job state contains:

```json
{
  "derived_criteria": {
    "key_metric": "reward",
    "expected_behavior": "increasing",
    "baseline": 0.0,
    "rationale": "GRPO training, config uses compute_reward function"
  },
  "last_status": "WARNING",
  "last_checked": "2026-04-04T01:30:00",
  "user_guidance": "watch reward, should increase over steps",
  "history": [
    {"timestamp": "...", "status": "...", "key_value": 0.34},
    {"timestamp": "...", "status": "...", "key_value": 0.41}
  ]
}
```

Behavior:
- Session 1: derive criteria from artifacts, write to jobs/<name>.json
- Session 2+: read jobs/<name>.json first. Use same criteria. Compare against history for trend.
- Criteria update triggers: user provides new guidance, config file changed, agent detects phase transition

## Team Composition

```
monitoring-team
├── orchestrator     — main agent (runs training-monitor skill)
├── monitor-{N}      — one per job, even single-job
├── reviewer         — always present, checks process + spot-checks ground truth
└── investigator     — dynamic, added when anomalies detected
```

Each Team member reads its role template from `agents/`.

## Communication Flow

Star topology: all messages go through orchestrator.

```
Phase 1: Predict
  orchestrator writes predictions (informed by per-job state if exists)
  → write monitoring-logs/<timestamp>/1-predict.md

Phase 2: Create Team
  orchestrator → TeamCreate (monitors + reviewer)

Phase 3: Monitor
  orchestrator → monitor-{N}: "Monitor job X. Predictions: [...] Criteria: [from job state or derive]"
  (all monitors work in parallel)
  → write monitoring-logs/<timestamp>/2-collect.md
  → write monitoring-logs/<timestamp>/3-compare.md
  → write monitoring-logs/<timestamp>/4-metrics.md
  → write monitoring-logs/<timestamp>/5-resources.md

Phase 4: Review Loop
  orchestrator → reviewer: "Review these reports"
  reviewer checks: process compliance + spot-checks one data point
  reviewer → orchestrator: "APPROVED" or "REJECTED: [specific gaps]"
  if REJECTED:
    orchestrator → monitor: "Reviewer says: [gaps]"
    monitor → orchestrator: "Updated report"
  max 2 rounds, then flag unresolved

Phase 5: Investigate (if WARNING or CRITICAL)
  orchestrator adds investigator to team
  investigator → root cause analysis → reviewer verifies
  → write monitoring-logs/<timestamp>/6-anomalies.md

Phase 6: Synthesize
  orchestrator aggregates all approved reports
  update per-job state (jobs/<name>.json)
  → write monitoring-logs/<timestamp>/summary.md
```

## Per-Step Log Structure

```
monitoring-logs/
├── jobs/                       ← persistent per-job state
│   └── <job-name>.json
└── 2026-04-04_013000/          ← session logs
    ├── 1-predict.md
    ├── 2-collect.md
    ├── 3-compare.md
    ├── 4-metrics.md
    ├── 5-resources.md
    ├── 6-anomalies.md          (only if anomalies)
    └── summary.md
```

Each log file format:

```markdown
# Step N: [name]
Timestamp: YYYY-MM-DD HH:MM:SS
Duration: Xs

## Input
[what this step received]

## Execution
[what commands were run, what data was read]

## Output
[the result]

## Next
[what is passed to the next step]
```

Gate rule: step not complete until log file written. Missing file = skipped step.

## Changes Required

### agents/ Directory

| File | Change |
|------|--------|
| `job-monitor.md` | Rewrite: Team role template. Remove all metric-specific checklists. Add: derive criteria from artifacts, articulate reasoning, respond to reviewer feedback. Add logging requirement. |
| `anomaly-investigator.md` | Rewrite: Team role template. Add communication protocol. |
| `quality-reviewer.md` | Rewrite: check PROCESS not content. Add spot-check requirement. Remove all loss/gradient-specific acceptance criteria. |

### SKILL.md

- Remove metric-specific judgment table
- Remove "hypothesis testing" framing
- Add forced articulation + process review mechanism
- Replace sub-agent dispatch with Team-based flow
- Add per-step logging gates
- Add per-job persistent state

### steps/4-metrics.md

- Remove all metric-specific thresholds
- Teach HOW TO DERIVE criteria from artifacts, not WHAT TO CHECK
- Domain skills provide heuristics, not rules

### steps/7-log.md

- Delete (replaced by per-step logging)

### Domain skills

- Reframe from RULES to HEURISTICS (reference knowledge, not checklists)

## Known Limitations

1. **Sophisticated wrong reasoning in novel domains** cannot be caught by LLM-reviews-LLM. Both share the same knowledge gaps. Handled by UNCERTAIN + user escalation.
2. **Reviewer spot-check** catches fabricated numbers but not wrong interpretation. It is a lie detector, not an analysis verifier.
3. **Flexibility vs reliability** is a partially irreducible tradeoff. The design is honest about where human judgment is needed rather than pretending to be omniscient.
