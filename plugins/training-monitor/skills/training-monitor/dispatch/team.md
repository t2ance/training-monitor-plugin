# Dispatch: Team Mode

You are a teammate executing training-monitor on behalf of the team lead. You CANNOT use Team features (TeamCreate, SendMessage) because teammates cannot create sub-teams. Use the **Agent tool** for all dispatch.

## Phase 0: Contract -- Single-Round Sub-Agent

1. Write your contract proposal.
2. Spawn a reviewer sub-agent using the **Agent tool**:
   ```
   Agent(prompt: "You are a quality reviewer. Read agents/quality-reviewer.md for your role. Review this contract proposal for a monitoring pass and return your response (accept, or suggest modifications): [proposal content]")
   ```
3. Read the reviewer's response. Apply any modifications.
4. Record the final contract.

Single round only -- no multi-round negotiation. If the reviewer suggests modifications, incorporate them directly.

## Phase 2: Collect -- Parallel Sub-Agents

Same as ralph mode. Spawn 4-5 collector sub-agents in parallel using the **Agent tool**:

```
Agent(name: "gpu-collector", prompt: "Collect GPU evidence. Run: [commands]. Return structured markdown.")
Agent(name: "log-collector", prompt: "Collect training log evidence. Log file: [path]. Return structured markdown.")
Agent(name: "process-collector", prompt: "Collect process evidence. PID: [pid]. Return structured markdown.")
Agent(name: "resource-collector", prompt: "Collect resource evidence. Checkpoint dir: [path]. Return structured markdown.")
```

Domain collectors if needed. All run in parallel.

## Phase 3: Analyze -- Self

You do this directly. No dispatch needed.

## Phase 4: Audit -- New Sub-Agent

Spawn a fresh reviewer sub-agent (cannot reuse Phase 0's reviewer):

```
Agent(prompt: "You are a quality reviewer. Read agents/quality-reviewer.md for your role. Audit this monitoring report against the contract. Contract: [Phase 0 contract]. Gate logs: [Phases 1-3 paths]. Check contract compliance, process compliance, and logical coherence. Spot-check one claim. Extract pitfalls if noteworthy.")
```

If REJECTED: revise and spawn another reviewer sub-agent for round 2 (max 2 rounds).

## Phase 5a: Troubleshoot -- Sub-Agent

Same as ralph mode:
```
Agent(prompt: "Investigate this anomaly: [description]. Follow phases/5-act.md troubleshoot procedure. Return structured findings.")
```

## Phase 5b: Strategize -- Self

You do this directly. Requires AskUserQuestion.

## Phase 6: Persist -- Self

You do this directly. No dispatch needed.
