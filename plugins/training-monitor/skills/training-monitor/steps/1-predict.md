# Step 1: Write Predictions

Write predictions for EVERY running job based on previous checks and training config. This MUST happen before any tool calls.

## Prediction Template

```
PREDICTIONS (written before reading any data):
---
[Job Name]
- Running since: [timestamp or duration]
- Expected current step: [N / total] (based on elapsed time and observed step rate)
- Expected step time: [X min] (based on previous steps)
- Expected GPU power: [X W out of Y W TDP] (primary utilization metric)
- Expected GPU memory: [X GB per GPU]
- Expected phase: [init / forward / backward / optimizer / checkpoint / eval / idle]
- Expected key metrics: [loss/reward range based on trend from previous steps]
- Risk factors: [what could go wrong given current state]
---
```

## GPU Power as Primary Utilization Indicator

nvidia-smi "utilization %" is misleading. It only measures whether any kernel is running, not how hard the GPU is working. A GPU doing token-by-token inference shows 99% utilization but only 80W power (20% of TDP).

**Always use power draw as the primary utilization indicator.**

### Power Reference

| GPU | TDP (SXM) | TDP (PCIe) | Idle |
|-----|-----------|------------|------|
| A100 80GB | 400W | 300W | ~60W |

Real utilization = (actual_power - idle_power) / (TDP - idle_power)

### Power by Training Phase

| Phase | Expected Power | Notes |
|-------|---------------|-------|
| Heavy training (backward pass) | 250-400W | High utilization, this is the useful work |
| Light inference (generation) | 80-150W | Normal for autoregressive generation |
| Idle / waiting | 60-70W | No useful work happening |
