# Dispatch: Ralph Mode

You are the main agent executing training-monitor directly. You CAN use Team features (TeamCreate, SendMessage) because you are the top-level agent.

## Phase 0: Contract -- Team with Reviewer

1. **TeamCreate** a monitoring team (or reuse existing).
2. Spawn a reviewer teammate using the **Agent tool** with:
   - `team_name`: your team name
   - Role template: `agents/quality-reviewer.md`
   - Instructions: "Review this contract proposal for a monitoring pass. [proposal content]"
3. **SendMessage** to the reviewer with your contract proposal.
4. Read the reviewer's response. If they suggest modifications, negotiate via SendMessage.
5. When agreed, record the final contract.

The reviewer teammate persists through the session -- you will reuse it in Phase 4.

## Phase 2: Collect -- Parallel Sub-Agents

Spawn 4-5 collector sub-agents in parallel using the **Agent tool** (all in one message):

```
Agent(name: "gpu-collector", prompt: "Collect GPU evidence. Run: [commands]. Return structured markdown: [format from phases/2-collect.md]")
Agent(name: "log-collector", prompt: "Collect training log evidence. Log file: [path]. Return structured markdown: [format]")
Agent(name: "process-collector", prompt: "Collect process evidence. PID: [pid]. Return structured markdown: [format]")
Agent(name: "resource-collector", prompt: "Collect resource evidence. Checkpoint dir: [path]. Return structured markdown: [format]")
```

If domain collectors are needed (wandb, k8s, distributed):
```
Agent(name: "domain-collector", prompt: "Collect [domain] evidence. [domain-specific instructions]. Return structured markdown.")
```

All collectors run in parallel. Wait for all to complete. Merge their outputs into the evidence bundle.

## Phase 3: Analyze -- Main Agent

You (main agent) do this directly. No dispatch needed.

## Phase 4: Audit -- SendMessage to Existing Reviewer

The reviewer teammate from Phase 0 is still active.

1. **SendMessage** to the reviewer with:
   - The contract from Phase 0
   - Gate logs from Phases 1-3
   - Your status assessment
   - Instruction: "Audit this monitoring report against the contract. Check process compliance and logical coherence per agents/quality-reviewer.md."
2. Read the reviewer's APPROVED/REJECTED response.
3. If REJECTED: revise and SendMessage again (max 2 rounds).
4. Collect any PITFALLS from the reviewer's response.

## Phase 5a: Troubleshoot -- Sub-Agent

If troubleshooting is triggered:

```
Agent(name: "troubleshooter", prompt: "Investigate this anomaly: [description with numbers]. Follow the troubleshoot procedure in phases/5-act.md. Return: observation, root cause, recommended action, risk, confidence.")
```

## Phase 5b: Strategize -- Main Agent

You (main agent) do this directly. Requires AskUserQuestion for user interaction.

## Phase 6: Persist -- Main Agent

You (main agent) do this directly. No dispatch needed.

## Team Lifecycle

After Phase 6, the reviewer teammate is no longer needed for this pass. If running in a cron loop, the team persists for the next cycle's Phase 0. If this is a one-shot monitoring pass, clean up the team.
