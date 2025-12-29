# **Mastering journalctl: The Ultimate Linux Logging Guide**
## **From Beginner to Log Analysis Expert**

---

## **Part 1: What is journald/journalctl?**

**Analogy:** Think of journald as a **universal flight recorder** for your Linux system:
- Every program, service, kernel message goes here
- **Structured** (not just text) - knows severity, timestamps, source
- **Centralized** - one place for ALL logs

### **Why journald vs Traditional Logs?**
```bash
# OLD WAY: Scattered files
ls /var/log/
# messages, secure, auth.log, httpd/, mysql/, kernel.log...

# NEW WAY: One command
journalctl  # Everything in one place
```

---

## **Part 2: journalctl Basics - Start Here**

### **The 5 Essential Commands**
```bash
# 1. View ALL logs (live)
journalctl

# 2. View logs in REAL-TIME (like tail -f)
journalctl -f
# Press Ctrl+C to stop

# 3. View logs for specific service
journalctl -u nginx          # nginx service
journalctl -u sshd           # ssh service
journalctl -u docker         # docker service

# 4. View logs since boot
journalctl -b                # Current boot
journalctl -b -1             # Previous boot
journalctl -b -2             # Boot before that

# 5. View kernel messages only
journalctl -k                # Like dmesg but better
```

### **Understanding Output Format**
```
Dec 15 10:30:45 server sshd[1234]: Accepted password for user from 192.168.1.100
```
Translation:
- `Dec 15 10:30:45` = Timestamp
- `server` = Hostname
- `sshd[1234]` = Process (sshd) with PID 1234
- `Accepted password...` = The actual message

---

## **Part 3: Filtering - Find What You Need**

### **Time-Based Filtering (CRITICAL for troubleshooting)**
```bash
# Relative time
journalctl --since "1 hour ago"
journalctl --since "30 min ago"
journalctl --since "yesterday"
journalctl --since "2 days ago"

# Absolute time
journalctl --since "2024-12-15 09:00:00"
journalctl --since "2024-12-15" --until "2024-12-16"
journalctl --since "09:00" --until "10:00"  # Today

# Today's logs
journalctl --since today
journalctl --since 00:00

# Last 10 minutes
journalctl --since -10m
```

### **Priority/Level Filtering (Most Important!)**
```bash
# Show ONLY errors (priority 3 and above)
journalctl -p err
journalctl -p error          # Same as above

# Show specific levels (0 = emerg, 7 = debug)
journalctl -p 0..3           # Emergency, Alert, Critical, Error
journalctl -p 4              # Warning only
journalctl -p 5              # Notice only
journalctl -p 6              # Info only
journalctl -p 7              # Debug only

# Priority levels (remember this!):
# 0: emerg    - System is unusable
# 1: alert    - Action must be taken immediately
# 2: crit     - Critical conditions
# 3: err      - Error conditions
# 4: warning  - Warning conditions
# 5: notice   - Normal but significant
# 6: info     - Informational messages
# 7: debug    - Debug-level messages
```

### **Unit/Service Filtering**
```bash
# Single service
journalctl -u nginx

# Multiple services
journalctl -u nginx -u mysql

# Service + time
journalctl -u sshd --since "1 hour ago"

# All services of a type
journalctl _SYSTEMD_UNIT=nginx.service
```

### **PID/Process Filtering**
```bash
# Show logs from specific PID
journalctl _PID=1234

# Find PID first, then check logs
pidof nginx
journalctl _PID=$(pidof nginx)

# Show logs from specific executable
journalctl /usr/sbin/sshd
```

---

## **Part 4: Output Formatting - Make Logs Readable**

### **Different Output Formats**
```bash
# Short (default) - good for quick look
journalctl -u nginx

# Verbose - show ALL fields
journalctl -u nginx -o verbose
# Shows: MESSAGE_ID, PRIORITY, SYSLOG_FACILITY, etc.

# JSON - for scripting/automation
journalctl -u nginx -o json
journalctl -u nginx -o json-pretty

# Export format
journalctl -u nginx -o export > logs.bin

# Show only message text (no timestamps)
journalctl -u nginx -o cat

# Table view (compact)
journalctl -u nginx -o short-full

# With colors for priority
journalctl -u nginx --output=short-full --colors
```

