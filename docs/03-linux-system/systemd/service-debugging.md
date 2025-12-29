# **Systemd Service Debugging: Step-by-Step Real-World Guide**
## **From "It's broken!" to "Fixed it!" - Complete Walkthrough**

---

## **Part 1: The Debugging Mindset**

### **The 80/20 Rule of Service Debugging**
80% of problems are found with these steps:
1. Check service status
2. Read the logs
3. Check dependencies
4. Test manually

20% require deep diving:
5. Check permissions
6. Verify configuration
7. Test environment
8. Debug with strace

---

## **Part 2: Our Case Study - "webapp" Service**

Let's debug a real (simulated) service that's failing. We'll create it, break it, and fix it step by step.

### **Step 0: Create Our Test Service**
```bash
# Create a simple Python web app that will fail
sudo tee /usr/local/bin/webapp.py << 'EOF'
#!/usr/bin/env python3
import time
import sys
import os
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b'WebApp is working!\n')
    
    def log_message(self, format, *args):
        # Disable default logging
        pass

def main():
    # Simulate a common bug: wrong port or permissions
    port = 8080
    
    # Check if port is available
    import socket
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        sock.bind(('127.0.0.1', port))
        sock.close()
    except:
        print(f"ERROR: Port {port} already in use or no permission")
        sys.exit(1)
    
    # Check if we can write to log directory
    log_dir = '/var/log/webapp'
    if not os.path.exists(log_dir):
        print(f"ERROR: Log directory {log_dir} doesn't exist")
        sys.exit(1)
    
    server = HTTPServer(('127.0.0.1', port), Handler)
    print(f"Starting webapp on port {port}")
    server.serve_forever()

if __name__ == '__main__':
    main()
EOF

# Make it executable
sudo chmod +x /usr/local/bin/webapp.py

# Create service file with INTENTIONAL BUGS
sudo tee /etc/systemd/system/webapp.service << 'EOF'
[Unit]
Description=Web Application Service
After=network.target
Requires=network.target

[Service]
Type=simple
User=nobody  # Limited user (common cause of issues)
Group=nogroup
WorkingDirectory=/tmp
ExecStart=/usr/local/bin/webapp.py
Restart=on-failure
RestartSec=5

# Environment variables
Environment="PORT=8080"
Environment="LOG_DIR=/var/log/webapp"

# Security
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

---

## **Part 3: The Debugging Process - Step by Step**

### **Step 1: Start the Service & See It Fail**
```bash
# Reload systemd to recognize new service
sudo systemctl daemon-reload

# Try to start it (it will fail!)
sudo systemctl start webapp

# Check status (FIRST COMMAND ALWAYS!)
sudo systemctl status webapp
```
**Expected output shows:** `failed` with red text

### **Step 2: Read the Logs (journalctl)**
```bash
# Show ALL logs for our service
sudo journalctl -u webapp

# More detailed with timestamps
sudo journalctl -u webapp -o short-full

# Just the last 10 lines
sudo journalctl -u webapp -n 10

# Follow logs in real-time (start service again)
sudo systemctl restart webapp
sudo journalctl -u webapp -f
```

**What you'll likely see:**
```
Dec 15 11:30:00 server webapp.py[1234]: ERROR: Log directory /var/log/webapp doesn't exist
```

### **Step 3: Fix the Obvious Issue**
```bash
# Create the missing directory
sudo mkdir -p /var/log/webapp

# Try again
sudo systemctl restart webapp
sudo systemctl status webapp
```

**Still failing?** Check logs again:
```bash
sudo journalctl -u webapp --since "1 min ago"
```
Now you might see: `ERROR: Port 8080 already in use or no permission`

### **Step 4: Check for Port Conflicts**
```bash
# What's using port 8080?
sudo ss -tulpn | grep :8080
# Or
sudo lsof -i :8080

# If something is using it, stop it or change our port
# Let's check if we have permission as 'nobody' user
sudo -u nobody /usr/local/bin/webapp.py
```

### **Step 5: Test the Service Manually**
```bash
# Stop the systemd service first
sudo systemctl stop webapp

# Test as current user (root)
/usr/local/bin/webapp.py &
curl http://localhost:8080
kill %1  # Kill background job

# Test as the service user
sudo -u nobody /usr/local/bin/webapp.py &
# This will likely fail - permission issues!
```

### **Step 6: Permission Debugging**
```bash
# Check what user 'nobody' can do
sudo -u nobody id
sudo -u nobody whoami

# Check file permissions
ls -la /usr/local/bin/webapp.py
ls -la /var/log/webapp/

