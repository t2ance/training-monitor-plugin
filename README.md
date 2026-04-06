# Training Monitor

Prediction-first autonomous monitoring for ML/DL training jobs. A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) that monitors GPU training with derived judgment criteria, adversarial review, and cross-session learning.

## Install

```bash
claude plugin marketplace add t2ance/training-monitor-plugin
claude plugin install training-monitor@training-monitor
```

Or add to `~/.claude/settings.json`:

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

## Update

```bash
claude plugin update training-monitor
```

Restart Claude Code to activate the new version. Or use `/reload-plugins` to hot-reload without restarting.

## Setup

After installation, run the interactive setup wizard:

```
/monitor-doctor
```

It detects your training environment, checks dependencies (nvidia-smi, kubectl, wandb), installs what is missing, and lets you choose a monitoring mode (team or ralph).

## Quick Start

```
/training-monitor        # One-shot monitoring pass
/monitor-team            # Periodic monitoring via teammates
/monitor-ralph           # Periodic monitoring via auto-compact loop
```

## Documentation

See [plugins/training-monitor/](plugins/training-monitor/) for the full plugin documentation: architecture, skills, phases, and domain extensions.
