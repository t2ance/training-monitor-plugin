# Step 3: Compare & Validate

## Comparison Table

For each prediction, compare against actual data:

```
COMPARISON:
---
[Job Name]
| Metric | Predicted | Actual | Match? |
|--------|-----------|--------|--------|
| Step   | X/Y       | A/B    | YES/NO |
| GPU %  | X%        | A%     | YES/NO |
| Memory | X GB      | A GB   | YES/NO |
| Power  | X W       | A W    | YES/NO |
| Phase  | ...       | ...    | YES/NO |
| Metric | ...       | ...    | YES/NO |

DEVIATIONS: [list any mismatches with >20% discrepancy]
ACTION REQUIRED: [none / investigate X / restart Y]
---
```

## Cross-Source Validation

Multiple data sources should agree. If they disagree, one of them is wrong — find which.

| Source A | Source B | If they disagree |
|----------|----------|------------------|
| Log file step count | External tracker step count | Logging bug or tracker sync delay |
| Process "alive" (ps) | GPU power (nvidia-smi) | Zombie process or deadlock |
| Log says "training" | GPU utilization <10% | Process waiting on I/O, network, or deadlocked |

If using W&B, load the `wandb-primary` skill (from `wandb/skills`) for W&B-specific cross-validation and API access patterns.

## Trust Hierarchy

When sources disagree, trust the lower-level source:

GPU hardware (nvidia-smi) > process status (ps) > log file > external dashboard