# Test if 'nobody' can bind to port 8080
sudo -u nobody python3 -c "import socket; s=socket.socket(); s.bind(('127.0.0.1', 8080)); print('OK')"
```

### **Step 7: Fix Permissions & Configuration**
```bash
# Give 'nobody' write access to log directory
sudo chown nobody:nogroup /var/log/webapp/

# Use a higher port (above 1024) that doesn't need root
sudo sed -i 's/8080/18080/' /usr/local/bin/webapp.py

# Update service file to use different user or capabilities
sudo tee /etc/systemd/system/webapp.service << 'EOF'
[Unit]
Description=Web Application Service
After=network.target

[Service]
Type=simple
User=webappuser  # Create dedicated user
Group=webappgroup
WorkingDirectory=/var/lib/webapp
ExecStart=/usr/local/bin/webapp.py
Restart=on-failure
RestartSec=5

# Logging
StandardOutput=journal
StandardError=journal

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/log/webapp

[Install]
WantedBy=multi-user.target
EOF

# Create dedicated user and directories
sudo groupadd webappgroup
sudo useradd -r -s /bin/false -g webappgroup webappuser
sudo mkdir -p /var/lib/webapp /var/log/webapp
sudo chown webappuser:webappgroup /var/lib/webapp /var/log/webapp
```

### **Step 8: Apply Fixes & Test Again**
```bash
# Reload and restart
sudo systemctl daemon-reload
sudo systemctl restart webapp
sudo systemctl status webapp

# Check logs
sudo journalctl -u webapp --since "1 min ago"

# Test the endpoint
curl http://localhost:18080
```

---

## **Part 4: Advanced Debugging Techniques**

### **Technique 1: Debug Mode with Extra Logging**
```bash
# Create a debug version of service
sudo systemctl edit webapp
```
Add:
```ini
[Service]
Environment=DEBUG=1
ExecStartPre=/bin/echo "Starting webapp with DEBUG mode"
ExecStartPost=/bin/echo "Webapp started at $(date)"
```

### **Technique 2: Use strace to See System Calls**
```bash
# Find the PID
sudo systemctl status webapp | grep PID
# Or
sudo pgrep -f webapp

# Attach strace (if service is hanging)
sudo strace -p <PID> -f

# Or run service with strace from start
sudo systemctl stop webapp
sudo strace -f -o /tmp/webapp-strace.log /usr/local/bin/webapp.py
```

### **Technique 3: Check Dependencies**
```bash
# What does webapp depend on?
systemctl list-dependencies webapp

# What depends on webapp?
systemctl list-dependencies webapp --reverse

# Show startup order
systemd-analyze critical-chain webapp
```

### **Technique 4: Test with systemd-run (Sandbox)**
```bash
# Run service temporarily without installing
sudo systemd-run --unit=test-webapp \
    --property="User=webappuser" \
    --property="Group=webappgroup" \
    --property="WorkingDirectory=/var/lib/webapp" \
    /usr/local/bin/webapp.py

# Check status
systemctl status test-webapp

# See logs
journalctl -u test-webapp

# Clean up
systemctl stop test-webapp
systemctl reset-failed test-webapp
```

### **Technique 5: Check Security Context (SELinux)**
```bash
# If SELinux is enabled
getenforce  # Should show Enforcing, Permissive, or Disabled

# Check SELinux denials
sudo ausearch -m avc -ts recent

# Check file contexts
ls -laZ /usr/local/bin/webapp.py

# If SELinux issues, check audit logs
sudo journalctl -t setroubleshoot
```

---

## **Part 5: Common Service Problems & Solutions**

### **Problem 1: Service Starts Then Immediately Stops**
```bash
# Check service type
systemctl cat webapp | grep Type=

# If Type=oneshot, it's supposed to exit
# If Type=simple or forking, it should stay running

# Check Restart policy
systemctl show webapp | grep Restart

# Look for exit code
journalctl -u webapp -o json | jq 'select(.MESSAGE | contains("exited"))'
```

### **Problem 2: Service Hangs on Startup**
```bash
# Check timeout
systemctl show webapp | grep Timeout

# Increase timeout
sudo systemctl edit webapp
```
Add:
```ini
[Service]
TimeoutStartSec=300
```

### **Problem 3: Permission Denied Errors**
```bash
# Check all file permissions
namei -l /usr/local/bin/webapp.py
namei -l /var/log/webapp/

# Check capability requirements
# If binding to port < 1024:
sudo systemctl edit webapp
```
Add:
```ini
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### **Problem 4: Environment Variables Not Set**
```bash
# Check environment
systemctl show webapp | grep Environment

# Test with printenv
sudo systemctl edit webapp
```
Temporarily add:
```ini
[Service]
ExecStartPre=/usr/bin/printenv
```

