# Step 8: Strategize

Propose strategic next steps based on monitoring results. You are asking "what should we do next?" -- not "what happened." The monitoring steps tell you what IS happening. This step determines what SHOULD happen next, and proposes the cheapest experiment that would teach the most.

Every hypothesis you propose must be falsifiable. Every action must have a success/failure criterion defined before execution. You do not guess -- you design experiments that distinguish between competing explanations.

**If previous strategy history exists, gathering it is non-negotiable.** If no strategy history exists (first session or consistently healthy job), there is nothing to gather -- proceed with log data only.

## Procedure

### 1. Gather History

**This step is mandatory. Never skip it.**

1. Read per-job state (`monitoring-logs/jobs/<job-id>.json`) for previous strategy decisions and outcomes.
2. Read previous session logs (`monitoring-logs/*/8-strategy.md`) for this job.
3. **If per-job state contains previous strategy decisions** OR previous session logs contain strategy outcomes: ask the user via AskUserQuestion: "What approaches have you already tried for this training job, and what were the results?" Include what you found in logs so the user can correct or supplement.
   **If no previous strategy history exists** (first session, or job has never triggered strategist before): skip this question. Logs are the primary source; there is nothing for the user to confirm.

**If user provides vague or minimal history** ("I don't remember", "a few things"): proceed with per-job state logs as the primary source. Note "History: incomplete -- user could not recall prior attempts, relying on per-job state logs only."

4. Compile a list of previously attempted approaches and their outcomes.

**First session:** If no per-job state exists and user reports no prior attempts, state "First monitoring session, no prior history." This is a valid state, not an error. Proceed to step 2.

### 2. Evaluate Previous Hypothesis

If per-job state contains a pending hypothesis from a previous session:

1. Read the recorded hypothesis, success criterion, failure criterion, and `evaluate_after` timestamp.
2. If current time < `evaluate_after`: note "Previous hypothesis has not had enough time to evaluate (budget: [X], elapsed: [Y])." Carry it forward as still pending. Skip to step 3.
3. Compare current monitoring data against the recorded success and failure criteria.
4. Classify the outcome:
   - **CONFIRMED**: success criterion met. Record what was learned. This biases the Decision Gate toward CONTINUE (if metrics are now on track) or REFINE (if still suboptimal despite improvement).
   - **REFUTED**: failure criterion met, or success criterion clearly not met. Record what was learned. Promote the counter-hypothesis as a candidate for step 5 (it pre-fills one of the 3 hypothesis slots; generate 2 new ones).
   - **INCONCLUSIVE**: neither criterion clearly met. Note what additional evidence would resolve it. Carry the hypothesis forward as still pending.
5. Update per-job state with the outcome, evidence, and what was learned.

If no pending hypothesis exists, skip this step.

### 3. Articulate What Changed Our Belief

Before proposing any action, state explicitly what new information the current monitoring cycle revealed:

- What did we expect? (from predictions or previous session)
- What did we observe? (from monitoring report)
- What does the delta tell us? (information gain)
- What beliefs should we update based on this?

If the monitoring cycle revealed nothing new, state that explicitly. Do not fabricate information gain.

**First session:** Use the monitoring session's own Step 1 predictions as the prior expectation. These predictions were written before evidence was collected, so they serve as the baseline for comparison. The delta is: predictions vs actual monitoring observations.

### 4. Decision Gate

Classify the situation into exactly one category before generating hypotheses. The classification determines the scope and urgency of your proposals:

| Category | Criteria | Implication |
|----------|----------|-------------|
| CONTINUE | Training is on track, metrics progressing as expected | No strategic changes needed. 0-3 optional efficiency suggestions (lightweight format). Low urgency. |
| REFINE | Some progress visible, but suboptimal or slowing | Propose parameter adjustments. Medium urgency. |
| PIVOT | Fundamentally wrong direction -- metrics flat or diverging after substantial compute | Propose architectural or methodological changes. High urgency. |
| INVESTIGATE | Insufficient information to classify | Propose diagnostic experiments to gather missing data. |

State your classification and the evidence that supports it.

**First session:** With only one snapshot, classify using prediction vs actual delta (not trend data). If predictions matched reality within expected variance, classify as CONTINUE. If significant deviation, classify accordingly. Multi-snapshot trend is not required for the first session.

### 5. Generate 3 Hypotheses

**Conditional on Decision Gate:**
- CONTINUE: generate 0-3 optional suggestions. Suggestions use a lightweight format (one sentence each: what to try and why). The full hypothesis structure is not required. It is valid to produce zero suggestions ("No strategic changes needed, continue monitoring.").
- REFINE / PIVOT / INVESTIGATE: generate exactly 3 full hypotheses with all required fields.

Generate hypotheses on DIFFERENT dimensions. Do not generate 3 variations of the same idea. Each hypothesis must be a structured object:

For each hypothesis:

```
Hypothesis [N]: [one-sentence falsifiable claim]
  Dimension: [what aspect of training this addresses -- e.g., learning rate, data quality, architecture, regularization]
  Action: [minimum experiment to test this -- the cheapest thing that would confirm or refute]
  Expected outcome: [what you expect to see if the hypothesis is correct, with specific metrics]
  Failure criterion: [what would indicate this hypothesis is wrong]
  Counter-hypothesis: [if this is wrong, the alternative explanation is...]
  Cost: [GPU-hours, human-hours, risk of wasted compute]
  Learning value: [what we learn regardless of outcome -- information gain per effort]
```

