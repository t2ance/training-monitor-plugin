# Step 4: Check Training Metrics

Monitor metrics appropriate to the training type. Not all apply to every job — use the ones relevant to the current training. For domain-specific metrics (e.g., GRPO/RL), see the corresponding file in `domains/`.

## General (All Training Types)

- **Step time consistency**: if step N+1 takes >1.5x step N, investigate (variable sequence lengths, data loading, checkpoint I/O).
- **Time breakdown**: what fraction is forward, backward, optimizer, data loading, communication, checkpoint? If any one component >50% and was not before, investigate.
- **Training-to-eval ratio**: eval should be <20% of wall time. If more, reduce eval frequency or eval samples.

## Loss and Gradients

- **Loss curve**: should decrease monotonically early, then plateau. Sudden spike = bad batch or LR issue.
- **Learning rate**: verify warmup happened, verify current LR matches schedule.
- **Gradient norm**: stable is good. Sudden 10x jump = exploding gradients. Sudden drop to ~0 = vanishing.
- **Eval loss vs train loss gap**: widening gap = overfitting.

## Pretraining-Specific

- Loss should decrease smoothly. Any upward spike >5% of current loss = investigate the batch.
- **Tokens/second throughput**: should be stable. Decrease = data pipeline bottleneck or hardware issue.
- **MFU (model flops utilization)**: <30% on A100 = something is wrong.
