---
name: monitor-doctor
description: Check that all external dependencies for the training-monitor plugin are available. Verifies third-party plugins, CLI tools, and Python packages.
---

# Monitor Doctor

Run this skill to verify that external dependencies for the training-monitor plugin are in place.

## Procedure

### 1. Check Built-in Skills

Verify the following skills from this plugin are loadable:
- `training-monitor` (core flow)
- `grpo-monitor` (GRPO/RL domain)
- `distributed-monitor` (multi-GPU domain)
- `k8s-monitor` (Kubernetes domain)

### 2. Check External Plugin Dependencies

These are optional. Only needed if the user's training job uses the corresponding tool.

| External Plugin | Needed when | How to check |
|----------------|-------------|-------------|
| W&B monitoring plugin | Job logs to Weights & Biases | Check if a `wandb` or `wandb-monitor` skill is available. If not, W&B-specific monitoring (heartbeat stall detection, metric key variations, run comparison scripts) will not be available. |

### 3. Check CLI Tools

```bash
# Always needed
nvidia-smi --version 2>/dev/null && echo "OK: nvidia-smi" || echo "MISSING: nvidia-smi"

# Only for K8s
kubectl version --client 2>/dev/null && echo "OK: kubectl" || echo "MISSING: kubectl (needed for k8s-monitor)"

# Only for W&B
python -c "import wandb" 2>/dev/null && echo "OK: wandb" || echo "MISSING: wandb Python package (needed for W&B monitoring)"
```

### 4. Report

Print a summary:

```
Training Monitor Doctor
-----------------------
Built-in skills:    OK
External plugins:   [list installed / missing]
CLI tools:          [list available / missing]
-----------------------
Ready to monitor:   [YES / YES with limitations / NO]
```

If any dependency is missing, provide the install command or plugin name.
