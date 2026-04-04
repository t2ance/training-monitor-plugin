---
name: monitor-doctor
description: Check and set up external dependencies for the training-monitor plugin. Verifies third-party skills, CLI tools, and Python packages. Reports what is available and provides install commands for what is missing.
---

# Monitor Doctor

Run this skill to verify that external dependencies for the training-monitor plugin are in place. Reports available features and provides install commands for missing dependencies.

## Procedure

### 1. Check Built-in Skills

Verify the following skills from this plugin are loadable:
- `training-monitor` (core orchestrator)
- `grpo-monitor` (GRPO/RL domain)
- `distributed-monitor` (multi-GPU domain)
- `k8s-monitor` (Kubernetes domain)

### 2. Check External Skills

#### W&B Monitoring (wandb/skills)

The official W&B skill provides API patterns for querying runs, metrics, and traces. Our agents use it for W&B-based evidence collection and heartbeat stall detection.

**Check**: Is a `wandb-primary` skill available? Look for it in the skill list.

**If missing**:
```
To install the W&B skill:
  npx skills add wandb/skills

This provides the wandb-primary skill with W&B SDK and Weave SDK patterns
that our monitoring agents use for W&B data access.
```

### 3. Check CLI Tools

```bash
# Baseline (always needed)
nvidia-smi --version 2>/dev/null && echo "OK: nvidia-smi" || echo "MISSING: nvidia-smi — GPU monitoring will not work"

# K8s (only if using k8s-monitor)
kubectl version --client 2>/dev/null && echo "OK: kubectl" || echo "MISSING: kubectl — install from https://kubernetes.io/docs/tasks/tools/"

# W&B (only if using W&B monitoring)
python -c "import wandb; print(f'OK: wandb {wandb.__version__}')" 2>/dev/null || echo "MISSING: wandb — pip install wandb && wandb login"
```

### 4. Check W&B Authentication

If the wandb Python package is installed, verify authentication:

```bash
python -c "import wandb; api = wandb.Api(); print(f'OK: authenticated as {api.viewer.entity}')" 2>/dev/null || echo "WARNING: wandb installed but not authenticated — run: wandb login"
```

### 5. Report

Print a summary:

```
Training Monitor Doctor
=======================

Built-in skills:
  [OK] training-monitor
  [OK] grpo-monitor
  [OK] distributed-monitor
  [OK] k8s-monitor

External skills:
  [OK/MISSING] wandb-primary (wandb/skills)
    Install: npx skills add wandb/skills

CLI tools:
  [OK/MISSING] nvidia-smi
  [OK/MISSING] kubectl
  [OK/MISSING] wandb Python package

Authentication:
  [OK/MISSING] wandb login

=======================
Features available:
  [OK] Core monitoring (single GPU PyTorch)
  [OK/DEGRADED] W&B monitoring — requires wandb-primary skill + wandb package + wandb login
  [OK/DEGRADED] K8s monitoring — requires kubectl
  [OK] GRPO monitoring (knowledge only, no external deps)
  [OK] Distributed monitoring (knowledge only, no external deps)
```

If anything is MISSING or DEGRADED, provide the exact install command.
