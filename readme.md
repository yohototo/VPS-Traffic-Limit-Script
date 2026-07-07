Here is the complete, fully polished **English version** of the README. It is tailored for your fully automated, robust traffic-monitoring system on GCP Debian instances. You can save this as `README.md` for your future reference or GitHub repositories.

---

# GCP Debian Instance Outbound Traffic Limiter & Auto-Block Guide (Smart Adaptive Version)

This guide provides a comprehensive outbound traffic monitoring and network-kill solution tailored for **Debian** (and other mainstream Linux) VPS instances.

Since Google Cloud Platform (GCP) enforces strict data egress quotas with expensive overage fees, this solution leverages Linux kernel-level `iptables` to accurately track total data leaving your instance. When the cumulative outbound traffic within a 30-day billing cycle exceeds your defined threshold (default: **190 GB**), the system automatically triggers a network-kill protection, **blocking all outbound traffic except for SSH connections**. This completely eliminates the risk of unexpected cloud bill spikes.

## 🌟 Key Features of the Ultimate Version

1. **Smart Interface Auto-Detection**: The script dynamically detects the active primary network interface (e.g., `ens4` or `eth0`) using `ip route`. **No hardcoded interface names required; zero risk of selecting the wrong network card.**
2. **Absolute Path Resolution**: Core commands explicitly use absolute paths (e.g., `/sbin/iptables`), completely fixing the classic `command not found` errors triggered by the stripped-down environment of后台 `crontab` daemons.
3. **Robust Multi-Line Filtering**: Data extraction utilizes precise string matching coupled with `tail -n 1`, preventing script crashes and syntax errors (`integer expression expected`) caused by redundant firewall rules.
4. **Convenient CLI Alias**: Run `liuliang` from any directory to instantly check your current active network interface and precise bandwidth consumption (up to 4 decimal places).

---

## 🛠️ Step-by-Step Setup Guide

Log in to your GCP Linux instance as the **`root`** user and execute the following four steps in order:

### Step 1: Initialize System Locale (Fix Warnings)

To prevent Debian from throwing annoying `setlocale: LC_ALL: cannot change locale` warnings during execution, run the following commands to configure the system locale properly:

```bash
apt update && apt install -y locales
sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
export LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

```

### Step 2: Deploy the Smart Monitoring Script

Copy and paste the entire block below into your terminal. This automatically writes the script into the global executable directory, enables automatic interface discovery, and grants execution permissions:

```bash
cat << 'EOF' > /usr/local/bin/traffic_quota.sh
#!/bin/bash

# ==========================================
# Configuration Parameters
# ==========================================
QUOTA_GB=190  # Maximum allowed outbound traffic threshold (in GB)

# ==========================================
# ⚡ Smart Network Interface Auto-Detection
# ==========================================
# Dynamically fetch the primary interface handling default outbound routes
INTERFACE=$(/sbin/ip route show | grep default | awk '{print $5}' | tail -n 1)

if [ -z "$INTERFACE" ]; then
    echo "Error: Unable to auto-detect the public network interface."
    exit 1
fi

# ==========================================
# Automatic Firewall Chain Initialization
# ==========================================
/sbin/iptables -N TRAFFIC_MONITOR 2>/dev/null
if ! /sbin/iptables -C OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR 2>/dev/null; then
    /sbin/iptables -A OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR
fi

# ==========================================
# Core Traffic Aggregation & Logic
# ==========================================
# Extract byte counts cleanly, taking only the last line to ensure an exact integer output
CURRENT_BYTES=$(/sbin/iptables -L OUTPUT -v -n -x | grep "TRAFFIC_MONITOR" | awk '{print $2}' | tail -n 1)

if [ -z "$CURRENT_BYTES" ]; then
    echo "Error: Failed to fetch data from iptables counter."
    exit 1
fi

# Calculate current traffic in GB (formatted to 4 decimal places)
CURRENT_GB=$(awk -v bytes="$CURRENT_BYTES" 'BEGIN {printf "%.4f", bytes / 1024 / 1024 / 1024}')

# Print logs
echo "Interface: ${INTERFACE} | Current Outbound Traffic: ${CURRENT_GB} GB / Limit: ${QUOTA_GB} GB"

# Safe integer conversion for comparison using Bash internal arithmetic
QUOTA_BYTES=$(( QUOTA_GB * 1024 * 1024 * 1024 ))

# Network Kill Decision Logic
if [ "$CURRENT_BYTES" -gt "$QUOTA_BYTES" ]; then
    echo "WARNING: Traffic limit exceeded ${QUOTA_GB}GB! Activating network-kill protection..."
    
    # Check if protection rules are already in place to avoid duplicate rule appending
    if ! /sbin/iptables -C TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT 2>/dev/null; then
        # 1. Whitelist SSH Port 22 (Inbound & Outbound to ensure you never get disconnected)
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --sport 22 -j ACCEPT
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT
        
        # 2. DROP all other outbound packets traveling through the active public interface
        /sbin/iptables -A TRAFFIC_MONITOR -j DROP
        echo "Protection active. All non-SSH outbound traffic is now blocked."
    fi
fi
EOF

# Grant execution permissions
chmod +x /usr/local/bin/traffic_quota.sh

```

### Step 3: Configure the `liuliang` Shortcut Command

Inject a global shell alias into the system Bash profile so you can monitor your traffic by typing just 6 letters:

```bash
echo "alias liuliang='/usr/local/bin/traffic_quota.sh'" >> /etc/bash.bashrc
source /etc/bash.bashrc

```

### Step 4: Automate via Cron Daemons

1. Open the crontab scheduler configuration:
```bash
crontab -e

```


2. Paste the following two standard configurations at the **very bottom** of the file (ensure each rule occupies its own separate line with no stray symbols at the end):
```text
# Run traffic quota check every 5 minutes
*/5 * * * * /usr/local/bin/traffic_quota.sh >> /var/log/traffic_quota.log 2>&1

# Automatically reset/flush the traffic counter at midnight every 30 days
0 0 */30 * * /sbin/iptables -Z TRAFFIC_MONITOR >> /var/log/traffic_quota.log 2>&1

```


3. Save and exit the editor. When the terminal prompts `crontab: installing new crontab`, your automated firewall shield is officially operational.

---

## 📊 Everyday Management Commands

### 1. Check Real-time Traffic Consumption

Type the shortcut command from any directory path in the terminal:

```bash
liuliang

```

**Example Output:**

> Interface: ens4 | Current Outbound Traffic: 0.0695 GB / Limit: 190 GB

### 2. Audit Background Cron Automation Logs

Inspect the execution history to verify that the script is running properly every 5 minutes:

```bash
tail -n 10 /var/log/traffic_quota.log

```

### 3. How to Unblock the Network Manually?

If your network gets locked because you hit the traffic ceiling and you need to lift the ban immediately (or when a new billing cycle rolls over), execute these two quick commands to **flush the block rules and reset the data counter**:

```bash
# 1. Flush all DROP blocks inside the TRAFFIC_MONITOR chain
/sbin/iptables -F TRAFFIC_MONITOR

# 2. Zero-out the data counter back to 0 bytes
/sbin/iptables -Z TRAFFIC_MONITOR

```
