---
name: distributed-monitor
description: Heuristics for monitoring multi-GPU and multi-process distributed training. Common patterns, NCCL diagnostics, known failure modes. Reference knowledge, not rules.
---

# Distributed Training Monitor

Heuristics for monitoring distributed training jobs (DDP, FSDP, DeepSpeed, Megatron, Ray, etc.). This skill provides **reference knowledge** about common distributed training patterns — not rules or checklists. Use it to inform your reasoning about what to check and what failure modes are known.

## Multi-Process Evidence Collection

The launcher process (e.g., torchrun, Ray driver, deepspeed launcher) may be dead while worker processes continue running. Always check ALL process PIDs, not just the launcher.

```bash
# Find all related training processes
ps aux | grep <training_script_name>
# Check each worker PID
ps -p <PID1>,<PID2>,... -o pid,etime,state --no-headers
```

## Multi-Process Stall Detection

In addition to single-process stall detection:

1. Check ALL worker PIDs, not just the launcher. Launcher dead + workers alive is a known pattern.
2. Check for NCCL hangs: all GPUs show high memory usage but idle power, no log output from any rank. One rank crashed or hung, others wait at collective op.
3. Check for stragglers: if one rank is slower than others (different GPU, thermal throttling), all ranks wait at the synchronization barrier.

### NCCL Diagnostics

```bash
# Check if NCCL env vars are set
env | grep NCCL
# Check for NCCL errors in logs
grep -i "nccl\|timeout\|watchdog" <log_file>
```

Common NCCL issues:
- **NCCL timeout**: one rank crashed, others wait at collective op. Kill all ranks and restart.
- **NCCL init failure**: wrong MASTER_ADDR/PORT, firewall, or incompatible NCCL versions across nodes.
- **Topology mismatch**: ranks moved to different nodes with different NVLink/PCIe topology after restart.

## Network Resources

- **NCCL timeouts**: inter-GPU communication failing or slow.
- **Inter-node network**: for multi-node training, check for network congestion or packet loss.
- If gradient sync dominates step time, consider gradient compression or reducing communication frequency.

## Distributed Resource Anomalies

- **Uneven GPU memory**: one GPU uses significantly more memory than others. Check for model parallel imbalance or one rank accumulating extra state.
- **GPU utilization divergence**: all GPUs should show similar utilization patterns. If one is consistently lower, it may have a slower interconnect or be thermal throttling.