### **Show Specific Fields Only**
```bash
# Show only timestamp and message
journalctl -o json --output-fields=MESSAGE,_SOURCE_REALTIME_TIMESTAMP | jq .

# Show in custom format
journalctl -o json-pretty | jq '.[] | {timestamp: ._SOURCE_REALTIME_TIMESTAMP, message: .MESSAGE}'
```

---

## **Part 5: Advanced Filtering - Become a Power User**

### **Combining Multiple Filters**
```bash
# Show errors from nginx in last hour
journalctl -u nginx -p err --since "1 hour ago"

# Show kernel warnings since yesterday
journalctl -k -p warning --since yesterday

# SSH failures in last 30 minutes
journalctl -u sshd -p err --since "30 min ago" | grep -i "fail"
```

### **Using Field-Based Filtering**
```bash
# Show logs from specific user
journalctl _UID=1000

# Show logs from specific group
journalctl _GID=1000

# Show logs with specific hostname
journalctl _HOSTNAME=webserver1

# Show logs from specific command
journalctl _COMM=sshd

# Show kernel subsystem messages
journalctl _TRANSPORT=kernel
```

### **Finding Specific Patterns**
```bash
# Case-insensitive grep
journalctl -u nginx | grep -i error

# Show lines before/after match
journalctl -u mysql | grep -B2 -A2 "connection"

# Regular expressions
journalctl -u sshd --grep="Failed password"

# Multiple grep patterns
journalctl -u apache | grep -E "(error|crit|alert)"
```

---

## **Part 6: Real-World Troubleshooting Scenarios**

### **Scenario 1: "My website is down!"**
```bash
# 1. Check nginx logs for errors
journalctl -u nginx -p err --since "5 min ago"

# 2. If no errors, check info level
journalctl -u nginx --since "5 min ago"

# 3. Check if nginx is actually running
systemctl status nginx

# 4. Check kernel messages for OOM killer
journalctl -k --since "10 min ago" | grep -i "killed"
```

### **Scenario 2: "SSH keeps disconnecting"**
```bash
# 1. Look for authentication issues
journalctl -u sshd -p warning --since "1 hour ago"

# 2. Look for connection refusals
journalctl -u sshd | grep -i "refused"

# 3. Check for brute force attacks
journalctl -u sshd --since today | grep "Failed password" | wc -l

# 4. Monitor real-time SSH attempts
journalctl -u sshd -f | grep --color "Failed password"
```

### **Scenario 3: "System is slow"**
```bash
# 1. Check for disk I/O errors
journalctl -k | grep -i "I/O error"

# 2. Check for memory pressure
journalctl -k | grep -i "oom"

# 3. Check for service restarts (crashing)
journalctl --since "1 hour ago" | grep "Stopped\|Started"

# 4. Check systemd for failed units
systemctl --failed
journalctl -u $(systemctl --failed --no-legend | awk '{print $1}')
```

### **Scenario 4: "Something crashed at 2 AM"**
```bash
# 1. Find all critical+ messages from that time
journalctl -p crit --since "02:00" --until "02:10"

# 2. Check kernel panic/OOPS
journalctl -k --since "02:00" --until "02:10"

# 3. List all units that failed
journalctl --since "02:00" --until "02:10" | grep "failed\|error"

# 4. Export logs for that period for later analysis
journalctl --since "02:00" --until "02:10" --output=json > crash_logs.json
```

---

## **Part 7: Log Management & Maintenance**

### **Checking Disk Usage**
```bash
# How much space are logs using?
journalctl --disk-usage
# Output: Archived and active journals take up 1.2G on disk.

# Detailed breakdown
journalctl --vacuum-size=500M --dry-run

# List all journal files
ls -lh /var/log/journal/
```

### **Cleaning Up Old Logs**
```bash
# Keep only last 2 days
journalctl --vacuum-time=2d

# Keep only 500MB of logs
journalctl --vacuum-size=500M

# Keep only 10 journal files
journalctl --vacuum-files=10

# Do all three
journalctl --vacuum-time=7d --vacuum-size=500M --vacuum-files=10

# Aggressive cleanup (emergency)
journalctl --rotate       # Rotate current journal
journalctl --vacuum-time=1s  # Keep almost nothing
```

