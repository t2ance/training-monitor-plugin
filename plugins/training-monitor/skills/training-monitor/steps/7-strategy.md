# Step 7: Strategy Gate Log

## Goal

Record the strategist's analysis, hypotheses, user decision, and execution plan for this monitoring session.

## Format

This gate log is a **decision document**, not an evidence document. Each section records a stage of the strategic decision process.

### Input

- Monitoring report summary (status, key indicator, deviations)
- Troubleshooter report (if Phase 5 was triggered)
- User history response (what was tried before, or "first session / no history")
- Previous hypothesis evaluation (if pending hypothesis existed: CONFIRMED/REFUTED/INCONCLUSIVE with evidence)

### Execution

- Decision Gate classification with evidence
- "What Changed Our Belief" statement
- 3 hypotheses (or 0-3 suggestions for CONTINUE) with all structured fields
- User's choice and reasoning (or "CONTINUE: no user interaction needed")

### Output

- Chosen hypothesis (or "no changes")
- Execution plan with specific commands/config changes
- Success/failure criteria
- Time budget and evaluate_after timestamp

### Next

- What to monitor in the next session to evaluate whether the chosen hypothesis was confirmed or refuted
- Any carry-forward items (inconclusive hypotheses, unresolved questions)
