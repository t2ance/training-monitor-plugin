---
name: wandb-monitor
description: Monitoring guidance for training jobs that log to Weights & Biases. Covers heartbeat stall detection, metric key variations, cross-source validation, health thresholds, and run comparison. Depends on wandb-primary skill (wandb/skills) for API access. Use standalone or alongside training-monitor.
---

# W&B Training Monitor

Monitoring guidance for training jobs that log to Weights & Biases. This skill provides the **monitoring logic** (what to check, what thresholds mean, how to interpret results). For W&B API patterns (how to call the SDK), it depends on the `wandb-primary` skill from `wandb/skills`.

**Dependency**: `wandb-primary` skill must be installed (`npx skills add wandb/skills`) and `wandb` Python package must be available with valid authentication.

## W&B Evidence Collection

### Heartbeat Sync Check

```bash
# Quick local check: is W&B syncing?
stat --format='%Y' <wandb_dir>/latest-run/*.wandb && date +%s
```

If the wandb file's mtime is >10min behind current time, W&B sync may be stalled.

### Python API Collection

```python
import wandb
api = wandb.Api()

# Get run by ID
run = api.run("entity/project/run_id")

# Key properties for monitoring
run.state        # running | finished | failed | crashed | canceled
run.heartbeat_at # last heartbeat timestamp — primary stall indicator
run.summary      # final/current metrics
run.config       # hyperparameters

# Get metric history (use wandb-primary patterns for efficient access)
history = list(run.scan_history(keys=["train/loss", "train/grad_norm"], page_size=1000))
```

### Run Overview

```python
# List all running jobs for an entity
runs = api.runs("entity/project", filters={"state": "running"})
for run in runs:
    print(f"{run.name}: step {run.summary.get('_step', '?')}, heartbeat {run.heartbeat_at}")
```

## Metric Key Variations

Different training frameworks log to W&B with different key names. When collecting metrics, check all common variants:

| Metric | Common Key Variants |
|--------|-------------------|
| Loss | `train/loss`, `loss`, `train_loss`, `training_loss` |
| Gradients | `train/grad_norm`, `grad_norm`, `gradient_norm` |
| Steps | `train/global_step`, `global_step`, `step`, `_step` |
| Eval | `eval/loss`, `eval_loss`, `eval/accuracy`, `eval_acc` |
| Learning rate | `train/lr`, `lr`, `learning_rate` |

When a metric key is not found, try all variants before concluding the metric is not logged.

## Health Thresholds

| Condition | Severity | Meaning |
|-----------|----------|---------|
| Gradients > 10 | Critical | Exploding gradients |
| Gradients > 5 | Warning | Spiky gradients |
| Gradients < 0.0001 | Warning | Vanishing gradients |
| Heartbeat > 30min stale | Critical | Job likely stalled or crashed |
| Heartbeat > 10min stale | Warning | Possibly stuck, investigate |
| Run state = `crashed` | Critical | Job crashed, check logs |
| Run state = `failed` | Critical | Job failed, check exit code |

## Heartbeat-Based Stall Detection

W&B heartbeat is more reliable than log file timestamps for detecting stalls, because it updates independently of training step logging.

- `run.heartbeat_at` gives last heartbeat timestamp.
- Compare against current time to get staleness.
- Heartbeat > 10min stale while process appears alive via `ps`: investigate.
- Heartbeat > 30min stale: treat as confirmed stall regardless of process status.

### False Positives

Heartbeat staleness does NOT always mean the job is stuck:
- **Checkpoint save**: large checkpoints can block the main thread for minutes. Check if disk I/O is active.
- **W&B offline mode**: if the network is down, heartbeat stops updating but training continues. Check log file timestamps as a backup.
- **Rate limiting**: W&B API may be throttled. Check `wandb/latest-run/logs/` for sync errors.

## W&B Cross-Source Validation

| Source A | Source B | If they disagree |
|----------|----------|------------------|
| Log file step count | W&B step count | Logging bug or W&B sync delay |
| Step N log metrics | W&B step N metrics | Log parsing error or W&B batching |
| Process alive (ps) | W&B heartbeat stale | W&B sync issue or true stall — check GPU power to disambiguate |
| W&B `run.state` | Process status | State update delay — W&B state lags by up to 60s |

## Run Comparison

When investigating whether a training run is behaving normally, compare against previous runs:

```python
# Get finished runs for comparison baseline
baseline_runs = api.runs("entity/project", filters={"state": "finished"}, order="-created_at")

# Compare loss at same step
current_loss = current_run.summary.get("train/loss")
baseline_losses = [r.summary.get("train/loss") for r in baseline_runs[:3]]
```

Key comparison points:
- Loss at same step count: is current run within 2x of baseline?
- Step time: is current run within 1.5x of baseline?
- GPU memory: is current run using significantly more than baseline? (potential memory leak)
