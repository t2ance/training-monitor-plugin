# Phase 3: Analyze

## Goal

Answer: **is this training making productive progress toward its objective?**

This is the core judgment phase. The main agent works with the evidence bundle from Phase 2 and derives a status assessment with articulated reasoning.

## Part 1: Compare Predictions vs Evidence

Build a comparison table:

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

### Cross-Source Validation

Multiple data sources should agree. If they disagree, one of them is wrong -- find which.

| Source A | Source B | If they disagree |
|----------|----------|------------------|
| Log file step count | External tracker step count | Logging bug or tracker sync delay |
| Process "alive" (ps) | GPU power (nvidia-smi) | Zombie process or deadlock |
| Log says "training" | GPU utilization <10% | Process waiting on I/O, network, or deadlocked |

### Trust Hierarchy

When sources disagree, trust the lower-level source:

GPU hardware (nvidia-smi) > process status (ps) > log file > external dashboard

## Part 2: Derive Judgment Criteria

Do NOT use a fixed checklist -- derive criteria from the training's own artifacts.

### 1. Read the training artifacts

- Training config (YAML, JSON, Python script, command line args)
- What loss function is defined? What optimizer?
- What metrics are actually being logged?
- Is there a reward function? An evaluation script?

### 2. Identify the key progress indicator

The metric that most directly measures whether the training objective is being achieved:

- It might be loss (SFT, pretraining)
- It might be reward (RL)
- It might be a behavior metric (output length, pass rate, ELO rating)
- It might be equilibrium (GAN -- bounded oscillation, not monotonic decrease)
- It might be something the user specified
- If per-job state exists, use the previously derived indicator unless something changed

**State your choice explicitly and explain WHY.**

### 3. Establish a baseline

What would this metric look like if the model were NOT learning?

- For cross-entropy loss: random baseline = -ln(1/num_classes) or ln(2) for binary
- For reward: zero or random-policy reward
- For output length: initial model's average output length
- For custom metrics: reason about what "no learning" means

### 4. State expected behavior

What does progress look like? Be specific:

- "Loss should decrease over time" (SFT)
- "Reward should increase over time" (RL)
- "Generator and discriminator losses should oscillate in a bounded range [X, Y]" (GAN)

This must be stated explicitly so the reviewer can check logical coherence.

### 5. Evaluate progress

- Is the key indicator behaving as expected?
- Is this distinguishable from noise? (compare rolling averages, not individual steps)
- If deferring judgment for warmup: state (1) what signal ends warmup, (2) maximum step count before forced assessment.

### 6. Assess status with articulated reasoning

"The key indicator for this training is [X] because [reason]. Expected behavior: [Z]. The baseline is [Y]. After [N] steps, [X] has moved from [initial] to [current]. This [matches/does not match] the expected behavior because [reasoning]. Therefore status is [STATUS]."

## Part 3: General Observations (Always Check)

These apply regardless of key progress indicator:

- **Step time consistency**: if step N+1 takes >1.5x step N, investigate.
- **Time breakdown**: if any one component >50% of step time and was not before, investigate.
- **Training-to-eval ratio**: eval should be <20% of wall time.

## Domain Skills as Heuristics

Domain skills provide knowledge about common patterns. You MUST load the relevant skill when conditions match. Use the knowledge to **inform your reasoning** -- not as a checklist to follow. If a domain skill says "reward should increase" but your analysis suggests otherwise, trust your analysis and explain.

## Gate Log Format

Record in `monitoring-logs/<timestamp>/3-analyze.md`:

### Comparison
- Prediction vs actual table
- Deviations flagged

### Criteria Derivation
- Key progress indicator and WHY
- Baseline
- Expected behavior

### Assessment
- Status: [HEALTHY / WARNING / CRITICAL / UNCERTAIN]
- Articulated reasoning (the full paragraph from step 6)
- General observations (step time, eval ratio)

### Contract Compliance
- Which contract focus areas were addressed
- Which acceptance criteria were met
