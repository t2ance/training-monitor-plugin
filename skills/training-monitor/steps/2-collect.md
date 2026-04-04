# Step 2: Collect Evidence

Run these checks for each job. Collect ALL data in parallel where possible.

## Local Jobs

### Process Status
```bash
ps -p <PID> -o pid,etime --no-headers 2>/dev/null || echo "DEAD"
```

### GPU Metrics
```bash
nvidia-smi --query-gpu=index,utilization.gpu,memory.used,power.draw --format=csv,noheader
```

### Recent Logs
```bash
tail -5 <log_file>
```

### Disk Space
```bash
df -h <checkpoint_dir> | tail -1
```

## Notes

- Run all commands in parallel when checking multiple jobs.
- For distributed training, load the `distributed-monitor` skill for multi-process PID management.
- For K8s jobs, load the `k8s-monitor` skill for kubectl-based collection.
- For W&B integration, the `wandb-primary` skill from `wandb/skills` is required. Use `monitor-doctor` to verify.
