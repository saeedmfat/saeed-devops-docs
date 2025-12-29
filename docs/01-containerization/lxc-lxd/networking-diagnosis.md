# LXD Container Networking Issue: Diagnosis and Resolution

## Table of Contents
- [Problem Overview](#problem-overview)
- [Initial Symptoms](#initial-symptoms)
- [Root Cause Analysis](#root-cause-analysis)
- [Solution Implementation](#solution-implementation)
- [Commands Reference](#commands-reference)
- [Network Configuration](#network-configuration)
- [Prevention Steps](#prevention-steps)

## Problem Overview

We encountered a networking issue where LXD containers could not reach external networks despite having proper IP configuration, DNS resolution, and the host system having full internet connectivity. The containers could communicate with the gateway and resolve DNS, but all outbound traffic to external IPs was failing.

## Initial Symptoms

### Container Network Status
- ✅ Container had IP address assigned (`10.0.100.252/24`)
- ✅ Gateway communication working (`ping 10.0.100.1` successful)
- ✅ DNS resolution working (`nslookup google.com` successful)
- ❌ External connectivity failing (`ping 8.8.8.8` 100% packet loss)
- ❌ External domain connectivity failing (`ping google.com` failing despite DNS resolution)

### Network Configuration Check
```bash
# Container network interfaces were properly configured
lxc exec ansible-base-template -- ip addr show eth0
lxc exec ansible-base-template -- ip route show

# LXD bridge showed correct configuration
lxc network show lxdbr0
```

## Root Cause Analysis

### Investigation Steps

1. **Verified LXD Bridge Configuration**
   ```bash
   lxc network show lxdbr0
   ```
   Output showed proper NAT and DHCP settings:
   ```yaml
   config:
     ipv4.address: 10.0.100.1/24
     ipv4.dhcp: "true"
     ipv4.nat: "true"
   ```

2. **Checked Host System Connectivity**
   ```bash
   ping -c 3 8.8.8.8  # Host could reach external networks
   ```

3. **Examined iptables Configuration**
   ```bash
   sudo iptables -t nat -L -v
   sudo iptables -L FORWARD -v
   ```

### Root Cause Identified
The iptables NAT rules were configured to use the wrong network interface. The rules were pointing to `enx128a72ca47f9` but the actual internet-facing interface was `enxee98f4c537ca`.

**Incorrect Configuration:**
```bash
# Wrong interface in NAT rules
POSTROUTING chain: MASQUERADE for 10.0.100.0/24 via enx128a72ca47f9
FORWARD chain: Rules using enx128a72ca47f9 instead of actual interface
```

**Actual Network Interfaces:**
```bash
ip a  # Showed enxee98f4c537ca as the main interface with internet access
```

## Solution Implementation

### Step 1: Remove Incorrect iptables Rules
```bash
# Remove incorrect NAT rule
sudo iptables -t nat -D POSTROUTING -s 10.0.100.0/24 -o enx128a72ca47f9 -j MASQUERADE

# Remove incorrect forward rules
sudo iptables -D FORWARD -i lxdbr0 -o enx128a72ca47f9 -j ACCEPT
sudo iptables -D FORWARD -i enx128a72ca47f9 -o lxdbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Step 2: Add Correct iptables Rules
```bash
# Add NAT masquerading for the correct interface
sudo iptables -t nat -A POSTROUTING -s 10.0.100.0/24 -o enxee98f4c537ca -j MASQUERADE

# Allow forwarding from LXD bridge to internet
sudo iptables -I FORWARD -i lxdbr0 -o enxee98f4c537ca -j ACCEPT

# Allow established connections back to LXD bridge
sudo iptables -I FORWARD -i enxee98f4c537ca -o lxdbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Step 3: Restart Container and Verify
```bash
# Restart container to ensure clean network state
lxc stop ansible-base-template --force
lxc start ansible-base-template

# Verify connectivity
lxc exec ansible-base-template -- ping -c 3 8.8.8.8
lxc exec ansible-base-template -- ping -c 3 google.com
```

## Commands Reference

### Diagnostic Commands
| Command | Purpose |
|---------|---------|
| `lxc network show lxdbr0` | Display LXD bridge configuration |
| `lxc exec <container> -- ip addr show` | Show container network interfaces |
| `lxc exec <container> -- ip route show` | Display container routing table |
| `lxc exec <container> -- ping -c 3 8.8.8.8` | Test external connectivity |
| `lxc exec <container> -- nslookup google.com` | Test DNS resolution |
| `sudo iptables -t nat -L -v` | Show NAT table with packet counts |
| `sudo iptables -L FORWARD -v` | Show forward chain rules |
| `ip a` | Display host network interfaces |
| `ip route` | Show host routing table |

### Resolution Commands
| Command | Purpose |
|---------|---------|
| `sudo iptables -t nat -D POSTROUTING -s <subnet> -o <interface> -j MASQUERADE` | Remove incorrect NAT rule |
| `sudo iptables -D FORWARD -i <in_iface> -o <out_iface> -j ACCEPT` | Remove forward rule |
| `sudo iptables -t nat -A POSTROUTING -s <subnet> -o <correct_interface> -j MASQUERADE` | Add correct NAT rule |
| `sudo iptables -I FORWARD -i lxdbr0 -o <correct_interface> -j ACCEPT` | Allow outbound traffic |
| `sudo iptables -I FORWARD -i <correct_interface> -o lxdbr0 -m state --state RELATED,ESTABLISHED -j ACCEPT` | Allow inbound established traffic |

## Network Configuration

### Final Working Configuration
```bash
# NAT Table
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  10.0.100.0/24        anywhere            via enxee98f4c537ca

# FORWARD Chain
Chain FORWARD (policy DROP)
target     prot opt in     out     source               destination         
ACCEPT     all  --  enxee98f4c537ca lxdbr0  anywhere             anywhere             state RELATED,ESTABLISHED
ACCEPT     all  --  lxdbr0 enxee98f4c537ca  anywhere             anywhere            
```

### LXD Network Profile
```yaml
lxc network show lxdbr0:
  ipv4.address: 10.0.100.1/24
  ipv4.dhcp: "true"
  ipv4.nat: "true"
  ipv4.dhcp.ranges: 10.0.100.100-10.0.100.200
```

## Prevention Steps

1. **Always verify the actual network interface name:**
   ```bash
   ip a | grep -E "state UP"
   ```

2. **Use consistent interface naming in iptables rules**

3. **Test connectivity after network changes:**
   ```bash
   lxc exec <container> -- ping -c 3 8.8.8.8
   lxc exec <container> -- ping -c 3 google.com
   ```

4. **Monitor iptables counters to verify traffic flow:**
   ```bash
   sudo iptables -t nat -L -v
   sudo iptables -L FORWARD -v
   ```

## Verification Checklist

- [ ] Container can ping external IP addresses (8.8.8.8)
- [ ] Container can resolve and ping external domains (google.com)
- [ ] Container can download packages (`apt update`)
- [ ] iptables counters show packet flow in NAT and FORWARD chains
- [ ] LXD bridge shows correct NAT and DHCP configuration
