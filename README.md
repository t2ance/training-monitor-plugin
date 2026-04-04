# Training Monitor Marketplace

Plugin marketplace for ML/DL training monitoring.

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

## Plugins

| Plugin | Description |
|--------|-------------|
| [training-monitor](plugins/training-monitor/) | Prediction-first autonomous monitoring for ML/DL training jobs |

After installation, run `/monitor-doctor` for interactive setup.
