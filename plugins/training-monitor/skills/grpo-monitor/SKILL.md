---
name: grpo-monitor
description: Monitoring guidance for GRPO, PPO, and other RL training. Covers RL-specific metrics, generation phase resource usage, and RL anomaly patterns. Use standalone or alongside training-monitor.
---

# GRPO / RL Training Monitor

Monitoring guidance for training jobs that use GRPO, PPO, RLOO, or other RL algorithms. This skill can be used standalone or loaded alongside `training-monitor` for domain-specific coverage.

## RL Metrics

- **Reward trend**: is mean reward increasing over steps? Flat for 5+ steps after warmup = stalled.
- **Reward distribution**: are rewards binary (0/1) or continuous? If binary, what is the positive ratio?
- **KL divergence**: if use_kl_loss=true, is KL growing? KL > 10 = model diverging dangerously.
- **Entropy**: decreasing is normal (model getting more confident). Near-zero = collapsed, no exploration left.
- **Clip fraction**: >0.3 = updates too aggressive, LR may be too high.
- **Generation length**: is mean response length changing? Sudden drop = model learned to produce short junk. Sudden increase = model learned to pad.
- **Aborted ratio**: >5% = sequences hitting max_length too often, may need longer max_response_length.
- **Train vs eval divergence**: train reward up but eval flat/down for 3+ evals = overfitting.

## Generation Phase Resource Usage

RL training has a generation/rollout phase not present in standard training:

- **Generation phase**: expect 20-40% of GPU TDP. This is normal for autoregressive token-by-token generation. Optimize by increasing batch size or parallel sequences.

### Phase Time Balance

A healthy GRPO step time breakdown:

| Phase | Expected % | If higher |
|-------|-----------|-----------|
| Rollout generation | 5-15% | Increase inference throughput |
| Log prob computation (old + ref) | 20-40% | Increase micro_batch_size |
| Actor update (backward + optimizer) | 30-50% | This is the useful work |
| Checkpoint save | <10% | Reduce save_freq or use faster storage |

If any single phase is >60%, it is the bottleneck. Focus optimization there.

## RL Metric Anomalies

In addition to general metric anomalies (NaN, loss=0, gradient explosion):

- **Reward declining monotonically** for 3+ steps after warmup = training is degrading.
- **Entropy collapsed near zero** = model has no exploration left, learning signal gone.
- **KL divergence > 10** = model diverging dangerously from reference policy.

## Generation Quality Collapse

**Detect:** response_length suddenly drops to minimum, OR response_length hits max_response_length for >50% of samples (clip_ratio > 0.5), OR reward becomes constant (always 0 or always 1).

**Root causes:**
1. Model learned to produce short/empty responses to avoid negative reward.
2. Model found reward hack — producing a fixed pattern that always scores high.
3. Temperature too low — no diversity, all samples identical, advantages all zero, no learning signal.
4. KL penalty too high �� model cannot deviate from reference, stuck at initial policy.

**Action:** Check actual generated text (not just metrics). Read a few samples from the log. Is the model producing coherent, diverse outputs? If collapsed, this likely requires a config change (temperature, KL coef, reward function) — use `team-decision-review` before changing.

## RL Overfitting Response

In addition to general overfitting actions (early stopping, regularization, reduce LR):

- Increase KL penalty to keep the model closer to the reference policy.
- Check if the reward function itself is being exploited (reward hacking).

## Background Process Conflicts

**Detect:** container RAM usage spikes during checkpoint merge+upload, training step time increases during merge, merge process fails with OOM.

**Action:**
1. Check if model merger and training are running simultaneously: `ps aux | grep -E "model_merger|main_ppo"`.
2. If RAM conflict: stagger the merge to run only during the rollout phase (lower RAM usage) or increase container memory request.
3. If the merge process keeps failing: disable background push, add only end-of-training push.
