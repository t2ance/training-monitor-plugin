# Dispatch: Team Mode

You are a teammate executing training-supervisor on behalf of the team lead. You CANNOT use Team features (TeamCreate, SendMessage) because teammates cannot create sub-teams.

**CRITICAL: When using the Agent tool, NEVER pass a `name` or `team_name` parameter.** Passing `name` attempts to create a teammate, which will fail with "Teammates cannot spawn other teammates." Omit `name` entirely to spawn a plain sub-agent.

## Phase 0: Contract -- Single-Round Sub-Agent

1. Write your contract proposal.
2. Spawn a reviewer sub-agent using the **Agent tool** (no `name` parameter):
   ```
   Agent(prompt: "You are a quality reviewer. Read agents/quality-reviewer.md for your role. Review this contract proposal for a supervision pass and return your response (accept, or suggest modifications): [proposal content]")
   ```
3. Read the reviewer's response. Apply any modifications.
4. Record the final contract.

Single round only -- no multi-round negotiation. If the reviewer suggests modifications, incorporate them directly.

## Phase 2: Collect -- Parallel Sub-Agents

Spawn 4-5 collector sub-agents in parallel using the **Agent tool**. Do NOT use `name` parameter -- use plain sub-agents:

```
Agent(prompt: "Collect GPU evidence for PID [pid] on GPUs [list]. Run nvidia-smi commands. Return structured markdown per phases/2-collect.md GPU Collector format.")
Agent(prompt: "Collect training log evidence. Log file: [path]. Return structured markdown per phases/2-collect.md Log Collector format.")
Agent(prompt: "Collect process evidence. PID: [pid]. Return structured markdown per phases/2-collect.md Process Collector format.")
Agent(prompt: "Collect resource evidence. Checkpoint dir: [path]. Return structured markdown per phases/2-collect.md Resource Collector format.")
```

Domain collectors if needed (no `name` parameter):
```
Agent(prompt: "Collect [domain] evidence. [domain-specific instructions]. Return structured markdown.")
```

All collectors run in parallel. Wait for all to complete. Merge their outputs into the evidence bundle.

## Phase 3: Analyze -- Fresh Sub-Agent (CRITICAL)

**Do NOT do this yourself.** Spawn a fresh sub-agent with NO prior context from this session:

```
Agent(prompt: "You are an experienced ML practitioner evaluating a training job. Read phases/3-analyze.md for your procedure. Here is the raw metrics data: [tier1_facts from state]. Here is the training config: [config summary]. Here is the evidence: [Phase 2 evidence bundle]. [If applicable: Here is the domain skill content for this training type.] Perform qualitative-first analysis and return your decision (CONTINUE or STOP) with full reasoning.")
```

**What to include in the prompt**: tier1_facts (raw metrics, timestamps), training config summary, Phase 2 evidence bundle, relevant domain skill content.

**What to NOT include**: previous decisions, previous qualitative descriptions, any tier2_interpretations, any status labels from prior cycles. The sub-agent must judge with fresh eyes.

## Phase 4: Audit -- New Sub-Agent

Spawn a fresh reviewer sub-agent (no `name` parameter):

```
Agent(prompt: "You are a quality reviewer. Read agents/quality-reviewer.md for your role. Audit this supervision report against the contract. Contract: [Phase 0 contract]. Gate logs: [Phases 1-3 paths]. Check contract compliance, process compliance, and logical coherence. Spot-check one claim. Extract pitfalls if noteworthy.")
```

If REJECTED: revise and spawn another reviewer sub-agent for round 2 (max 2 rounds).

## Phase 5a: Troubleshoot -- Sub-Agent

If troubleshooting is triggered (no `name` parameter):
```
Agent(prompt: "Investigate this anomaly: [description with numbers]. Follow phases/5-act.md troubleshoot procedure. Return structured findings.")
```

## Phase 5b: Strategize -- Self

You do this directly. Requires AskUserQuestion.

## Phase 6: Persist -- Self

You do this directly. No dispatch needed.
