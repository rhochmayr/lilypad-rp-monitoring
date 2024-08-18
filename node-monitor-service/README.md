# Node Monitoring Service with Automatic Service Restarts

This guide contains a script and systemd service to monitor the online status of a Lilypad resource provider node and check for PoW signals. The service runs every 5 minutes and restarts the services `bacalhau` and `lilypad-resource-provider` if the node appears to be offline or no PoW signals have been detected for the node within the last hour.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup](#setup)
  - [1. Create the Timer Unit File](#1-create-the-timer-unit-file)
  - [2. Create the Service Unit File](#2-create-the-service-unit-file)
  - [3. Create the Service User](#3-create-the-service-user)
  - [4. Create the Monitoring Script](#4-create-the-monitoring-script)
  - [5. Enable and Start the Timer](#5-enable-and-start-the-timer)
- [Conclusion](#conclusion) 


## Prerequisites

- A Linux-based system with `systemd`
- `curl` and `jq` installed
- Proper permissions to create and manage systemd service and timer units

## Setup

### 1. Create the Timer Unit File

Create a timer unit file at `/etc/systemd/system/node-monitor.timer` with the following content:

```sh
sudo vi /etc/systemd/system/node-monitor.timer
```

```ini
[Unit]
Description=Run Node Monitor Service every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=node-monitor.service

[Install]
WantedBy=timers.target
```

### 2. Create the Service Unit File

Create a service unit file at `/etc/systemd/system/node-monitor.service` with the following content:

```sh
sudo vi /etc/systemd/system/node-monitor.service
```

```ini
[Unit]
Description=Node Monitor Service for Online Status
After=network.target

[Service]
Environment="NODE_WALLET_ID=<WALLET_ID>"
Environment="TESTING_MODE=true"
ExecStart=/usr/local/bin/online-status-check.sh
Type=simple
User=your_service_user

[Install]
WantedBy=multi-user.target
```
Hint: replace `NODE_WALLET_ID` with your node's wallet ID and `your_service_user` with the user that will run the service (eg. `node_monitor`). By default the service runs in testing mode, which means it will only perform the checks but won't restart the services. To enable the restarts, set `TESTING_MODE` to `false`.

### 3. Create the Service User

First, edit the `/etc/sudoers` file to allow the service user to restart the specific services without a password. You can use the visudo command to safely edit the sudoers file:

```sh
sudo visudo
```

Add the following line to grant the necessary permissions (replace `your_service_user` with the actual user running the service):


```ini
your_service_user ALL=(ALL) NOPASSWD: /bin/systemctl restart bacalhau, /bin/systemctl restart lilypad-resource-provider
```

Then, create the service user:

```bash
sudo useradd -r -s /bin/false your_service_user
```

### 4. Create the Monitoring Script

Ensure you have the `online-status-check.sh` script in the same directory as the service unit file. The script should contain the following content:

```sh
sudo vi /usr/local/bin/online-status-check.sh
```

```bash
#!/bin/bash

# Configuration
ONLINE_STATUS_URL="https://api-testnet.lilypad.tech/metrics-dashboard/nodes"
POW_SIGNAL_URL="https://api-sepolia.arbiscan.io/api?module=account&action=txlist&address=$NODE_WALLET_ID&page=1&offset=1000&sort=desc"
COOLDOWN_FILE="/tmp/node-monitor-cooldown"
COOLDOWN_PERIOD=3600  # 1 hour

# Function to check online status
check_online_status() {
    echo "Checking online status..."
    response=$(curl -s $ONLINE_STATUS_URL)
    online_status=$(echo $response | jq -r '.[] | select(.ID == "'$NODE_WALLET_ID'").Online')
    echo "Online status: $online_status"
    if [ "$online_status" != "true" ]; then
        echo "Node is offline."
        return 1
    fi
    echo "Node is online."
    return 0
}

# Function to check PoW signals
check_pow_signals() {
    echo "Checking PoW signals..."
    response=$(curl -s $POW_SIGNAL_URL)
    now=$(date +%s)
    one_hour_ago=$((now - 3600))
    transactions=$(echo $response | jq -r '.result')
    filtered_results=$(echo $transactions | jq -r '[.[] | select(.methodId == "0xda8accf9" and (.timeStamp | tonumber) >= '$one_hour_ago')]')
    signal_count=$(echo $filtered_results | jq -r 'length')
    echo "Number of PoW signals in the last hour: $signal_count"
    if [ "$signal_count" -lt 1 ]; then
        echo "No PoW signals found."
        return 1
    fi
    echo "PoW signals found."
    return 0
}

# Function to restart services with cooldown for PoW signals
restart_services() {
    if [ "$1" == "pow" ]; then
        if [ -f "$COOLDOWN_FILE" ]; then
            last_restart=$(cat $COOLDOWN_FILE)
            now=$(date +%s)
            if [ $((now - last_restart)) -lt $COOLDOWN_PERIOD ]; then
                echo "Cooldown period active. Skipping restart for PoW signals."
                return 0
            fi
        fi
        date +%s > $COOLDOWN_FILE
    fi

    echo "Restarting services..."
    if [ "$TESTING_MODE" == "true" ]; then
        echo "Restarting services in testing mode. No actual restarts performed."
    else
        echo "restarting bacalhau."
        sudo systemctl restart bacalhau

        echo "restarting lilypad-resource-provider."
        sudo systemctl restart lilypad-resource-provider     
    fi
}

# Main logic
echo "Starting node monitor service..."
if ! check_online_status; then
    restart_services "online"
elif ! check_pow_signals; then
    restart_services "pow"
fi
echo "Node monitor service completed."
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/online-status-check.sh
```

### 5. Enable and Start the Timer

Enable and start the timer to ensure it runs every 5 minutes:

```bash
sudo systemctl enable node-monitor.timer
sudo systemctl start node-monitor.timer
```

## Conclusion
With these steps, the `node-monitor.service` will be triggered every 5 minutes by the `node-monitor.timer`, ensuring that the checks are performed at the desired interval.


