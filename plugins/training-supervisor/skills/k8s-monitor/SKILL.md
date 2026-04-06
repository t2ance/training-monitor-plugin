---
name: k8s-monitor
description: Heuristics for monitoring training jobs on Kubernetes. Common patterns, pod anomalies, scheduling failures, escalation ladder. Reference knowledge, not rules.
---

# Kubernetes Training Monitor

Heuristics for monitoring training jobs on Kubernetes. This skill provides **reference knowledge** about common K8s patterns and failure modes — not rules or checklists. Use it to inform your reasoning about what to check in K8s environments.

## K8s Evidence Collection

```bash
# Pod status
kubectl get pods -n <ns> -l job-name=<name> -o wide

# Recent logs
kubectl logs -n <ns> -l job-name=<name> --tail=20

# Error scan
kubectl logs -n <ns> -l job-name=<name> --tail=200 2>/dev/null | grep -i "error\|exception\|killed\|oom\|timeout\|nan\|inf" | tail -5
```

## K8s Resource Checks

- Check PVC capacity: `kubectl exec <pod> -n <ns> -- df -h /mount/path`
- Disk >80% on PVC: delete old checkpoints or resize PVC.

## Infrastructure Failures

**Detect:** pod not Running, RESTARTS > 0, pod evicted/preempted, NCCL errors, OOM-killed, node not ready, pod stuck in Pending/ContainerCreating >5 min.

**Action:**
1. `kubectl describe pod <name> -n <ns> | tail -30` for events.
2. Full error from last 200 log lines.
3. Preempted: resubmit immediately (use same YAML, K8s picks new node).
4. OOM-killed: check if GPU OOM or container RAM OOM. Reduce batch_size (GPU) or increase memory request (RAM). If increased memory still cannot be scheduled, fall back to reduced config.
5. NCCL error: verify NCCL env vars, check if node changed (different GPU topology).

## Unschedulable / Pending >5 min

The cluster cannot satisfy your resource request. Do NOT wait indefinitely. Diagnose and adapt:

```bash
kubectl describe pod <name> -n <ns> | grep -A3 "Events\|FailedScheduling"
```

Common reasons:
- **Insufficient memory**: reduce memory request (e.g. 256Gi to 192Gi). Estimate actual need from previous run's peak RSS.
- **Insufficient GPU**: requested GPU type unavailable. Try a different GPU type. Adjust config (batch_size, model parallel) to fit the new GPU's memory.
- **Node taints / affinity**: remove nodeSelector or add tolerations.
- **PVC region mismatch**: PVC is in one region, available GPUs in another.

### Scheduling Escalation Ladder

Try in order, stop when it schedules:

1. Reduce memory request to minimum viable (peak RSS from last run + 20% headroom)
2. Switch GPU type (check available: `kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu`)
3. Drop PVC mount (use HF push for checkpoint persistence instead)
4. Reduce to 1 GPU (if model fits with gradient checkpointing + smaller batch)
5. Alert user: "Cannot schedule on K8s with any viable config. Need manual intervention."

After each adaptation, resubmit and set a 5-minute timer. If still Pending, try next rung on the ladder.
