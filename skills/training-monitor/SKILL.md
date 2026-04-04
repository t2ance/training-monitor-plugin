---
name: training-monitor
description: Prediction-first autonomous monitoring for ML/DL training jobs. Write expectations BEFORE reading logs, compare, detect anomalies, fix issues on the spot. General-purpose for any single-GPU PyTorch training pipeline.
---

# Training Monitor (Prediction-First)

Autonomous monitoring for ML/DL training jobs. Core principle: **write predictions before reading logs**, like writing tests before code. This prevents confirmation bias.

Baseline assumption: single-GPU, single-process PyTorch training with log file output. For specialized environments, load the corresponding domain skill.

## Procedure

### Step 1: Write Predictions

> Detail: [steps/1-predict.md](steps/1-predict.md)

Based on previous checks and training config, write predictions for EVERY running job **BEFORE making any tool calls**. Include expected step, GPU power, memory, phase, and key metrics.

You MUST write these predictions as text output before reading any data.

### Step 2: Collect Evidence

> Detail: [steps/2-collect.md](steps/2-collect.md)

Run parallel checks: process status, GPU metrics, log tail, disk space. Collect ALL data before analyzing.

### Step 3: Compare & Validate

> Detail: [steps/3-compare.md](steps/3-compare.md)

Build prediction-vs-actual table for each job. Cross-validate across data sources. Any >20% deviation = anomaly to investigate.

### Step 4: Check Training Metrics

> Detail: [steps/4-metrics.md](steps/4-metrics.md)

Verify metric trends match expected training dynamics. Check loss, gradient norm, LR schedule, step time consistency, eval gap.

### Step 5: Check Resources

> Detail: [steps/5-resources.md](steps/5-resources.md)

Assess GPU compute utilization (power-based, not nvidia-smi %), memory headroom, disk for remaining checkpoints, CPU memory trend, network health.

### Step 6: Handle Anomalies

> Detail: [steps/6-anomalies.md](steps/6-anomalies.md)

If ANY anomaly detected, classify and act immediately. Do not defer.

### Step 7: Log Findings

> Detail: [steps/7-log.md](steps/7-log.md)

Append structured report to the project's autonomous log file.

## Domain Skills

For specialized training types or infrastructure, load the corresponding skill alongside this core flow:

| Skill | When to load |
|-------|-------------|
| `grpo-monitor` | Training uses GRPO, PPO, or other RL algorithms |
| `distributed-monitor` | Training uses multiple GPUs or multiple processes |
| `k8s-monitor` | Job runs on Kubernetes |

For external metric trackers (W&B, TensorBoard, MLflow), check if a corresponding third-party plugin is installed. Use `monitor-doctor` to verify dependencies.

## Rules

- NEVER read logs before writing predictions.
- If ANY prediction wrong by >20%, treat as anomaly — investigate root cause.
- If a job is DEAD, do not restart without understanding why. Use `superpowers:systematic-debugging`.
- Report ALL GPUs, not just the ones you expect busy. Unexpected GPU usage = someone else's process or zombie.
- If anomaly detected, resolve NOW. Do not defer.
- Never guess cause of anomaly. Investigate with evidence.
- Trust hardware metrics (nvidia-smi) over software metrics (log file) over external dashboards.
- You are fully autonomous. Spawn sub-agents, teams, or any combination to diagnose and fix.
