---
name: monitor-doctor
description: Interactive setup wizard for the training-monitor plugin. Discovers the user's training environment, checks dependencies, offers to install missing ones, and reports available monitoring capabilities.
---

# Monitor Doctor

Interactive setup wizard for the training-monitor plugin. Guides the user through environment discovery, dependency checking, and installation.

## Procedure

### Phase 1: Discover Environment

Use AskUserQuestion to ask these questions in a SINGLE round (all at once):

**Question 1 — Training type** (single select):
- Standard PyTorch (SFT, pretraining, fine-tuning)
- GRPO / RL (PPO, GRPO, RLOO, DPO with online generation)

**Question 2 — Infrastructure** (single select):
- Local single GPU
- Local multi-GPU / distributed
- Kubernetes

**Question 3 — Metric tracker** (multi-select, user may use multiple):
- None (log files only)
- Weights & Biases
- TensorBoard

Store the answers as the user's **environment profile**. This determines which skills and dependencies are needed.

### Phase 2: Determine Requirements

Based on the environment profile, build a requirements table:

| Environment answer | Required skill | External dependency |
|-------------------|---------------|-------------------|
| Any | `training-monitor` (core) | `nvidia-smi` |
| GRPO / RL | `grpo-monitor` | none |
| Local multi-GPU / distributed | `distributed-monitor` | none |
| Kubernetes | `k8s-monitor` | `kubectl` with cluster access |
| Weights & Biases | `wandb-monitor` | `wandb-primary` skill (`npx skills add wandb/skills`) + `wandb` Python package + `wandb login` |

Skills from this plugin are always available (built-in). Only check external dependencies.

### Phase 3: Check and Install

For each external dependency in the requirements table:

1. **Check** if it is already installed (run the check command).
2. **If installed**: mark as OK, show version.
3. **If missing**: collect into a list for the next step.

If there are missing dependencies, use AskUserQuestion with **multi-select**:

**Question — Which dependencies do you want to install?**

List each missing dependency as an option with its install command in the description. For example:
- "wandb Python package" — `pip install wandb`
- "wandb-primary skill" — `npx skills add wandb/skills`
- "wandb authentication" — will run `wandb login` (requires API key)

The user selects which ones to install. For each selected:
- Execute the install command.
- Verify it succeeded.
- If it fails, report the error and move on (do not block other installs).

For each NOT selected:
- Mark the corresponding feature as DEGRADED and explain what the user will miss.

### Phase 4: Capability Report

After all installations complete (or are skipped), present the final capability report:

```
Training Monitor Setup Complete
================================

Environment: [training type] on [infrastructure] with [metric tracker]

Skills loaded:
  [OK] training-monitor (core orchestrator)
  [OK/--] grpo-monitor           (loaded if RL training)
  [OK/--] distributed-monitor    (loaded if multi-GPU)
  [OK/--] k8s-monitor            (loaded if K8s)
  [OK/--] wandb-monitor          (loaded if W&B)

External dependencies:
  [OK/MISSING] nvidia-smi
  [OK/MISSING/--] kubectl
  [OK/MISSING/--] wandb package
  [OK/MISSING/--] wandb-primary skill
  [OK/MISSING/--] wandb authentication

Monitoring capabilities:
  [OK] Prediction-first monitoring flow
  [OK] GPU power and memory monitoring
  [OK] Anomaly detection (Class A-H)
  [OK] Sub-agent parallel job monitoring
  [OK/DEGRADED/OFF] GRPO metric monitoring — [reason if degraded]
  [OK/DEGRADED/OFF] Distributed stall detection — [reason if degraded]
  [OK/DEGRADED/OFF] K8s pod monitoring — [reason if degraded]
  [OK/DEGRADED/OFF] W&B heartbeat monitoring — [reason if degraded]
  [OK/DEGRADED/OFF] W&B cross-source validation — [reason if degraded]

================================
```

If everything is OK: "Ready to monitor. Use `training-monitor` to start."

If anything is DEGRADED: "Some features are limited. You can re-run `monitor-doctor` anytime to install missing dependencies."

### Phase 5: Save Environment Profile (optional)

If the user's project has a `.claude/` directory, offer to save the environment profile so that `training-monitor` can automatically load the right domain skills without asking again:

```json
{
  "training_type": "grpo",
  "infrastructure": "k8s",
  "metric_trackers": ["wandb"],
  "skills_to_load": ["grpo-monitor", "k8s-monitor", "wandb-monitor"]
}
```

Use AskUserQuestion:
- "Save this environment profile for future monitoring sessions?" — Yes / No

If yes, write to `.claude/training-monitor-profile.json` in the project directory.
