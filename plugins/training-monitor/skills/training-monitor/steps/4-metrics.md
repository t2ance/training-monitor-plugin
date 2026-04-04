# Step 4: Check Training Metrics

## Hypothesis Testing Methodology

The goal of this step is to answer: **is this training making productive progress toward its objective?**

This is NOT a fixed checklist. The relevant metrics depend on what the training is trying to achieve. You must first understand the training's objective, then identify the right metrics to evaluate progress.

### How to Apply

1. **Identify the training objective.** What is this training trying to optimize? Loss? Reward? Output quality? A specific behavior (e.g., longer outputs, tool use accuracy)?

2. **Identify the key progress indicator.** This is the metric that most directly measures whether the objective is being achieved. Examples:
   - SFT/pretraining: loss decreasing
   - GRPO/RL: reward increasing, or a specific behavior metric (e.g., output length, pass rate)
   - Alignment: preference win rate improving
   - The user may specify a custom indicator — always ask or check the training config

3. **Establish the baseline.** What would this metric look like if the model were NOT learning? (e.g., random baseline loss, zero reward, random output length)

4. **Evaluate progress.** Is the key indicator moving in the right direction, away from the baseline? Is it statistically distinguishable from random?

5. **Assess status:**
   - **HEALTHY**: key indicator is clearly moving in the right direction, away from baseline
   - **WARNING**: key indicator is flat, unclear, or still at baseline level after sufficient steps
   - **CRITICAL**: key indicator is moving in the WRONG direction, or NaN/Inf/collapsed

**"Process alive + GPU busy" is never evidence of progress. It only means the job has not crashed.**

## General Metrics (All Training Types)

These apply regardless of the training objective:

- **Step time consistency**: if step N+1 takes >1.5x step N, investigate (variable sequence lengths, data loading, checkpoint I/O).
- **Time breakdown**: what fraction is forward, backward, optimizer, data loading, communication, checkpoint? If any one component >50% and was not before, investigate.
- **Training-to-eval ratio**: eval should be <20% of wall time. If more, reduce eval frequency or eval samples.

## Loss and Gradients

When loss is the relevant progress indicator:

- **Loss curve**: should decrease monotonically early, then plateau. Sudden spike = bad batch or LR issue.
- **Learning rate**: verify warmup happened, verify current LR matches schedule.
- **Gradient norm**: stable is good. Sudden 10x jump = exploding gradients. Sudden drop to ~0 = vanishing.
- **Eval loss vs train loss gap**: widening gap = overfitting.

## Pretraining-Specific

- Loss should decrease smoothly. Any upward spike >5% of current loss = investigate the batch.
- **Tokens/second throughput**: should be stable. Decrease = data pipeline bottleneck or hardware issue.
- **MFU (model flops utilization)**: <30% on A100 = something is wrong.
