# Step 4: Check Training Metrics

## Goal

Answer: **is this training making productive progress toward its objective?**

## How to Derive Judgment Criteria

Do NOT use a fixed checklist. The relevant metrics depend on what THIS training is trying to achieve. Derive the criteria from the training's own artifacts.

### 1. Read the training artifacts

- Training config (YAML, JSON, Python script, command line args)
- What loss function is defined? What optimizer?
- What metrics are actually being logged? (read the log output)
- Is there a reward function? An evaluation script?

### 2. Identify the key progress indicator

The metric that most directly measures whether the training objective is being achieved. This varies by training type:

- It might be loss (SFT, pretraining)
- It might be reward (RL)
- It might be a behavior metric (output length, pass rate, ELO rating)
- It might be equilibrium (GAN — bounded oscillation, not monotonic decrease)
- It might be something the user specified that isn't obvious from config
- If per-job state exists from a previous session, use the previously derived indicator unless something changed

**State your choice explicitly and explain WHY.**

### 3. Establish a baseline

What would this metric look like if the model were NOT learning?

- For cross-entropy loss: random baseline = -ln(1/num_classes) or ln(2) for binary
- For reward: zero or random-policy reward
- For output length: initial model's average output length
- For custom metrics: reason about what "no learning" means for this metric

### 4. Evaluate progress

- Is the key indicator moving away from the baseline in the expected direction?
- Is this distinguishable from noise? (compare rolling averages, not individual steps)
- Has there been sufficient training time? (don't judge too early — but also don't wait forever)

### 5. Assess status with articulated reasoning

Write your reasoning, not just a label:

- "The key indicator for this training is [X] because [reason]. The baseline is [Y]. After [N] steps, [X] has moved from [initial] to [current], which is [above/below/at] baseline. This [does/does not] show productive progress because [reasoning]."

## General Observations (All Training Types)

These are useful signals regardless of what the key indicator is:

- **Step time consistency**: if step N+1 takes >1.5x step N, investigate (variable sequence lengths, data loading, checkpoint I/O).
- **Time breakdown**: if any one component >50% of step time and was not before, investigate.
- **Training-to-eval ratio**: eval should be <20% of wall time.

## Domain Skills as Heuristics

Domain skills (grpo-monitor, distributed-monitor, etc.) provide knowledge about common patterns in their domains:
- What metrics are typically important
- What normal behavior looks like
- What common failure modes exist

Use them as **reference knowledge to inform your reasoning**, not as checklists to follow. If a domain skill says "reward should increase" but your analysis of the training config suggests otherwise, trust your analysis and explain your reasoning.
