# Problem 3 — Linux pipeline agent disk pressure

## Cause and risk

Root is full (99/100 GB); Docker uses 76 GB and the agent workspace 12 GB. The summary does not identify whether Docker usage is cache, images, containers, logs or volumes. This is a build agent, not an application host, but confirm artifacts are reproducible before removing anything.

## Investigate before deleting

Run these read-only checks to locate disk/inode usage and identify large consumers:

```bash
# Show root filesystem space, inode use, and mount backing /
df -h /
df -i /
findmnt /

# Summarize top-level usage on the same filesystem; sort smallest to largest
sudo du -xhd1 /var/lib /var/log /opt/ado-agent 2>/dev/null | sort -h

# Break down Docker and Azure DevOps workspace usage
sudo du -xhd1 /var/lib/docker /opt/ado-agent/_work 2>/dev/null | sort -h

# List the 50 largest files over 1 GB in Docker/work directories
sudo find /var/lib/docker /opt/ado-agent/_work -xdev -type f -size +1G \
  -printf '%s %TY-%Tm-%Td %p\n' 2>/dev/null | sort -nr | head -50

# Report Docker space by images, containers, and build cache
sudo docker system df -v

# List containers and their writable-layer sizes
sudo docker ps -a --size
# List local image digests, then volumes (do not assume volumes are disposable)
sudo docker image ls --digests
sudo docker volume ls

# Show classic builder cache usage; use `docker buildx du` for Buildx cache
sudo docker builder du

# Show journal size, then find deleted files still held open by processes
sudo journalctl --disk-usage
sudo lsof +L1

# Check whether Azure DevOps agent listener/worker processes are running
ps -ef | grep -E '[A]gent.Listener|[A]gent.Worker'
```

If Docker is unhealthy, inspect the service and recent logs rather than deleting its data root:

```bash
# Check Docker service state
sudo systemctl status docker
# Inspect Docker daemon logs from the last 24 hours
sudo journalctl -u docker --since "24 hours ago" --no-pager
```

Before cleanup, check the Azure DevOps pool for active jobs and drain/disable the agent; inspect owners, timestamps, logs, configured Docker data root and mounted filesystems. Confirm images/cache can be rebuilt or pulled and volumes contain no unique data. Never assume all of `/var/lib/docker` or `_work` is disposable.

## Safe recovery

1. Drain the agent, let jobs finish or cancel through Azure DevOps. If needed, stop its service using `./svc.sh stop` from the agent directory; do not kill arbitrary processes.
2. Review cache usage and prune cache unused for more than seven days. Docker prompts before removal; this can slow later builds but does not prune volumes:

   ```bash
   sudo docker builder prune --filter "until=168h"
   ```

3. If still critically low, confirm images can be pulled/rebuilt, then review and remove images not used by existing containers. The command prompts before deletion:

   ```bash
   sudo docker image prune -a
   ```

   Do not use `docker system prune --volumes` or manually remove Docker directories. Expect cache misses and image downloads.
4. Remove only confirmed stale agent job workspaces, using Azure DevOps maintenance controls where possible. Preserve needed logs and never blindly delete `_work`, `_tool`, or agent configuration.
5. Recheck with `df -h /` (free disk space), `df -i /` (free inodes), and `docker system df -v` (Docker usage); restore the service, re-enable the agent, run a small build, and record reclaimed space. If still full, identify the consumer and plan a documented disk expansion/data-root migration; do not move live Docker data ad hoc.

## Prevention

- Alert on filesystem bytes and inodes before exhaustion (for example, warn at 75% and act at 85%).
- Schedule bounded build-cache pruning and Azure DevOps workspace maintenance.
- Configure Docker log rotation and journal retention to limit log growth.
- Put Docker and work data on appropriately sized dedicated storage with a free-space buffer.
- Keep the agent dedicated to builds and retain the diagnostic logs needed for troubleshooting.

## Verification

- After cleanup, confirm free space and inodes with `df -h /` and `df -i /`, and Docker usage with `docker system df -v`.
- Run a small test build and confirm the agent returns to the Azure DevOps pool.
- Monitor filesystem use across several builds to confirm the cleanup/prevention measures are effective.

These are proposed actions; no agent VM was accessed.
