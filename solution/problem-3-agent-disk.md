# Problem 3 — Linux pipeline agent disk pressure

## Cause and risk

Root is full (99/100 GB); Docker uses 76 GB and the agent workspace 12 GB. The summary does not identify whether Docker usage is cache, images, containers, logs or volumes. This is a build agent, not an application host, but confirm artifacts are reproducible before removing anything.

## Investigate before deleting

Read-only checks:

```bash
df -h /; df -i /; findmnt /
sudo du -xhd1 /var/lib /var/log /opt/ado-agent 2>/dev/null | sort -h
sudo du -xhd1 /var/lib/docker /opt/ado-agent/_work 2>/dev/null | sort -h
sudo find /var/lib/docker /opt/ado-agent/_work -xdev -type f -size +1G \
  -printf '%s %TY-%Tm-%Td %p\n' 2>/dev/null | sort -nr | head -50
sudo docker system df -v
sudo docker ps -a --size; sudo docker image ls --digests; sudo docker volume ls
sudo docker builder du   # or: sudo docker buildx du, if Buildx is used
sudo journalctl --disk-usage; sudo lsof +L1
ps -ef | grep -E '[A]gent.Listener|[A]gent.Worker'
```

`df -i` checks inode exhaustion; `lsof +L1` finds deleted files still held open. If Docker is unhealthy, inspect rather than deleting its data root:

```bash
sudo systemctl status docker
sudo journalctl -u docker --since "24 hours ago" --no-pager
```

Before cleanup, check the Azure DevOps pool for active jobs and drain/disable the agent; inspect owners, timestamps, logs, configured Docker data root and mounted filesystems. Confirm images/cache can be rebuilt or pulled and volumes contain no unique data. Never assume all of `/var/lib/docker` or `_work` is disposable.

## Safe recovery

1. Drain the agent, let jobs finish or cancel through Azure DevOps. If needed, stop its service using `./svc.sh stop` from the agent directory; do not kill arbitrary processes.
2. Review cache usage and prune aged build cache interactively:

   ```bash
   sudo docker builder prune --filter "until=168h"
   ```

3. If still critically low, after confirming images can be pulled/rebuilt, review and prune unused images:

   ```bash
   sudo docker image prune -a
   ```

   Do not use `docker system prune --volumes` or manually remove Docker directories. Expect cache misses and image downloads.
4. Remove only confirmed stale agent job workspaces, using Azure DevOps maintenance controls where possible. Preserve needed logs and never blindly delete `_work`, `_tool`, or agent configuration.
5. Recheck `df -h /`, `df -i /`, and `docker system df -v`; restore the service, re-enable the agent, run a small build, and record reclaimed space. If still full, identify the consumer and plan a documented disk expansion/data-root migration; do not move live Docker data ad hoc.

## Prevention and verification

Alert on bytes and inodes before exhaustion (for example warn at 75%, act at 85%). Schedule bounded cache pruning and agent workspace maintenance; configure Docker log rotation and journal retention. Put Docker and work data on appropriately sized dedicated storage with a free-space buffer. Keep the agent dedicated to builds and retain necessary diagnostics.

Verify a test build after cleanup and monitor usage across several builds. These are proposed actions; no agent VM was accessed.
