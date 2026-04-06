---
name: grpo-monitor
description: How an experienced RL practitioner thinks about GRPO, PPO, and other RL training. Describes the reasoning process for evaluating RL training health, not numerical thresholds.
---

# GRPO / RL Training: How to Think About It

This skill describes how an experienced RL practitioner evaluates training health. It is a **thinking guide** -- it tells you what to look at and what the patterns mean. It is NOT a checklist of thresholds to compare against.

Use this to inform your qualitative description and holistic judgment in Phase 3. If what you observe doesn't match these patterns, trust your observation and explain why.

## Reading the Reward/Score Trajectory

The reward or score trajectory is the primary signal. Ask yourself:

**Is the trajectory still rising?** Early in training, reward should clearly improve. The key question is whether the rate of improvement is itself changing:
- Rising and accelerating: healthy early learning
- Rising but decelerating: approaching a plateau -- not necessarily bad, but watch closely
- Flat: the training may have extracted what it can from the current setup
- Declining after a peak: the model is getting worse, not better

**Is the trajectory noisy or smooth?** Some noise is normal in RL (stochastic rewards). The question is whether you can see a clear trend through the noise, or whether the noise IS the signal (no trend at all).

**Has there been a peak followed by decline?** If reward peaked and then fell, the model may have overfit or the policy may have drifted past the sweet spot. Note the peak location -- if the model was better earlier, continuing training is actively harmful.

## Reading the KL-Reward Relationship

In GRPO/PPO, KL divergence measures how far the policy has moved from the reference model. The relationship between KL and reward is the most important diagnostic:

**KL rising with reward rising**: The model is changing AND improving. This is healthy -- the policy is learning useful behaviors that differ from the reference.

**KL rising with reward flat**: The model is changing but NOT improving. This is the critical warning pattern. The policy keeps drifting from the reference but the drift is not producing better results. An experienced practitioner would recognize this as "the training has run its course" -- continuing will only increase divergence without benefit.

**KL rising with reward declining**: The model is changing, getting worse, AND moving further from a known-good reference. This is actively destructive. Stop immediately.

**KL stable with reward rising**: Ideal but rare -- the model is finding ways to improve without drifting. Usually seen only in early training or with strong KL penalties.

**KL stable with reward flat**: The training has converged. Nothing is changing. Whether to stop depends on whether the current performance is satisfactory.

## Reading pg_loss Stability

The policy gradient loss reflects the direction and magnitude of policy updates:

**Stable and gradually declining**: Normal -- the policy updates are getting smaller as the model converges.

**Oscillating around zero**: The policy gradient has no consistent direction. Updates are random noise. Continuing training is pointless.

**Went negative after being positive**: The update direction reversed -- the model is now being pushed away from reward improvement. This is a strong signal to stop.

**Suddenly doubled or halved**: A discontinuity in pg_loss magnitude suggests something changed abruptly (data distribution shift, learning rate schedule step, gradient explosion/collapse). Investigate what caused the discontinuity.

## Generation Quality

RL training can produce metric improvement that masks generation degradation:

**Response length**: If responses suddenly get much shorter or much longer, the model may have found a reward shortcut (short junk to avoid negative reward, or padding to hit length bonuses) rather than genuinely improving.

**Response diversity**: If all outputs look the same, the model has collapsed to a single strategy. It is no longer exploring, and the reward signal is meaningless.

**Coherence**: If available, spot-check actual model outputs. A model can achieve good reward scores while producing incoherent text if the reward function has exploitable patterns.

## The Cost-Benefit Judgment

After reading the metrics, the final question is practical:

**What is the realistic upside of continuing?** If reward has plateaued and KL is rising, the realistic upside is near zero -- the model is unlikely to discover a new improvement direction on its own.

**What is the risk of continuing?** Rising KL means the model is drifting further from a known-good reference. The longer it trains, the harder it is to recover. Each additional step increases the distance from the reference without guaranteed benefit.

**Where is the best checkpoint?** If the model peaked earlier, the best version already exists on disk. Continuing training makes the current model worse, not better. The right action is to stop and use the peak checkpoint.

## Resource Usage Patterns

RL training has a generation/rollout phase that uses GPU differently than training:

**Generation phase**: GPU power is noticeably lower than during training. This is normal for autoregressive token-by-token generation. If the generation phase is taking a disproportionate share of step time, it may be the bottleneck.

**Memory during generation vs training**: Memory usage may shift between phases. If memory is near capacity during one phase, the other phase has headroom. This is normal but worth noting if memory is tight.

## Background Process Conflicts

If a model merger, checkpoint uploader, or other background process is running alongside training, it may compete for resources. Look for: step time increases during merge/upload operations, or memory spikes that coincide with background activity.
