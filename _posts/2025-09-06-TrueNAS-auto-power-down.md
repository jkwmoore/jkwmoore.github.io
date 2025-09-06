---
layout: post
title: "Scripting automatic power down for idle TrueNAS instances"
category: TrueNAS
---

Chances are that if you run one TrueNAS instance, you run a backup TrueNAS instance that you don't actually need powered on all the time.

I'll leave any automatic power up scheduling and replication job configuration to the reader. You can fill your boots with Wake-on-LAN, iDRAC, iLO API prodding etc...

I approached the automatic power down problem as follows:

1. Write a bash script which will power the system down only when no-one is logged in, no replication tasks are running and not enough online time has passed for scheduled replications to start.
2. Add a matching CRON job.

### Script

```bash
#!/bin/bash

echo "Checking if Admin session exists"
if who | grep '^truenas_admin' | grep -vq '192\.168\.0\.22[1-3]$'; then
    # truenas_admin is logged in, but not from 192.168.0.221–223 (aka my TrueCommand instance)
    echo "truenas_admin currently has a web or SSH session (not from 192.168.0.221–223 aka the TrueCommand instance). Skipping power off."
    exit 0
fi

echo "Checking if replications are active."

# Check if any replication task is running
if midclt call replication.query | jq -r '.[].state.state' | grep -q '^RUNNING$'; then
  # Replication running; don't power down
  echo "Replications currently active, not shutting down."
  exit 0
fi

# Check uptime in seconds
echo "No replications active, checking uptime."
uptime_seconds=$(cut -d' ' -f1 /proc/uptime | cut -d'.' -f1)

if [ "$uptime_seconds" -gt 600 ]; then
  logger "No replication running and uptime > 600s, powering off system."
  echo  "No replication running and uptime > 600s, powering off system."
  systemctl poweroff
else
  echo "Uptime lower than 600 seconds, skipping power off."
fi
```

Note: If you don't have a TrueCommand instance you need to ignore, you should edit the logic in the first check block.

### Setup

1. Add the script above to your local file system e.g. ``/root/check_replication_and_poweroff.sh``
2. Add a CRON job.via the ``System --> Advanced Settings --> Cron Jobs`` which points at where you saved the script above. Choose to run as root, set your schedule (e.g. ``*/1 * * * *``), make sure to hide standard output and enable the job.
3. Do some testing to make sure this works how you need it to.
   
