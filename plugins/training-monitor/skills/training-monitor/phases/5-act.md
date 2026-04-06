# Phase 5: Act

Two sub-phases: troubleshoot (conditional) then strategize (always).

---

## 5a. Troubleshoot (Conditional)

**Skip if**: status is HEALTHY, or WARNING with no specific anomalies.
**Trigger if**: status is CRITICAL, OR specific anomalies or deviations were found in Phases 2-3.

Dispatch a troubleshooter sub-agent for systematic investigation.

### Procedure

#### 1. Observe
State exactly what is wrong with numbers.
- Bad: "training seems slow"
- Good: "step 5 took 120min vs average 61min for steps 1-4"

#### 2. Reproduce
Is this consistent or intermittent? Re-run the failing check. Scan earlier logs for the same pattern.

#### 3. Isolate
Narrow down root cause category (check in order):
1. **Hardware**: GPU power, temperature, memory errors
2. **Data**: current batch corrupted, abnormally long, or all padding?
3. **Config**: did LR, batch size, or any hyperparameter change at this step?
4. **Software**: Python process state, OOM killer, disk full, CUDA errors
5. **Network**: if distributed, check inter-GPU communication

#### 4. Root Cause
Identify root cause(s). If multiple independent issues, report each separately. Do not force-fit independent problems into a single narrative.

#### 5. Recommended Action
Propose a specific action for each root cause: a concrete command/config change OR a diagnostic step. "Investigate further" is not acceptable.

#### 6. Risk Assessment
For each action: what could go wrong? Is it reversible?

### Anomaly Classes

Consult domain skills for domain-specific anomaly classes. General classes:

| Class | Detect | Action |
|-------|--------|--------|
| Silent Stall | Step count unchanged >2 checks, log timestamp stale >2x step time | Check process state, GPU power. If stall confirmed, kill and restart. |
| Metric Anomaly | Loss=0.0, NaN/Inf, grad_norm exploded >100x or dropped to ~0 | Verify real vs logging artifact. Check batch data, model weights, optimizer state. |
| Resource Anomaly | GPU memory >95%, utilization <50% sustained, CPU memory growing | GPU >95%: stop before OOM. Low util: check CPU/disk/network bottleneck. |
| Step Time Regression | Step N >1.5x average of previous steps | Check variable sequence lengths, checkpoint I/O, disk throughput. |
| Checkpoint/Storage | Disk >80%, checkpoint save failed, zero-byte files | Delete old checkpoints, verify atomic writes. |
| Train-Eval Divergence | Train improving, eval flat/declining for 3+ evals | Overfitting. Use checkpoint from before divergence. |

### Escalation Protocol

If confidence remains LOW after investigation: record what was tried, why insufficient, and the ONE diagnostic command that would most narrow down the cause.

---

## 5b. Strategize (Always)

Propose strategic next steps. You are asking "what should we do next?" -- not "what happened."

### 1. Gather History

Read per-job state and previous strategy logs. If previous strategy decisions exist, ask the user what they've tried. If no history exists (first session), skip the user question.

### 2. Evaluate Previous Hypothesis

If per-job state contains a pending hypothesis:
- If current time < `evaluate_after`: carry forward as pending.
- Otherwise: compare monitoring data against success/failure criteria.
- Classify: CONFIRMED / REFUTED / INCONCLUSIVE.
- If REFUTED: promote counter-hypothesis as a candidate.

### 3. Articulate What Changed Our Belief

State explicitly: what did we expect? what did we observe? what does the delta tell us?

### 4. Decision Gate

| Category | Criteria | Implication |
|----------|----------|-------------|
| CONTINUE | On track, metrics progressing | 0-3 optional suggestions (lightweight). |
| REFINE | Some progress, but suboptimal | Parameter adjustments. Medium urgency. |
| PIVOT | Wrong direction, metrics flat/diverging | Architectural changes. High urgency. |
| INVESTIGATE | Insufficient information | Diagnostic experiments. |

### 5. Generate Hypotheses

- CONTINUE: 0-3 optional suggestions (one sentence each).
- REFINE / PIVOT / INVESTIGATE: exactly 3 full hypotheses on DIFFERENT dimensions.

Each hypothesis:
```
Hypothesis [N]: [one-sentence falsifiable claim]
  Dimension: [what aspect]
  Action: [minimum experiment to test]
  Expected outcome: [specific metrics if correct]
  Failure criterion: [what indicates wrong]
  Counter-hypothesis: [alternative causal mechanism, not just negation]
  Cost: [GPU-hours, human-hours, risk]
  Learning value: [information gain per effort]
```

Constraints:
- Different dimensions (confirm/refute A tells nothing about B).
- Never repeat previously failed approach without new evidence.
- Rank by learning-to-effort ratio.

### 6. Present to User

Present via AskUserQuestion. Include classification, belief change, and 3 options with test/counter-hypothesis/cost/learning.

### 7. Execution Plan

After user chooses:
- Specific commands or config changes
- Success/failure criteria
- Checkpoint/rollback plan
- Time budget
- Classify actions: SAFE (read-only) or DANGEROUS (everything else, default DANGEROUS)
- SAFE: execute directly. DANGEROUS: confirm with user first.

---

## Gate Log Format

Record in `monitoring-logs/<timestamp>/5-act.md`:

### Troubleshoot (if triggered)
- Observation, root cause, recommended action, risk, confidence

### Strategy
- Decision Gate classification with evidence
- "What Changed Our Belief" statement
- Hypotheses (or suggestions for CONTINUE)
- User's choice
- Execution plan with success/failure criteria
- `evaluate_after` timestamp
