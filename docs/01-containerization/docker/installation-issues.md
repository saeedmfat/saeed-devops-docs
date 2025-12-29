Docker Installation Issue Resolution Document
Problem Overview
Initial State: Mixed Docker installation causing conflicts

Snap Docker: Version 28.4.0

APT Docker (docker.io): Version 28.2.2

Symptoms: Version mismatches, permission errors, and socket connection issues

Diagnostic Steps Performed
1. Identification of Installation Methods
bash
# Check Snap installation
snap list | grep docker
# OUTPUT: docker 28.4.0 (installed)

# Check APT installation  
dpkg -l | grep docker
# OUTPUT: docker.io 28.2.2 (installed)
2. Binary Location Analysis
bash
which docker
# OUTPUT: /usr/bin/docker (APT location)

docker --version
# OUTPUT: Docker version 28.4.0 (but binary from APT - indicating conflict)
3. Service Status Check
bash
systemctl status docker
# OUTPUT: Active (running) but socket connection failures
Issues Discovered
Primary Issue: Mixed Installation Conflict
Snap Docker: Version 28.4.0 creating PATH conflicts

APT Docker: Version 28.2.2 providing actual binaries

Shell Cache: Remembering incorrect paths

Secondary Issue: Socket Creation Failure
Docker daemon running but no socket file created

Systemd socket activation service corrupted

Permission and filesystem issues

Resolution Steps Attempted
✅ Successful Solutions
1. Removed Conflicting Snap Installation
bash
sudo snap remove docker
Result: Eliminated version conflict source

2. Cleared Shell Command Cache
bash
hash -d docker
hash -r
Result: Fixed PATH resolution, now showing correct APT version (28.2.2)

3. Fixed Systemd Socket Service
bash
sudo systemctl stop docker.socket
sudo systemctl disable docker.socket
sudo rm -f /var/run/docker.sock /run/docker.sock
sudo systemctl daemon-reload
sudo systemctl start docker.service
Result: Socket files created successfully at both locations:

/run/docker.sock

/var/run/docker.sock

❌ Attempted but Unsuccessful Solutions
Permission fixes alone - Insufficient without socket creation

Environment variable setting - Didn't resolve core socket issue

Manual socket creation - System-level blocking prevented creation

Simple service restart - Didn't address corrupted socket activation

Root Cause Analysis
The Core Problem
The docker.socket systemd unit was in a corrupted state showing "Device or resource busy" error, preventing proper socket file creation despite the Docker daemon running normally.

The Conflict Chain
Snap Docker installation hijacked command resolution

Shell cache remembered incorrect paths

Systemd socket activation service entered failed state

Socket files couldn't be created due to resource locks

Final Working Configuration
Verified Working State
Installation Method: APT (docker.io package)

Version: 28.2.2 (consistent client and server)

Binary Location: /usr/bin/docker

Socket Files: Created at /run/docker.sock and /var/run/docker.sock

Service: docker.service running directly (socket activation disabled)

Required User Setup
bash
# Add user to docker group
sudo usermod -aG docker $USER

# Activate group changes
newgrp docker
# OR log out and back in
Key Lessons Learned
1. Installation Method Consistency
Avoid mixing Snap and APT installations of the same software

Choose one package manager and stick with it

2. Shell Behavior Understanding
Shell caches command locations (hash table)

Changes require cache clearance or new sessions

3. Systemd Socket Activation
Socket activation can fail independently of the main service

Sometimes direct service start is more reliable

"Device or resource busy" indicates deeper system-level issues

4. Diagnostic Methodology
Always check both installation methods (Snap and APT)

Verify actual binary locations vs. cached paths

Check service status AND socket file existence separately

Prevention Recommendations
Choose installation method intentionally - Don't install both Snap and APT versions

Verify clean removal - When switching methods, ensure complete removal

Monitor systemd unit states - Regular checks for failed units

Use diagnostic scripts - Regular system health checks for Docker

Quick Recovery Script for Future
bash
#!/bin/bash
# Docker recovery script
sudo snap remove docker 2>/dev/null
hash -r
sudo systemctl stop docker.socket docker.service 2>/dev/null
sudo rm -f /var/run/docker.sock /run/docker.sock
sudo systemctl disable docker.socket 2>/dev/null
sudo systemctl daemon-reload
sudo systemctl start docker.service
This document captures the complete troubleshooting journey and provides a reference for resolving similar Docker installation conflicts in the future.

