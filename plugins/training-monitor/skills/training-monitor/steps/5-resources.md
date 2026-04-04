# Step 5: Check Resources

Answer each item with evidence.

## 1. GPU Compute (Power-Based)

What is actual power / TDP? Break down by training phase:

- **Training (forward + backward + optimizer)**: expect 60-90% of TDP. If lower, check if batch size is too small or data loading is starving the GPU.
- **Checkpoint**: expect idle power. This is I/O-bound.
- If overall power-weighted utilization across a full step is <40% of TDP, the pipeline has a major bottleneck. Identify which phase dominates wall time and optimize that phase.

## 2. GPU Memory

How close to max? Memory usage varies by phase within a single step. Check peak memory (during backward pass), not just current snapshot. If peak was >95% on any previous step, future batches with longer sequences WILL OOM.

## 3. Disk Space

Enough room for remaining checkpoints? Estimate: current_ckpt_size * remaining_saves.

## 4. CPU Memory

Growing unboundedly? Check for memory leak in data pipeline, tokenizer cache, etc.

## 5. Network

Any failures affecting training? Check for data download retries, checkpoint upload failures, or other network-related errors in the log.

## 6. Idle Resources

Any GPUs allocated but unused? Zombie processes occupying memory?
