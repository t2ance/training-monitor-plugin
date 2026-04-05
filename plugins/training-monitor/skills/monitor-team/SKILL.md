---
name: monitor-team
description: Cron-based team monitoring loop. Creates a fresh teammate each cycle to run training-monitor, then shuts it down. Team lead only manages lifecycle.
---

# Monitor Team

Orchestrates a periodic monitoring loop using teammates. Each cycle, the team lead
creates a fresh teammate that executes one monitoring pass (using /training-monitor),
then terminates. The team lead does not interpret results or make decisions -- all
autonomy belongs to the teammate.

## Execution Model

```
cron trigger -> team lead creates teammate -> teammate runs /training-monitor once
-> teammate exits -> team lead confirms shutdown -> wait for next trigger
```

Roles:
- **Team lead (you)**: lifecycle only -- create, wait, shutdown. No analysis,
  no reactions to report content.
- **Teammate**: one-shot operator. Runs /training-monitor skill, then exits.
  Authority and behavior determined by user preferences collected at setup.

## Procedure

### Step 1: Collect user preferences

Before setting anything up, ask the user these 4 questions (use AskUserQuestion,
all in one call):

1. **Monitoring target**: What training job(s) should we monitor?
   (e.g., local GPU training, a K8s job, a specific process name or log path)

2. **Work mode**:
   - Fully autonomous -- teammate diagnoses and acts on its own, never asks
     questions. Best when the user is away.
   - Human-in-the-loop -- teammate reports findings and asks for approval
     before making changes. Best when the user is actively working.

3. **Monitoring frequency**: How often should we check?
   (e.g., every 30 minutes, every hour, every 2 hours)

4. **Authority level**:
   - Observe and report only -- no changes to training.
   - Full authority -- teammate may restart, adjust config, kill processes.

### Step 2: Generate cron prompt from preferences

Build the teammate instructions based on the user's answers. The prompt MUST
include all of the following:

**From Q1 (target):**
> You are responsible for monitoring [user's description of the training job].

**From Q2 (work mode):**
- If autonomous: "Do not ask the user or team lead any questions. Diagnose and
  act on your own."
- If human-in-the-loop: "When you find issues that require intervention, report
  your findings and proposed action to the team lead before proceeding."

**From Q4 (authority):**
- If observe-only: "You may only observe and report. Do not stop, restart, or
  modify any training processes or configuration."
- If full authority: "You have full authority. You may start, stop, or restart
  processes and change training config at your own judgment."

**Always included (one-shot lifecycle):**
> Use the /training-monitor skill.
> You are one step in a periodic monitoring loop controlled by the team lead.
> The team lead creates a fresh instance of you every cycle. Your job is to
> complete one monitoring pass, handle any issues you find, then exit. Do not
> loop, do not wait for the next cycle, do not persist. When done, exit.

### Step 3: Set up cron

Create a recurring cron job at the user's requested frequency (from Q3). Pick
an off-round minute (e.g., :23, :47) to avoid API contention.

The cron prompt should instruct the team lead to:
1. Create a new teammate (mode: dontAsk) with the instructions from Step 2.
2. When the teammate finishes, send it a shutdown request.

### Step 4: Trigger first cycle immediately

Spawn the first teammate right away so the user sees results without waiting
for the first cron tick.

### Step 5: On teammate completion

When the teammate sends a report, idle notification, or terminated message:
1. Send a shutdown request.
2. If no response after 2 attempts, stop retrying -- it will time out.
3. Do not analyze, summarize, or react to the report content.

### Step 6: On next cron trigger

Create a brand new teammate. No state is carried in context -- cross-session
state lives in `monitoring-logs/jobs/*.json` (managed by /training-monitor
inside the teammate).