### **Permanent Configuration**
Edit `/etc/systemd/journald.conf`:
```ini
[Journal]
# Store logs on disk (persistent after reboot)
Storage=persistent

# Or keep in memory only (volatile)
# Storage=volatile

# Maximum disk usage
SystemMaxUse=1G
SystemKeepFree=10%  # Leave 10% disk free

# Maximum age of logs
MaxRetentionSec=1month

# Compress old logs
Compress=yes
```

---

## **Part 8: Boot & System Analysis**

### **Analyzing Boot Process**
```bash
# Show boot time for each service
systemd-analyze blame

# Show critical chain
systemd-analyze critical-chain

# Show boot chart
systemd-analyze plot > boot.svg

# Show time in kernel vs userspace
systemd-analyze
```

### **Working with Multiple Boots**
```bash
# List all boots
journalctl --list-boots
# Output shows: -2 abc123..., -1 def456..., 0 ghi789...

# Show previous boot logs
journalctl -b -1

# Show logs from specific boot
journalctl -b abc123def

# Follow logs from current boot only
journalctl -b -f
```

### **Kernel Message Analysis**
```bash
# All kernel messages
journalctl -k

# Kernel messages with timestamps
journalctl -k -o short-full

# Filter kernel messages by facility
journalctl -k SYSLOG_FACILITY=0

# Watch kernel messages in real-time
journalctl -k -f
```

---

## **Part 9: Scripting & Automation with journalctl**

### **Extracting Data for Monitoring**
```bash
# Count errors in last hour
journalctl -p err --since "1 hour ago" | wc -l

# List unique services with errors
journalctl -p err --since today -o json | jq -r '._SYSTEMD_UNIT' | sort | uniq

# Create error report
journalctl -p err --since "24 hours ago" -o short-full > /tmp/error_report.txt

# Monitor specific error patterns
while true; do
    count=$(journalctl -u mysql --since "5 min ago" | grep -c "deadlock")
    if [ $count -gt 0 ]; then
        echo "ALERT: MySQL deadlocks detected: $count"
    fi
    sleep 300
done
```

### **JSON Output for Automation**
```bash
# Parse with jq for specific fields
journalctl -u nginx --since "1 hour ago" -o json | \
    jq '.[] | select(.PRIORITY <= 3) | {timestamp: ._SOURCE_REALTIME_TIMESTAMP, message: .MESSAGE}'

# Create CSV report
journalctl -u apache --since today -o json | \
    jq -r '.[] | [._SOURCE_REALTIME_TIMESTAMP, .PRIORITY, .MESSAGE] | @csv' > apache_logs.csv
```

### **Creating Custom Log Views**
```bash
# Create alias for common queries
alias jerr='journalctl -p err --since "1 hour ago"'
alias jnginx='journalctl -u nginx -f'
alias jboot='journalctl -b -0 --no-pager'

# Function to find errors between times
function jbetween() {
    journalctl --since "$1" --until "$2" -p err
}
# Usage: jbetween "09:00" "10:00"
```

---

## **Part 10: Practice Exercises**

### **Exercise 1: Find Failed SSH Attempts**
```bash
# Count failed SSH attempts in last hour
journalctl -u sshd --since "1 hour ago" | grep -c "Failed password"

# List IP addresses of failed attempts
journalctl -u sshd --since today | grep "Failed password" | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" | sort | uniq
```

### **Exercise 2: Create Daily Error Report**
```bash
#!/bin/bash
# daily-error-report.sh
DATE=$(date +%Y-%m-%d)
REPORT="/var/log/error_reports/errors_$DATE.txt"

echo "=== System Error Report for $DATE ===" > $REPORT
echo "" >> $REPORT

# Count errors by priority
echo "Error Count by Priority:" >> $REPORT
echo "Emerg: $(journalctl -p 0 --since today | wc -l)" >> $REPORT
echo "Alert: $(journalctl -p 1 --since today | wc -l)" >> $REPORT
echo "Crit:  $(journalctl -p 2 --since today | wc -l)" >> $REPORT
echo "Error: $(journalctl -p 3 --since today | wc -l)" >> $REPORT

# Top 5 services with errors
echo "" >> $REPORT
echo "Top 5 Services with Errors:" >> $REPORT
journalctl -p err --since today -o json | \
    jq -r '._SYSTEMD_UNIT' | sort | uniq -c | sort -rn | head -5 >> $REPORT
```

