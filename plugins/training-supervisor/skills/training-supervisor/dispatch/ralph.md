# Dispatch: Ralph Mode

You are the main agent executing training-supervisor directly. You CAN use Team features (TeamCreate, SendMessage) because you are the top-level agent.

**CRITICAL: Only the reviewer should be a named teammate (for SendMessage). All collectors and troubleshooters MUST be plain sub-agents -- do NOT pass `name` or `team_name` to them.** Passing `name` to a collector adds it to the team roster, which wastes resources and complicates lifecycle management.

## Phase 0: Contract -- Team with Reviewer

1. **TeamCreate** a supervision team (or reuse existing).
2. Spawn a reviewer teammate using the **Agent tool** with:
   - `name`: "reviewer" (this one DOES get a name -- it needs to be addressable for Phase 4)
   - `team_name`: your team name
   - Role template: `agents/quality-reviewer.md`
   - Instructions: "Review this contract proposal for a supervision pass. [proposal content]"
3. **SendMessage** to the reviewer with your contract proposal.
4. Read the reviewer's response. If they suggest modifications, negotiate via SendMessage.
5. When agreed, record the final contract.

The reviewer teammate persists through the session -- you will reuse it in Phase 4.

## Phase 2: Collect -- Parallel Sub-Agents (NO name parameter)

Spawn 4-5 collector sub-agents in parallel using the **Agent tool**. These are plain sub-agents, NOT teammates. Do NOT pass `name` or `team_name`:

```
Agent(prompt: "Collect GPU evidence for PID [pid] on GPUs [list]. Run nvidia-smi commands. Return structured markdown per phases/2-collect.md GPU Collector format.")
Agent(prompt: "Collect training log evidence. Log file: [path]. Return structured markdown per phases/2-collect.md Log Collector format.")
Agent(prompt: "Collect process evidence. PID: [pid]. Return structured markdown per phases/2-collect.md Process Collector format.")
Agent(prompt: "Collect resource evidence. Checkpoint dir: [path]. Return structured markdown per phases/2-collect.md Resource Collector format.")
```

If domain collectors are needed (wandb, k8s, distributed) -- also NO `name`:
```
Agent(prompt: "Collect [domain] evidence. [domain-specific instructions]. Return structured markdown.")
```

All collectors run in parallel. Wait for all to complete. Merge their outputs into the evidence bundle.

## Phase 3: Analyze -- Fresh Sub-Agent (CRITICAL)

**Do NOT do this yourself.** Spawn a fresh sub-agent using the Agent tool (no `name` parameter) with NO prior context:

```
Agent(prompt: "You are an experienced ML practitioner evaluating a training job. Read phases/3-analyze.md for your procedure. Here is the raw metrics data: [tier1_facts from state]. Here is the training config: [config summary]. Here is the evidence: [Phase 2 evidence bundle]. [If applicable: Here is the domain skill content for this training type.] Perform qualitative-first analysis and return your decision (CONTINUE or STOP) with full reasoning.")
```

**What to include**: tier1_facts (raw metrics, timestamps), training config summary, Phase 2 evidence bundle, relevant domain skill content.

**What to NOT include**: previous decisions, previous qualitative descriptions, any tier2_interpretations. The sub-agent must judge with fresh eyes.

If the sub-agent's judgment is unclear or it flags confusion about something unusual, you may provide tier2_interpretations in **third-person framing** via a follow-up message:

> "A previous supervision session noted: [description]. That session concluded [decision] because [reasoning]. Does this context change your assessment?"

## Phase 4: Audit -- SendMessage to Existing Reviewer

The reviewer teammate from Phase 0 is still active.

1. **SendMessage** to the reviewer with:
   - The contract from Phase 0
   - Gate logs from Phases 1-3
   - Your status assessment
   - Instruction: "Audit this supervision report against the contract. Check process compliance and logical coherence per agents/quality-reviewer.md."
2. Read the reviewer's APPROVED/REJECTED response.
3. If REJECTED: revise and SendMessage again (max 2 rounds).
4. Collect any PITFALLS from the reviewer's response.

## Phase 5a: Troubleshoot -- Sub-Agent (NO name parameter)

If troubleshooting is triggered -- plain sub-agent, no `name`:

```
Agent(prompt: "Investigate this anomaly: [description with numbers]. Follow the troubleshoot procedure in phases/5-act.md. Return: observation, root cause, recommended action, risk, confidence.")
```

## Phase 5b: Strategize -- Main Agent

You (main agent) do this directly. Requires AskUserQuestion for user interaction.

## Phase 6: Persist -- Main Agent

You (main agent) do this directly. No dispatch needed.

## Team Lifecycle

After Phase 6, the reviewer teammate is no longer needed for this pass. If running in a cron loop, the team persists for the next cycle's Phase 0. If this is a one-shot supervision pass, clean up the team.