### **Problem 5: Can't Write to Logs**
```bash
# Check StandardOutput setting
systemctl show webapp | grep StandardOutput

# Options:
# StandardOutput=journal (to journald)
# StandardOutput=file:/var/log/webapp/output.log
# StandardOutput=append:/var/log/webapp/output.log

# Or fix directory permissions
sudo mkdir -p /var/log/webapp
sudo chown webappuser:webappgroup /var/log/webapp
sudo chmod 755 /var/log/webapp
```

---

## **Part 6: Real-World Debugging Scenario**

Let's debug a complex issue: **Service works manually but not via systemd**

### **The Symptoms:**
```bash
# Manual test works
sudo -u webappuser /usr/local/bin/webapp.py &
curl http://localhost:18080  # Success!
kill %1

# Systemd fails
sudo systemctl start webapp
sudo systemctl status webapp  # Shows failed
journalctl -u webapp  # Shows cryptic error
```

### **Debugging Steps:**
```bash
# 1. Compare environments
echo "=== Manual Environment ==="
sudo -u webappuser env | sort

echo "=== Systemd Environment ==="
systemctl show webapp | grep Environment

# 2. Check working directory
sudo -u webappuser pwd
systemctl show webapp | grep WorkingDirectory

# 3. Check resource limits
sudo -u webappuser ulimit -a
systemctl show webapp | grep Limit

# 4. Check if it's a PATH issue
sudo systemctl edit webapp
```
Add debug:
```ini
[Service]
ExecStartPre=/usr/bin/env
ExecStartPre=/bin/pwd
ExecStartPre=/usr/bin/ulimit -a
```

### **Common Culprits Found:**
1. **Different PATH** - systemd has minimal PATH
2. **Missing environment variables**
3. **Resource limits** (ulimit)
4. **Working directory permissions**
5. **Missing shared libraries**

---

## **Part 7: Creating a Debugging Checklist**

### **The 10-Step Debugging Checklist**
```bash
#!/bin/bash
# debug-service.sh - Run this when a service fails

SERVICE=${1:-webapp}

echo "=== Debugging $SERVICE ==="
echo ""

echo "1. Service status:"
systemctl status $SERVICE --no-pager
echo ""

echo "2. Recent logs:"
journalctl -u $SERVICE -n 20 --no-pager
echo ""

echo "3. Failed units:"
systemctl --failed --no-pager
echo ""

echo "4. Dependencies:"
systemctl list-dependencies $SERVICE --no-pager
echo ""

echo "5. Service configuration:"
systemctl cat $SERVICE --no-pager
echo ""

echo "6. Show all properties:"
systemctl show $SERVICE --no-pager | grep -E "(Failed|Active|MainPID|Environment|WorkingDirectory|User|Group)"
echo ""

echo "7. Check for port conflicts (if applicable):"
# Extract port from service if possible
PORT=$(systemctl cat $SERVICE | grep -oP 'port[=:]\s*\K[0-9]+' | head -1)
if [ ! -z "$PORT" ]; then
    echo "Checking port $PORT:"
    ss -tulpn | grep :$PORT || echo "Port $PORT not in use"
fi
echo ""

echo "8. Test manually as service user:"
USER=$(systemctl show $SERVICE --property=User --value)
EXECSTART=$(systemctl show $SERVICE --property=ExecStart --value)
echo "User: $USER"
echo "Command: $EXECSTART"
echo ""

echo "9. Check file permissions:"
# Extract command path
CMD=$(echo $EXECSTART | awk '{print $1}')
if [ -f "$CMD" ]; then
    echo "Permissions for $CMD:"
    ls -la $CMD
    namei -l $CMD
fi
echo ""

echo "10. SELinux context (if enabled):"
if [ -f "$CMD" ] && [ "$(getenforce)" != "Disabled" ]; then
    ls -laZ $CMD
fi
```

---

## **Part 8: Using systemd-analyze for Performance Issues**

### **When Service is Slow to Start**
```bash
# Find what's taking time
systemd-analyze blame | grep webapp

# Critical path for our service
systemd-analyze critical-chain webapp

# Generate visualization (requires graphviz)
systemd-analyze plot > boot.svg
# Open with: xdg-open boot.svg  or convert to PNG

# Check service startup time
systemd-analyze service-time webapp
```

### **Optimizing Startup Time**
```bash
# Check if service can be delayed
sudo systemctl edit webapp
```
Add:
```ini
[Unit]
# Don't delay other services
After=network-online.target
Wants=network-online.target
```

---

## **Part 9: Practice Exercises - Fix These Broken Services**

