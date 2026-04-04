# Step 6: Handle Anomalies

If ANY anomaly is detected, act NOW. Do not defer. For domain-specific anomaly classes, load the corresponding skill (`grpo-monitor`, `distributed-monitor`, `k8s-monitor`).

## Class A: Silent Stalls

**Detect:** step count unchanged across 2+ checks, OR log timestamp stale >2x expected step time, OR external metric tracker not updated while process is "alive".

**Action:**
1. Check process state: `cat /proc/<PID>/status | grep State` — running, sleeping, or zombie?
2. Check GPU power: idle power + "alive" process = deadlock.
3. If stall confirmed, kill and restart. If uncertain, recheck in 10 min, then kill.

For distributed training stall detection (NCCL hangs, multi-process PID checks), load the `distributed-monitor` skill.

## Class B: Metric Anomalies

**Detect:** loss exactly 0.0 for multiple steps, NaN/Inf anywhere, grad_norm exploded >100x baseline, grad_norm dropped to ~0 (vanishing).

For RL-specific metric anomalies (reward, entropy, KL), load the `grpo-monitor` skill.

**Action:**
1. Verify the anomaly is real, not a logging artifact. Compare log file values to external tracker values (if available).
2. If real, invoke `superpowers:systematic-debugging`:
   - Identify exact step where anomaly started
   - Check the batch data at that step (corrupted? all padding? wrong format?)
   - Check model weights (all zeros? NaN in any layer?)
   - Check optimizer state (LR correct? momentum accumulated wrong?)
3. For complex diagnostic decisions, use `team-decision-review` to challenge your hypothesis.

## Class C: Resource Anomalies

**Detect:** GPU memory >95% (OOM risk), utilization <50% sustained, power at idle levels during "training", CPU memory growing unboundedly.

**Action:**
1. GPU memory >95%: PyTorch `expandable_segments` can cause memory to grow over steps. Monitor for 5 min — if still growing, stop before OOM corrupts the checkpoint. Better to lose 1 step than the whole run.
2. GPU utilization low: check for CPU bottleneck (data loading), disk bottleneck (checkpoint I/O), network bottleneck, or Python GIL contention.
3. Power at idle: treat as Class A stall.

## Class E: Step Time Regression

**Detect:** step N takes >1.5x the average of previous steps. OR checkpoint save time >2x previous. OR single sub-phase takes >2x previous.

**Root causes (check in order):**
1. Variable sequence lengths in batch — check `prompt_length/max` and `response_length/max` for the slow step.
2. Checkpoint save competing with training — if save_freq=1, every step writes to disk.
3. Disk throughput degraded — storage system congestion, NFS slowdown, or local disk full.
4. CUDA graph recompilation — new sequence length bucket triggers recompile.
5. Other process competing for GPU — check nvidia-smi for unexpected processes.

**Action:** If step time >2x average and growing, this is often a precursor to OOM. Monitor memory trend. If memory also increasing, stop and reduce max_response_length or batch_size.

## Class F: Checkpoint & Storage

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

## Class H: Train-Eval Divergence

**Detect:** train metric improving but eval metric flat or declining for 3+ consecutive evaluations.

This is overfitting. The model is memorizing training data, not generalizing.

**Action:**
1. Confirm with numbers: plot train vs eval over the last N steps.
2. Check if the eval set is representative (not too small, not from same distribution as train).
3. Options (use `team-decision-review` to decide):
   - Stop training and use checkpoint from before divergence started.
   - Add regularization (dropout, weight decay).
   - Reduce learning rate.
   - For RL-specific actions, load the `grpo-monitor` skill.

## Escalation Protocol

When an anomaly's root cause is not immediately clear:

1. Invoke `superpowers:systematic-debugging`:
   - **Observe**: state exactly what is wrong with numbers ("step 5 took 120min vs avg 61min", not "it seems slow")
   - **Reproduce**: consistent or intermittent?
   - **Isolate**: data? model? hardware? config? network?
   - **Root cause**: one fundamental cause, not symptoms
   - **Fix**: minimal change
   - **Verify**: confirm fix worked

2. For judgment calls (stop vs continue, change config vs wait), invoke `team-decision-review` to challenge your reasoning before acting.

3. Spawn sub-agents for parallel investigation when multiple hypotheses need checking simultaneously.
