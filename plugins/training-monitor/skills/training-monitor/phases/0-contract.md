# Phase 0: Contract

## Goal

Before any evidence collection, establish an explicit agreement between the monitor and reviewer on what this monitoring pass will focus on and what constitutes a passing audit.

This is NOT bureaucracy. It solves a specific problem: without a contract, the reviewer can only check generic process compliance. With a contract, the reviewer checks whether the monitor actually addressed what matters THIS time.

## Contract Proposal

Write a contract proposal with these sections:

### Focus Areas

Based on per-job state and pitfalls, list what this pass should prioritize:

- **From previous session**: what was the status? what anomalies were found? what strategy was chosen?
- **From pitfalls**: what mistakes should we avoid repeating?
- **From strategy.evaluate_after**: is there a pending hypothesis that needs evaluation this session?

If first session (no prior state): focus areas come from the training config and initial observations. State: "First session -- deriving focus from training config."

### Acceptance Criteria

What must the monitoring report contain for the reviewer to APPROVE:

- Which evidence sources MUST be checked (e.g., "must verify GPU 3 status -- it showed anomalies last session")
- Which domain skills MUST be loaded (e.g., "GRPO training -- grpo-monitor required")
- Any specific claims that MUST be spot-checked

### Pass Definition

What counts as a "complete" monitoring pass:

- All 5 evidence categories collected (GPU, logs, processes, resources, domain)
- Key progress indicator derived and assessed
- Comparison against predictions completed
- Status assigned with articulated reasoning

## Negotiation

Send the proposal to the reviewer. The reviewer may:
- Accept as-is
- Add focus areas the monitor missed
- Strengthen acceptance criteria
- Flag pitfalls the monitor forgot to account for

Dispatch determines the mechanism (Team multi-round vs Agent single-round). Regardless of mechanism, the goal is agreement -- not rubber-stamping.

## First Session

If no per-job state exists:
- Focus areas: "Initial assessment -- derive criteria from training config, establish baseline"
- Acceptance criteria: "Must read training config, identify key progress indicator, explain choice"
- Pass definition: same as default

## Gate Log Format

Record in `monitoring-logs/<timestamp>/0-contract.md`:

```
CONTRACT: [job name] -- [session timestamp]
---
Focus areas:
- [list]

Acceptance criteria:
- [list]

Pass definition:
- [list]

Reviewer modifications:
- [what the reviewer added/changed, or "none"]

Agreement: [AGREED / AGREED WITH MODIFICATIONS]
---
```
