# Staggered VolSync Backup Schedules

This guide explains how to stagger your VolSync backup schedules to prevent all backups from running at the same time (midnight), which can cause resource contention and potential cluster instability.

## Problem

By default, TrueCharts configures all VolSync ReplicationSources to run at midnight (`0 0 * * *`). When you have many applications using VolSync, this causes all backups to run simultaneously, which can lead to:

1. High resource usage spikes
2. Potential cluster instability
3. Timeouts or failures in backup operations

## Solution: Configure Schedules in Individual Helm Releases

The simplest solution is to configure custom schedules directly in each application's Helm release values. This approach:

- Keeps configuration with each application
- Doesn't require additional scripts or jobs
- Makes it easy to see and manage the schedule for each app

## Implementation Guide

### Step 1: Create a Schedule Plan

Here's a suggested schedule for your applications, spreading them throughout the day:

| Time (UTC) | Applications |
|------------|--------------|
| 00:00      | alist        |
| 01:00      | kavita       |
| 02:00      | mealie       |
| 03:00      | jellyfin     |
| 04:00      | audiobookshelf |
| 05:00      | freshrss     |
| 06:00      | lldap        |
| 07:00      | maintainerr  |
| 08:00      | minecraft    |
| 09:00      | minecraft-modded |
| 10:00      | notifiarr    |
| 11:00      | ntfy         |
| 12:00      | overseerr    |
| 13:00      | qbittorrent  |
| 14:00      | romm         |
| 15:00      | tautulli     |

### Step 2: Update Each Application's Helm Release

For each application, add a `schedule` field to the VolSync configuration in the Helm release:

```yaml
persistence:
  config:  # or data, depending on the app
    volsync:
      - credentials: ${VOLSYNC_BACKUP_MAIN_NAME}
        dest:
          enabled: true
        name: ${VOLSYNC_BACKUP_MAIN_NAME}
        src:
          enabled: true
          schedule: "0 X * * *"  # Replace X with the hour (0-23)
        type: restic
```

### Step 3: Verify the Schedules

After updating and applying the changes, verify that the schedules have been properly staggered:

```bash
kubectl get replicationsources -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,SCHEDULE:.spec.trigger.schedule
```

## Example Applications (Already Updated)

The following applications have already been updated with staggered schedules:

1. **alist**: `0 0 * * *` (Midnight)
2. **kavita**: `0 1 * * *` (1:00 AM)
3. **mealie**: `0 2 * * *` (2:00 AM)

## Additional Tips

- You can use more complex cron expressions if needed (e.g., `30 2 * * *` for 2:30 AM)
- Consider the resource usage of each backup when planning your schedule
- For very large backups, you might want to schedule them during periods of low cluster activity 