**Constraints:**
- Each hypothesis must address a different dimension (e.g., one about learning rate, one about data, one about architecture -- not three learning rate variants)
- Never repeat a previously failed approach unless you have new evidence justifying why the outcome would differ
- Rank by learning-to-effort ratio: "What is the cheapest experiment that would tell us the most?"

**Dimension independence test:** Two hypotheses are on different dimensions if and only if: confirming or refuting hypothesis A provides NO information about whether hypothesis B is true or false. Example: testing "increase LR" tells you nothing about "data quality is poor" -- different dimensions. Testing "increase LR" gives you information about "switch optimizer" -- same dimension (both address optimization hyperparameters). If two hypotheses fail this test, replace one.

**Rankings are qualitative by design.** The learning-to-effort ratio is a reasoned judgment, not a computed score. Quantitative scoring produces false precision -- the numbers would look rigorous but reflect no real measurement.

**Counter-hypothesis quality:** A counter-hypothesis must name a different causal mechanism, not just negate the primary. "The learning rate is NOT too low" is not valid. "The reward function is saturated" is valid.

**Promoted counter-hypothesis:** If step 2 classified a previous hypothesis as REFUTED, its counter-hypothesis automatically fills one of the 3 slots. Generate only 2 new hypotheses. The promoted counter-hypothesis must still pass the information independence test against the new ones.

### 6. Present Options to User

Present the 3 hypotheses to the user via AskUserQuestion. Format:

```
Based on the current monitoring data, I classified the situation as [CATEGORY] because [reason].

What changed our belief: [summary from step 3]

Option 1: [hypothesis 1 summary]
  - Test: [action]
  - If wrong: [counter-hypothesis]
  - Cost: [estimate]
  - We learn: [learning value]

Option 2: [hypothesis 2 summary]
  - Test: [action]
  - If wrong: [counter-hypothesis]
  - Cost: [estimate]
  - We learn: [learning value]

Option 3: [hypothesis 3 summary]
  - Test: [action]
  - If wrong: [counter-hypothesis]
  - Cost: [estimate]
  - We learn: [learning value]

Which option would you like to pursue? (or describe a different direction)
```

**If user says "none of these" or "you decide":** recommend the hypothesis with the highest learning-to-effort ratio and explain why.
**If user provides a custom direction:** generate a new hypothesis using the same structured format (all fields), then proceed to step 7 with the custom hypothesis.

### 7. Generate Execution Plan

After the user chooses an option (or provides a custom direction):

1. Write a concrete execution plan with:
   - Specific commands or config changes
   - Success criterion (what metric value or behavior confirms the hypothesis)
   - Failure criterion (what would refute it)
   - Checkpoint/rollback plan (how to revert if it fails)
   - Time budget (how long to run before evaluating)
2. Classify each action:
   - SAFE: pure read-only operations (reading logs, checking metrics, listing files, inspecting configs). Nothing else.
   - DANGEROUS: everything that writes, modifies, launches, kills, or consumes resources. This includes: changing config files, restarting training, killing processes, launching diagnostic runs, downloading data.
   - **Default is DANGEROUS.** When in doubt, classify as DANGEROUS. False positives (unnecessary confirmation) are cheap. False negatives (auto-executing something destructive) can waste GPU hours or destroy training state.
3. For SAFE actions: execute directly
4. For DANGEROUS actions: present to user via AskUserQuestion for explicit confirmation before execution
5. Explicitly tell the user: "Please run monitoring again after [time budget] to evaluate whether this hypothesis was confirmed or refuted."

### 8. Record Decision

Record the strategy decision in the gate log for future reference:
- Which hypothesis was chosen and why
- The execution plan
- Success/failure criteria
- Timestamp
- `evaluate_after`: timestamp (current time + time budget) -- used by next session to check if enough time has passed

**If a troubleshooter report exists for this session:** do not re-diagnose the same technical failure. Your hypotheses should address the strategic direction AFTER the immediate fix -- not the fix itself. Reference the troubleshooter's findings as established context.

## Gate Log Format

Record in `monitoring-logs/<timestamp>/8-strategy.md`. This gate log is a **decision document**, not an evidence document. Each section records a stage of the strategic decision process.

### Input

- Monitoring report summary (status, key indicator, deviations)
- Troubleshooter report (if Step 7 was triggered)
- User history response (what was tried before, or "first session / no history")
- Previous hypothesis evaluation (if pending hypothesis existed: CONFIRMED/REFUTED/INCONCLUSIVE with evidence)

### Execution

- Decision Gate classification with evidence
- "What Changed Our Belief" statement
- 3 hypotheses (or 0-3 suggestions for CONTINUE) with all structured fields
- User's choice and reasoning (or "CONTINUE: no user interaction needed")

### Output

- Chosen hypothesis (or "no changes")
- Execution plan with specific commands/config changes
- Success/failure criteria
- Time budget and evaluate_after timestamp

### Next

- What to monitor in the next session to evaluate whether the chosen hypothesis was confirmed or refuted
- Any carry-forward items (inconclusive hypotheses, unresolved questions)
