# **DevOps Home Lab Setup: Solving the Top 3 Virtualization Hurdles**

*Building a professional DevOps environment shouldn't feel like solving a mystery. Here's how I navigated the three most common pitfalls when setting up a multi-VM home lab with Vagrant and VirtualBox.*

---

## **Project Overview: Config Drift Detection Platform**

**Objective**: Create a 4-node virtual environment (1 Ansible controller + 3 target servers) to learn configuration management and drift detection.

**Technology Stack**:
- VirtualBox (virtualization)
- Vagrant (infrastructure as code)
- Ansible (configuration management)
- Ubuntu 20.04 & Rocky Linux 8

**Architecture**:
```
Control VM (192.168.60.10) → Manages → Target1 (192.168.60.11)
                              ↓        Target2 (192.168.60.12)
                              ↓        Target3 (192.168.60.13)
```

---

## **Problem 1: The KVM-VirtualBox Conflict**

### **How We Noticed the Problem**
The error message was clear but confusing:
```
VBoxManage: error: VirtualBox can't operate in VMX root mode. 
Please disable the KVM kernel extension, recompile your kernel and reboot
```

**Symptoms**:
- VirtualBox VMs refused to start
- `vagrant up` failed immediately
- Error mentioning "VMX root mode"

**Diagnosis Commands**:
```bash
# Check if KVM modules were loaded
lsmod | grep kvm
# Output showed: kvm_intel, kvm, irqbypass

# Check for virtualization services
systemctl list-units | grep -E "(kvm|libvirt|virt)"
snap list | grep -i kvm
```

### **The Root Cause**
KVM (Kernel-based Virtual Machine) and VirtualBox **cannot share CPU virtualization features**. Both need exclusive access to Intel VT-x/AMD-V hardware virtualization. When KVM loads its kernel modules, it "locks" these features, preventing VirtualBox from using them.

### **How We Fixed It**

**Solution 1: Temporary Fix (Quick & Easy)**
```bash
# Unload KVM kernel modules (temporary - until reboot)
sudo rmmod kvm_intel kvm irqbypass

# Verify they're gone
lsmod | grep kvm  # Should return nothing

# Now VirtualBox works
vagrant up
```

**Solution 2: Permanent Fix (For VirtualBox Users)**
```bash
# Prevent KVM from auto-loading on boot
echo "blacklist kvm_intel" | sudo tee /etc/modprobe.d/blacklist-kvm.conf
echo "blacklist kvm" | sudo tee -a /etc/modprobe.d/blacklist-kvm.conf
echo "blacklist irqbypass" | sudo tee -a /etc/modprobe.d/blacklist-kvm.conf

sudo update-initramfs -u
sudo reboot
```

