# Step 6: Reviewer Audit

## Goal

An independent sub-agent reviews your monitoring work for process compliance and logical coherence. You cannot meaningfully review your own reasoning -- this step exists because self-review is unreliable.

## How to Spawn the Reviewer

Spawn a sub-agent (using the Agent tool) with:
1. Instruction to read `agents/quality-reviewer.md` for the review checklist
2. Your gate log files from Steps 1-5 (file paths)
3. The monitoring report you would produce based on Steps 1-5

The reviewer checks PROCESS, not domain content:
- Did you derive judgment criteria from training artifacts?
- Did you state the key progress indicator, expected behavior, and WHY?
- Did you establish a baseline?
- Does your conclusion follow from your stated evidence? (logical coherence)
- Spot-check: the reviewer independently verifies one claim you made

## Handling Reviewer Feedback

If the reviewer sends APPROVED: proceed to Step 7.

If the reviewer sends REJECTED with specific feedback:
1. Read the feedback carefully
2. Re-run the specific checks the reviewer questioned (do not guess or restate old data)
3. Revise your assessment if new evidence warrants it
4. Resubmit to the reviewer

Maximum 2 rounds. After 2 rounds, record "what remains unverified" and proceed.

## Gate Log Format

Record in `monitoring-logs/<timestamp>/6-review.md`:
- Reviewer's verdict (APPROVED / REJECTED)
- What was spot-checked and the result
- If REJECTED: what was revised and why
- If unresolved after 2 rounds: what remains unverified