### **Exercise 1: The "oneshot" Service That Should Stay Running**
```bash
# Broken service
sudo tee /etc/systemd/system/broken1.service << 'EOF'
[Unit]
Description=Broken Service 1

[Service]
Type=oneshot  # WRONG TYPE!
ExecStart=/usr/bin/sleep infinity

[Install]
WantedBy=multi-user.target
EOF
```
**Task:** Fix it so it stays running.

### **Exercise 2: Missing Dependencies**
```bash
# Service that needs network but doesn't declare it
sudo tee /etc/systemd/system/broken2.service << 'EOF'
[Unit]
Description=Broken Service 2

[Service]
Type=simple
ExecStart=/usr/bin/curl -s http://example.com

[Install]
WantedBy=multi-user.target
EOF
```
**Task:** Fix dependency declaration.

### **Exercise 3: Permission Issues**
```bash
# Service trying to write to root-owned directory
sudo tee /etc/systemd/system/broken3.service << 'EOF'
[Unit]
Description=Broken Service 3

[Service]
Type=simple
User=nobody
ExecStart=/bin/bash -c "echo 'test' > /root/test.txt"

[Install]
WantedBy=multi-user.target
EOF
```
**Task:** Fix permissions or change directory.

---

## **Part 10: Debugging Tools Cheat Sheet**

### **Quick Reference**
```bash
# Status & Logs
systemctl status SERVICE      # First command always
journalctl -u SERVICE         # Second command
journalctl -u SERVICE -f      # Follow logs
journalctl -u SERVICE --since "5 min ago"

# Configuration
systemctl cat SERVICE         # Show service file
systemctl show SERVICE        # Show all properties
systemctl edit SERVICE        # Edit with override

# Testing
systemctl restart SERVICE     # Restart
systemctl reload SERVICE      # Reload config
systemctl try-restart SERVICE # Restart only if running

# Process Inspection
ps aux | grep SERVICE         # Find process
ss -tulpn | grep :PORT        # Check ports
lsof -p PID                   # What files process has open
strace -p PID                 # System calls

# Environment
sudo -u USER COMMAND          # Test as service user
env                           # Check environment
printenv                      # Print all env vars

# System-wide
systemctl --failed            # All failed services
systemd-analyze blame         # Slow services
dmesg | tail -20              # Kernel messages
```

### **Common Exit Codes & Meanings**
- **0**: Success
- **1**: General error
- **126**: Permission problem
- **127**: Command not found
- **137**: SIGKILL (OOM killer)
- **143**: SIGTERM (graceful shutdown)

---

## **Part 11: Real Interview Questions & Answers**

### **Q1: "A service fails with 'Permission denied'. What do you do?"**
**A:** 
1. Check service user with `systemctl show service --property=User`
2. Test manually as that user: `sudo -u serviceuser command`
3. Check file permissions with `namei -l /path/to/command`
4. Check SELinux with `ls -laZ` and `ausearch -m avc`
5. Check if port < 1024 needs CAP_NET_BIND_SERVICE

### **Q2: "How do you debug a service that starts but immediately exits?"**
**A:**
1. Check service type: `systemctl cat service | grep Type`
2. Check exit code: `journalctl -u service | grep "exited with"`
3. Test manually to see output
4. Add `RemainAfterExit=yes` if it's a oneshot service
5. Check Restart policy

### **Q3: "What's the difference between systemctl restart and systemctl stop/start?"**
**A:** `restart` does stop then start. `stop` sends SIGTERM, waits (TimeoutStopSec), then SIGKILL. `start` begins startup sequence. Use `try-restart` to only restart if already running.

### **Q4: "How do you make a service wait for network to be ready?"**
**A:** 
```ini
[Unit]
After=network-online.target
Wants=network-online.target
```
Not just `network.target` which is when network management starts, not when it's actually online.

---

## **Test Your Skills**

1. **Create a service that writes to `/var/log/myservice.log`. Debug why it might fail.**
2. **A service works as root but fails as a regular user. List 5 things to check.**
3. **Write a one-liner to find all services that failed in the last hour.**
4. **How would you increase the timeout for a slow-starting service?**
5. **What's wrong with this service file?**
   ```ini
   [Service]
   Type=simple
   ExecStart=/bin/echo "Starting"
   RemainAfterExit=yes
   ```

**Answers:**
1. Check directory exists, permissions, SELinux, disk space, user permissions
2. File permissions, port binding (<1024), capabilities, environment variables, resource limits
3. `systemctl list-units --state=failed --since="1 hour ago"`
4. Add `TimeoutStartSec=300` to [Service] section
5. Type=simple with RemainAfterExit=yes is contradictory. Use Type=oneshot instead.
