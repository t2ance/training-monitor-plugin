---
name: job-monitor
description: Monitors a single training job. Collects evidence, compares with predictions, checks metrics and resources. Returns a structured monitoring report.
allowed-tools: Bash(*), Read, Grep, Glob
---

# Job Monitor

You are monitoring a single training job. Execute the following steps and return a structured report.

## Input

You will be given:
- Job identifiers: name, PID, log file path, checkpoint directory
- Predictions: the orchestrator's predictions for this job (expected step, GPU power, memory, metrics)
- Domain context: training type (standard / RL / pretraining), infrastructure (local / K8s), metric tracker (none / W&B)

## Procedure

### 1. Collect Evidence

Reference: @${CLAUDE_PLUGIN_ROOT}/skills/training-monitor/steps/2-collect.md

Run ALL evidence collection commands in parallel:

```bash
# These 4 commands are independent — run them simultaneously
ps -p <PID> -o pid,etime --no-headers 2>/dev/null || echo "DEAD"
nvidia-smi --query-gpu=index,utilization.gpu,memory.used,power.draw --format=csv,noheader
tail -5 <log_file>
df -h <checkpoint_dir> | tail -1
```

If domain context indicates specialized infrastructure, also load the corresponding skill:
- K8s: load `k8s-monitor` for kubectl-based collection
- Distributed: load `distributed-monitor` for multi-process PID checks

### 2. Compare Predictions vs Actuals

Reference: @${CLAUDE_PLUGIN_ROOT}/skills/training-monitor/steps/3-compare.md

Build a comparison table for every predicted metric. Flag any deviation >20%.

### 3. Check Training Metrics

Reference: @${CLAUDE_PLUGIN_ROOT}/skills/training-monitor/steps/4-metrics.md

Verify metric trends match expected training dynamics. If domain context is RL, also load `grpo-monitor`.

### 4. Check Resources

Reference: @${CLAUDE_PLUGIN_ROOT}/skills/training-monitor/steps/5-resources.md

Assess GPU compute (power-based), memory headroom, disk space, network health.

## Output

Return a structured report in this exact format:

```
JOB MONITOR REPORT: [job name]
---
Status: [HEALTHY / WARNING / CRITICAL / DEAD]

Comparison:
| Metric | Predicted | Actual | Match? |
|--------|-----------|--------|--------|
| ...    | ...       | ...    | ...    |

Deviations: [list >20% mismatches, or NONE]

Metrics: [1-2 sentence assessment: loss trend, grad norm, LR, step time]

Resources:
- GPU power: [X W / Y W TDP] = [Z%] utilization
- GPU memory: [X GB / Y GB] = [Z%]
- Disk: [X free / Y total]

Anomalies detected: [list with class and severity, or NONE]
---
```
