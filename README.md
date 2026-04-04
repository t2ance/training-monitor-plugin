# Training Monitor Plugin

Prediction-first autonomous monitoring for ML/DL training jobs. Write expectations before reading logs, dispatch sub-agents for parallel monitoring, detect anomalies and investigate root causes.

## Installation

### Via CLI (recommended)

```bash
# Add the marketplace and install
claude plugin marketplace add t2ance/training-monitor-plugin
claude plugin install training-monitor@training-monitor
```

### Via settings.json

Add these to `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "training-monitor": {
      "source": {
        "source": "github",
        "repo": "t2ance/training-monitor-plugin"
      }
    }
  },
  "enabledPlugins": {
    "training-monitor@training-monitor": true
  }
}
```

Restart Claude Code to pull the plugin.

### Setup

After installation, run the interactive setup wizard:

```
/monitor-doctor
```

It will detect your training environment, check dependencies, and offer to install what is missing.

## Skills

| Skill | Description | Standalone? |
|-------|-------------|-------------|
| `training-monitor` | Core monitoring orchestrator. Dispatches sub-agents for parallel job monitoring and anomaly investigation. | Yes |
| `grpo-monitor` | GRPO/RL training: reward, KL, entropy, generation quality, phase time balance. | Yes |
| `distributed-monitor` | Multi-GPU/multi-process: NCCL, process hierarchy, straggler detection. | Yes |
| `k8s-monitor` | Kubernetes: kubectl collection, pod anomalies, scheduling escalation ladder. | Yes |
| `wandb-monitor` | W&B monitoring: heartbeat stall detection, metric key variations, health thresholds, run comparison. | Yes |
| `monitor-doctor` | Interactive setup wizard. Detects environment, checks dependencies, installs missing ones. | Yes |

## Agents

| Agent | Dispatched by | Purpose |
|-------|--------------|---------|
| `job-monitor` | `training-monitor` | Monitors a single job (collect, compare, metrics, resources). Dispatched in parallel for multi-job. |
| `anomaly-investigator` | `training-monitor` | Systematic root cause analysis for a detected anomaly. One per anomaly, in parallel. |

## External Dependencies

The plugin works out of the box for single-GPU PyTorch training with log file output. Additional features require external dependencies:

| Feature | Requires | Install |
|---------|----------|---------|
| W&B monitoring | `wandb-primary` skill + `wandb` package | `npx skills add wandb/skills` then `pip install wandb && wandb login` |
| K8s monitoring | `kubectl` with cluster access | See [Kubernetes docs](https://kubernetes.io/docs/tasks/tools/) |
| GPU monitoring (baseline) | `nvidia-smi` | Pre-installed on GPU machines |

Run `/monitor-doctor` to check what is installed and install what is missing interactively.

## Usage

### Monitor running training jobs

```
/training-monitor
```

The orchestrator will:
1. Write predictions before reading any data
2. Dispatch sub-agents for parallel monitoring (multi-job)
3. Synthesize results and detect anomalies
4. Dispatch investigation agents for any anomalies
5. Log findings

### Use domain skills standalone

```
/grpo-monitor          # GRPO/RL monitoring knowledge
/distributed-monitor   # Multi-GPU monitoring knowledge
/k8s-monitor           # Kubernetes monitoring knowledge
/wandb-monitor         # W&B monitoring knowledge
```

## Architecture

```
training-monitor (orchestrator)
├── Phase 1: Predict         [serial]
├── Phase 2: Monitor          [parallel: job-monitor agent per job]
├── Phase 3: Synthesize       [serial]
├── Phase 4: Investigate      [parallel: anomaly-investigator per anomaly]
└── Phase 5: Log              [serial]
```

Baseline: single-GPU, single-process PyTorch. Domain skills extend to GRPO, distributed, K8s, W&B.
