# Step 7: Troubleshoot

Investigate anomalies detected in Steps 2-5. This step triggers when status is CRITICAL, or when specific anomalies or deviations were found.

Consult the anomaly class definitions below for known patterns. Domain skills (grpo-monitor, k8s-monitor, etc.) provide heuristics about common failure modes -- use them as reference, not rigid rules.

## Procedure

### 1. Observe

State exactly what is wrong with numbers. No vague descriptions.

- Bad: "training seems slow"
- Good: "step 5 took 120min vs average 61min for steps 1-4"

### 2. Reproduce

Is this consistent or intermittent?

- Re-run the failing check right now
- Scan earlier log entries for the same pattern
- If intermittent, identify the pattern

### 3. Isolate

Narrow down the root cause category. Check in order:

1. **Hardware**: GPU power, temperature, memory errors
2. **Data**: current batch corrupted, abnormally long, or all padding?
3. **Config**: did LR, batch size, or any hyperparameter change at this step?
4. **Software**: Python process state, OOM killer, disk full, CUDA errors
5. **Network**: if distributed, check inter-GPU communication; if remote storage, check I/O

### 4. Root Cause(s)

Identify the root cause(s). If multiple independent issues exist, report each separately with its own evidence and recommended action. Only merge into a single root cause when one issue genuinely causes the others -- do not force-fit independent problems into a single narrative.

### 5. Recommended Action

Propose a specific action for each root cause. This can be:
- A **fix**: "reduce batch size to 4", "delete corrupted checkpoint and resume from previous"
- A **diagnostic step**: "run `nvidia-smi -q | grep 'Retired Pages'` to check for hardware memory errors"

What is NOT acceptable: vague hand-waving like "investigate further" or "monitor the situation." Every recommendation must be a concrete command or config change.

If you cannot determine the root cause (Confidence: LOW), propose the ONE diagnostic command that would most narrow down the cause, and explain what you expect to learn from it.

### 6. Risk Assessment

For each recommended action: what could go wrong? Is it reversible?

## Anomaly Classes

Use these known patterns to guide investigation. For domain-specific anomaly classes, load the corresponding skill (grpo-monitor, distributed-monitor, k8s-monitor).

### Class A: Silent Stalls

**Detect:** step count unchanged across 2+ checks, OR log timestamp stale >2x expected step time, OR external metric tracker not updated while process is "alive".

**Action:**
1. Check process state: `cat /proc/<PID>/status | grep State` -- running, sleeping, or zombie?
2. Check GPU power: idle power + "alive" process = deadlock.
3. If stall confirmed, kill and restart. If uncertain, recheck in 10 min, then kill.

For distributed training stall detection (NCCL hangs, multi-process PID checks), load the `distributed-monitor` skill.

### Class B: Metric Anomalies

**Detect:** loss exactly 0.0 for multiple steps, NaN/Inf anywhere, grad_norm exploded >100x baseline, grad_norm dropped to ~0 (vanishing).

For RL-specific metric anomalies (reward, entropy, KL), load the `grpo-monitor` skill.

**Action:**
1. Verify the anomaly is real, not a logging artifact. Compare log file values to external tracker values (if available).
2. If real, investigate systematically:
   - Identify exact step where anomaly started
   - Check the batch data at that step (corrupted? all padding? wrong format?)
   - Check model weights (all zeros? NaN in any layer?)
   - Check optimizer state (LR correct? momentum accumulated wrong?)

### Class C: Resource Anomalies

**Detect:** GPU memory >95% (OOM risk), utilization <50% sustained, power at idle levels during "training", CPU memory growing unboundedly.

**Action:**
1. GPU memory >95%: PyTorch `expandable_segments` can cause memory to grow over steps. Monitor for 5 min -- if still growing, stop before OOM corrupts the checkpoint. Better to lose 1 step than the whole run.
2. GPU utilization low: check for CPU bottleneck (data loading), disk bottleneck (checkpoint I/O), network bottleneck, or Python GIL contention.
3. Power at idle: treat as Class A stall.

### Class E: Step Time Regression

**Detect:** step N takes >1.5x the average of previous steps. OR checkpoint save time >2x previous. OR single sub-phase takes >2x previous.

**Root causes (check in order):**
1. Variable sequence lengths in batch -- check `prompt_length/max` and `response_length/max` for the slow step.
2. Checkpoint save competing with training -- if save_freq=1, every step writes to disk.
3. Disk throughput degraded -- storage system congestion, NFS slowdown, or local disk full.
4. CUDA graph recompilation -- new sequence length bucket triggers recompile.
5. Other process competing for GPU -- check nvidia-smi for unexpected processes.

**Action:** If step time >2x average and growing, this is often a precursor to OOM. Monitor memory trend. If memory also increasing, stop and reduce max_response_length or batch_size.

### Class F: Checkpoint and Storage

**Detect:** disk usage >80% of capacity, checkpoint save failed (partial write), checkpoint directory missing expected files, checkpoint size suddenly different from previous.

**Checks:**
```bash
df -h <checkpoint_dir>
ls -la <latest_checkpoint>/
du -sh <latest_checkpoint>
find <checkpoint_dir> -name "*.pt" -size 0    # zero-byte files = corruption
```

**Action:**
1. Disk >80%: delete old checkpoints (keep best + latest 2).
2. Corrupted checkpoint: delete it, the model will re-save at next save_freq.
3. Save failed mid-write: check if framework uses atomic writes (temp-file-then-rename). If not, verify checkpoint integrity.

### Class H: Train-Eval Divergence

**Detect:** train metric improving but eval metric flat or declining for 3+ consecutive evaluations.

This is overfitting. The model is memorizing training data, not generalizing.

**Action:**
1. Confirm with numbers: plot train vs eval over the last N steps.
2. Check if the eval set is representative (not too small, not from same distribution as train).
3. Options:
   - Stop training and use checkpoint from before divergence started.
   - Add regularization (dropout, weight decay).
   - Reduce learning rate.
   - For RL-specific actions, load the `grpo-monitor` skill.

## Escalation Protocol

When an anomaly's root cause is not immediately clear:

1. Follow the full procedure above (Observe, Reproduce, Isolate, Root Cause, Action, Risk Assessment).
2. If multiple hypotheses need checking simultaneously, spawn sub-agents for parallel investigation.
3. If confidence remains LOW after investigation, record:
   - What was tried
   - Why it was insufficient
   - The ONE diagnostic command that would most narrow down the cause
   - What you expect to learn from it

## Gate Log Format

Record in `monitoring-logs/<timestamp>/7-troubleshoot.md`:

```
TROUBLESHOOTING REPORT: [description] on [job name]
---
Root cause 1:
  Observation: [what is wrong, with numbers]
  Reproducible: [yes / no / intermittent (pattern)]
  Root cause: [one sentence]
  Evidence: [specific log lines, metrics, command output]
  Recommended action: [concrete command/config change OR diagnostic step]
  Risk: [what could go wrong, is it reversible]

[Root cause 2: (if independent second issue exists)]
  ...

Confidence: [HIGH / MEDIUM / LOW]
If LOW: [what additional data would raise confidence + the ONE diagnostic command to run]
---
```
