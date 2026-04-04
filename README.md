# Training Monitor Plugin

Prediction-first autonomous monitoring for ML/DL training jobs. Write expectations before reading logs, dispatch sub-agents for parallel monitoring, detect anomalies and investigate root causes.

## Quick Start

```bash
# 1. Clone the plugin
git clone https://github.com/t2ance/training-monitor-plugin.git ~/plugins/training-monitor

# 2. Install external dependencies (see below)

# 3. Verify setup
# In Claude Code, run: /monitor-doctor
```

## Skills

| Skill | Description | Standalone? |
|-------|-------------|-------------|
| `training-monitor` | Core monitoring orchestrator. Dispatches sub-agents for parallel job monitoring and anomaly investigation. | Yes |
| `grpo-monitor` | GRPO/RL training: reward, KL, entropy, generation quality, phase time balance. | Yes |
| `distributed-monitor` | Multi-GPU/multi-process: NCCL, process hierarchy, straggler detection. | Yes |
| `k8s-monitor` | Kubernetes: kubectl collection, pod anomalies, scheduling escalation ladder. | Yes |
| `monitor-doctor` | Dependency checker. Verifies external plugins, CLI tools, and Python packages. | Yes |

## Agents

| Agent | Dispatched by | Purpose |
|-------|--------------|---------|
| `job-monitor` | `training-monitor` | Monitors a single job (collect, compare, metrics, resources). Dispatched in parallel for multi-job monitoring. |
| `anomaly-investigator` | `training-monitor` | Systematic root cause analysis for a detected anomaly. One per anomaly, dispatched in parallel. |

## External Dependencies

Some features require third-party skills or tools. The plugin works without them, but with reduced coverage.

### W&B Monitoring (recommended)

If your training logs to Weights & Biases, install the official W&B skill for API access patterns:

```bash
npx skills add wandb/skills
```

This provides `wandb-primary` skill which our agents use to query runs, metrics, and heartbeat status via the W&B API.

You also need the W&B Python package:

```bash
pip install wandb
wandb login
```

### Kubernetes Monitoring

If your jobs run on Kubernetes, you need `kubectl` configured with cluster access:

```bash
kubectl version --client   # verify installed
kubectl get pods           # verify cluster access
```

### GPU Monitoring (baseline)

The baseline assumption is NVIDIA GPU with `nvidia-smi` available:

```bash
nvidia-smi --version       # verify installed
```

## Verify Setup

After installation, run the `monitor-doctor` skill to verify all dependencies:

```
/monitor-doctor
```

It will report which features are available and which dependencies are missing with install instructions.

## Usage

### Monitor running training jobs

```
/training-monitor
```

The orchestrator will:
1. Ask you about running jobs (or detect them)
2. Write predictions before reading any data
3. Dispatch sub-agents for parallel monitoring (multi-job)
4. Synthesize results and detect anomalies
5. Dispatch investigation agents for any anomalies
6. Log findings

### Use domain skills standalone

```
/grpo-monitor          # GRPO/RL monitoring knowledge
/distributed-monitor   # Multi-GPU monitoring knowledge
/k8s-monitor           # Kubernetes monitoring knowledge
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

Baseline: single-GPU, single-process PyTorch. Domain skills extend to GRPO, distributed, K8s. External plugins extend to W&B and other metric trackers.