### **Exercise 3: Monitor Service Health**
```bash
#!/bin/bash
# service-health-check.sh
SERVICES="nginx mysql sshd"

for service in $SERVICES; do
    echo "=== Checking $service ==="
    
    # Check if service is running
    if systemctl is-active --quiet $service; then
        echo "✓ Service is running"
        
        # Check for recent errors
        ERROR_COUNT=$(journalctl -u $service -p err --since "10 min ago" | wc -l)
        if [ $ERROR_COUNT -gt 0 ]; then
            echo "⚠  $ERROR_COUNT errors in last 10 minutes:"
            journalctl -u $service -p err --since "10 min ago" --no-pager | tail -5
        else
            echo "✓ No recent errors"
        fi
    else
        echo "✗ Service is NOT running!"
        echo "Last logs:"
        journalctl -u $service --no-pager | tail -10
    fi
    echo ""
done
```

---

## **Part 11: Common Problems & Solutions**

### **Problem 1: "journalctl shows no logs"**
```bash
# Check if journald is running
systemctl status systemd-journald

# Check storage mode
cat /etc/systemd/journald.conf | grep Storage
# Should be "persistent" or "auto"

# Check disk space
df -h /var/log/

# Force journal rotation
journalctl --rotate
```

### **Problem 2: "Logs are too big"**
```bash
# Check current usage
journalctl --disk-usage

# Reduce retention
sudo tee -a /etc/systemd/journald.conf << EOF
SystemMaxUse=500M
RuntimeMaxUse=100M
MaxRetentionSec=1week
EOF

# Apply changes
sudo systemctl restart systemd-journald
```

### **Problem 3: "Can't find old boot logs"**
```bash
# Check if persistent storage is enabled
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal

# Restart journald
sudo systemctl restart systemd-journald

# Now old boots should appear
journalctl --list-boots
```

---

## **Part 12: Quick Reference Cheat Sheet**

### **Most Used Commands**
```bash
# Daily use
journalctl -f                         # Follow logs
journalctl -u service --since "10 min ago"
journalctl -p err                     # Only errors
journalctl -b -1                      # Previous boot

# Investigation
journalctl --since "yesterday" --until "today"
journalctl -k                         # Kernel logs
journalctl _PID=1234                  # Specific PID

# Maintenance
journalctl --disk-usage               # Check size
journalctl --vacuum-size=500M         # Cleanup
```

### **Priority Quick Guide**
```bash
0: emerg    - 🔴 SYSTEM UNUSABLE
1: alert    - 🔴 TAKE ACTION NOW
2: crit     - 🔴 CRITICAL ERROR
3: err      - ⚠️  REGULAR ERROR
4: warning  - ⚠️  WARNING
5: notice   - ℹ️  NOTICE
6: info     - ℹ️  INFO (default)
7: debug    - 🔧 DEBUG
```

### **Pro Tips**
1. **Always use `--since`** - Don't scroll through all logs
2. **Combine filters** - `-u service -p err --since`
3. **Use `-f` for real-time** monitoring
4. **Check previous boots** when troubleshooting crashes
5. **Export to JSON** for analysis with tools

---

## **Test Your Skills**

1. **Find all errors from MySQL in the last 30 minutes:**
   ```bash
   journalctl -u mysql -p err --since "30 min ago"
   ```

2. **Watch SSH login attempts in real-time:**
   ```bash
   journalctl -u sshd -f | grep --color "Accepted\|Failed"
   ```

3. **Create a report of yesterday's warnings:**
   ```bash
   journalctl -p warning --since yesterday --until today > warnings.txt
   ```

4. **Find why the system crashed at 3 AM:**
   ```bash
   journalctl -p crit..err --since "03:00" --until "03:05"
   journalctl -k --since "03:00" --until "03:05"
   ```

5. **Count unique services that logged errors today:**
   ```bash
   journalctl -p err --since today -o json | jq -r '._SYSTEMD_UNIT' | sort | uniq | wc -l
   ```