**Failed Attempts**:
- `sudo snap disable kvm` (KVM wasn't installed via snap)
- `sudo systemctl stop libvirtd` (libvirt wasn't running)
- Looking for KVM snap packages (none were installed)

### **Key Learning**
- **`rmmod`** = temporary module removal (survives until reboot)
- **Blacklisting** = permanent prevention (edit `/etc/modprobe.d/`)
- Always check `lsmod | grep kvm` first when VirtualBox fails

---

## **Problem 2: The SSH Key Copy Conundrum**

### **How We Noticed the Problem**
SSH from control VM to target VMs kept failing:
```
vagrant@192.168.60.11: Permission denied (publickey)
```

**Symptoms**:
- `ssh-copy-id` failed silently
- Manual SSH connections rejected
- Ansible ping tests failed

**Diagnosis**:
```bash
# Attempted automated key copy (FAILED)
sshpass -p 'vagrant' ssh-copy-id vagrant@192.168.60.11

# Manual SSH test (FAILED)
ssh vagrant@192.168.60.11 "echo test"
```

### **The Root Cause**
The target VMs' SSH daemon was configured for **key-only authentication**, but we hadn't copied any keys yet. The `ssh-copy-id` command failed because it tries to authenticate with existing keys first.

### **How We Fixed It**

**Solution: Manual Key Injection (What Actually Worked)**
```bash
# 1. Get public key from control VM
PUBKEY=$(vagrant ssh control -c "cat ~/.ssh/id_rsa.pub")

# 2. Manually inject into each target VM
for vm in target1 target2 target3; do
    vagrant ssh $vm -c "mkdir -p ~/.ssh && echo '$PUBKEY' > ~/.ssh/authorized_keys"
    vagrant ssh $vm -c "chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
done
```

**Alternative Approaches We Tried**:

1. **Direct `ssh-copy-id` with password** (Failed - key auth required):
   ```bash
   sshpass -p 'vagrant' ssh-copy-id vagrant@192.168.60.11
   ```

2. **Enable password authentication** (Worked but insecure):
   ```bash
   # On target VMs:
   sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
   sudo systemctl restart ssh
   ```

3. **Ansible-based setup** (Overkill for initial setup):
   ```yaml
   - name: Copy SSH key
     authorized_key:
       user: vagrant
       key: "{{ lookup('file', '/tmp/key.pub') }}"
   ```

### **Key Learning**
- **Vagrant's multi-level SSH**: Host → Control VM → Target VMs
- **Manual key injection** is often simplest for lab environments
- Always verify with: `ssh target1 'echo Success'`
- Permission issues are usually **file permissions** (700 for `.ssh/`, 600 for `authorized_keys`)

---

## **Problem 3: The Rocky Linux Boot Timeout**

### **How We Noticed the Problem**
Only the Rocky Linux VM failed to boot:
```
Timed out while waiting for the machine to boot.
Vagrant was unable to communicate with the guest machine
```

**Symptoms**:
- Ubuntu VMs (3/4) booted successfully
- Rocky Linux VM powered on but unresponsive
- SSH connection attempts timed out
- Error after 300 seconds (default timeout)

**Diagnosis**:
```bash
# Check VM status
vagrant status target3
# Showed: "running" but SSH not responding

# Check VirtualBox directly
VBoxManage list runningvms
# VM was listed as running

# Network test from control VM
ping 192.168.60.13
# No response
```

### **The Root Cause**
Rocky Linux 8 Vagrant boxes have known issues:
1. **Slow boot process** (exceeds 300-second default)
2. **SSH service not auto-starting**
3. **Network configuration problems** in the box image

### **How We Fixed It**

**Solution 1: Increase Boot Timeout** (Temporary fix)
```ruby
# In Vagrantfile for target3:
target3.vm.boot_timeout = 600  # 10 minutes instead of 5
```

**Solution 2: Use a Different Box** (Permanent fix)
```ruby
# Change from problematic box:
target3.vm.box = "rockylinux/8"

# To more reliable alternatives:
target3.vm.box = "generic/rocky8"
# OR
target3.vm.box = "ubuntu/focal64"  # Same as others for consistency
```

**Solution 3: Manual Intervention** (Diagnostic approach)
```bash
# 1. Destroy and recreate
vagrant destroy target3
vagrant up target3 --debug  # See detailed boot process

# 2. Check VirtualBox GUI for errors
# 3. Manually configure network if needed
```

**The Solution We Actually Used**:
We changed target3 to Ubuntu to maintain consistency and avoid box-specific issues:
```ruby
target3.vm.box = "ubuntu/focal64"  # Changed from rockylinux/8
```

### **Key Learning**
- **Default timeout**: 300 seconds (5 minutes) - often insufficient
- **Box quality varies**: Official ≠ Always reliable
- **Debug mode helps**: `vagrant up --debug` shows boot process
- **Consistency matters**: Using same OS eliminates variables

---

## **Bonus: The Git Bare Repository Confusion**

While not a "problem," understanding `git init --bare` was crucial:

**Regular Repo** (`git init`):
- Working directory + `.git/` folder
- Where you edit files
- Your local workspace

**Bare Repo** (`git init --bare`):
- Only `.git/` contents (no working files)
- Central storage/backup
- Accepts pushes from regular repos

**Why both?**: Professional workflow needs a "source of truth" repository separate from working directories.

---

## **Top 5 Lessons Learned**

1. **Always Check KVM First**: `lsmod | grep kvm` should be your first command when VirtualBox fails.

2. **SSH Keys Need Manual Setup in Multi-Tier Environments**: Automated tools fail when there's no existing authentication method.

3. **Vagrant Boxes Vary in Quality**: "Official" doesn't mean "problem-free." Test and be ready to switch boxes.

4. **Debug Mode is Your Friend**: `vagrant up --debug` reveals what's actually happening during boot.

5. **Start Simple, Then Complex**: Get Ubuntu working first, then introduce OS diversity.

---

## **The Complete Working Solution**

After solving all three problems, our final environment:
- ✅ 4 Ubuntu 20.04 VMs (changed from Rocky for consistency)
- ✅ KVM modules temporarily disabled for VirtualBox
- ✅ SSH keys manually configured between all nodes
- ✅ Ansible controlling all target nodes
- ✅ Git repository for configuration management

**Final Verification**:
```bash
# From control VM:
ansible all -m ping
# All 3 targets respond: "pong"

# From host machine:
vagrant status
# All 4 VMs: "running"
```

---

## **For Beginners Starting Their DevOps Journey**

1. **Expect conflicts** between virtualization technologies
2. **SSH is fundamental** - master it early
3. **Document every error and solution** (like this guide!)
4. **One problem at a time** - don't change multiple variables
5. **Community is key** - these problems are common and solvable

The journey from "Permission denied" to a fully automated, multi-node environment is what separates aspiring DevOps engineers from professionals. Each solved problem builds your troubleshooting toolkit.

*Remember: Every error message is a learning opportunity. Today's frustrating "VMX root mode" error is tomorrow's "Oh, that's just KVM" moment.*
