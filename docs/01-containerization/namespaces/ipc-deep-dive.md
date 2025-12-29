# **IPC Namespace Deep Dive: From Zero Level**

## **Part 1: What is IPC? Why Do We Need It?**

### **The Problem IPC Solves**
Imagine two programs running on the same computer:

```c
// Program A (Producer)
// Needs to send data to Program B
// Can't just write to a file (too slow)
// Can't use network (both are local)
// Can't use pipes (need persistent connection)

// Program B (Consumer)
// Needs fast, reliable communication with Program A
```

**IPC is the solution** - Like passing notes between classmates without talking aloud.

---

## **Part 2: Types of IPC That Get Isolated**

### **1. System V IPC - The Old School Method**
Created in UNIX System V (1983), still widely used.

#### **A. Shared Memory (`shmget`, `shmat`)**
```c
// WITHOUT namespace isolation
key_t key = ftok("/tmp", 'A');
int shmid = shmget(key, 1024, 0666|IPC_CREAT);
// Any process with same key can access!

char *str = shmat(shmid, NULL, 0);
sprintf(str, "Hello from Process 1");
// Another container's process could read/write this!
```

**Visual analogy:**
```
BEFORE IPC Namespace:
┌─────────────────────────────────────┐
│        Host System                  │
├──────────┬──────────┬───────────────┤
│Container │Container │Host Process   │
│   A      │   B      │               │
│  PID 100 │ PID 200  │ PID 50        │
│          │          │               │
│  ┌─────┐ │ ┌─────┐  │ ┌─────┐       │
│  │SHM  │ │ │SHM  │  │ │SHM  │       │
│  │Key X├─┼─┤Key X│  │ │Key X│       │
│  └─────┘ │ └─────┘  │ └─────┘       │
│          │          │               │
└──────────┴──────────┴───────────────┘
All see same shared memory with Key X!
```

#### **B. Semaphores (`semget`, `semop`)**
```c
// Think of semaphores as bathroom keys
key_t key = ftok("/tmp", 'S');
int semid = semget(key, 1, 0666|IPC_CREAT);

// Lock (wait) - take the key
struct sembuf lock = {0, -1, SEM_UNDO};
semop(semid, &lock, 1);

// Critical section - in the bathroom
// ...

// Unlock (signal) - return the key
struct sembuf unlock = {0, 1, SEM_UNDO};
semop(semid, &unlock, 1);
```

**Problem without isolation**: Containers could deadlock each other by holding "keys" meant for others.

#### **C. Message Queues (`msgget`, `msgsnd`)**
```c
struct message {
    long mtype;
    char mtext[100];
};

key_t key = ftok("/tmp", 'M');
int msgid = msgget(key, 0666|IPC_CREAT);

// Send message
struct message msg;
msg.mtype = 1;
strcpy(msg.mtext, "Secret container data");
msgsnd(msgid, &msg, sizeof(msg.mtext), 0);
// Another container could receive this!
```

### **2. POSIX IPC - The Modern Standard**
```c
// Named semaphore
sem_t *sem = sem_open("/my_semaphore", O_CREAT, 0644, 1);

// POSIX shared memory
int fd = shm_open("/my_shm", O_CREAT|O_RDWR, 0644);

// POSIX message queue
mqd_t mq = mq_open("/my_queue", O_CREAT|O_RDWR, 0644, NULL);
```

**Key difference**: POSIX IPC uses filesystem paths, System V uses numeric keys.

---

## **Part 3: The IPC Namespace - The Isolation Solution**

### **How It Works Internally**
```c
// Kernel data structures simplified
struct ipc_namespace {
    struct ipc_ids ids[3];  // For shm, sem, msg
    struct mq_namespace *mq_ns;  // POSIX MQ
    // Each has its own ID counter starting at 0
};

// When process creates IPC object in namespace:
struct ipc_namespace *ns = current->nsproxy->ipc_ns;
int new_id = ipc_addid(ns->ids[type], new_object);
// This ID only valid within this namespace!
```

### **Visualizing Isolation**
```
AFTER IPC Namespace Isolation:
┌─────────────────────────────────────────────┐
│            Host System                      │
├─────────────────┬─────────────────┬─────────┤
│Container A      │Container B      │Host     │
│(IPC Namespace 1)│(IPC Namespace 2)│(Root NS)│
│                 │                 │         │
│ IPC Objects:    │ IPC Objects:    │IPC Obj: │
│ • SHM ID 0      │ • SHM ID 0      │• SHM ID 0│
│ • SEM ID 0      │ • MSG ID 0      │• SEM ID 1│
│ • /dev/shm/private │ • /dev/shm/b_data │         │
│                 │                 │         │
│ Key 1234 → ID 0 │ Key 1234 → ID 0│Key 1234 → ID 2│
│ (local mapping) │ (different obj!)│(different!)│
└─────────────────┴─────────────────┴─────────┘
Same key, different objects across namespaces!
```

---

## **Part 4: `/dev/shm` - The Shared Memory Filesystem**

### **What is `/dev/shm`?**
```bash
# It's a tmpfs mount for shared memory
$ df -h /dev/shm
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            64M     0   64M   0% /dev/shm

# How processes use it:
# 1. Create shared memory
int fd = shm_open("/myshm", O_CREAT|O_RDWR, 0644);
ftruncate(fd, 1024);

# 2. Memory map it
char *ptr = mmap(NULL, 1024, PROT_READ|PROT_WRITE, 
                 MAP_SHARED, fd, 0);
```

### **Isolation with Mount Namespaces**
```bash
# Container sees its own /dev/shm
$ docker run -it alpine sh
/ # ls /dev/shm/
# Empty - private to this container

# Host sees different /dev/shm
$ ls /dev/shm/
oracle_shm  postgres_shm  # Host processes' shared mem
```

**Kernel mounts it like this:**
```c
// Simplified kernel mount code
mount("tmpfs", "/dev/shm", "tmpfs", 
      MS_NOSUID|MS_NODEV, "size=64M");
// Each mount namespace gets its own instance
```

---

## **Part 5: Practical Examples & Commands**

### **1. Inspecting IPC Objects**
```bash
# On host/system
$ ipcs  # Show ALL IPC objects

# Shared memory
$ ipcs -m
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x00000000 65536      alice      600        1024       2                       

# Semaphores  
$ ipcs -s
------ Semaphore Arrays --------
key        semid      owner      perms      nsems     
0x00000000 32769      bob        666        1

# Message queues
$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x00000000 98304      charlie    644        80           2

# POSIX message queues
$ ls /dev/mqueue/
myqueue  another_queue
```

### **2. Creating IPC Namespace**
```bash
# Create new IPC namespace
$ sudo unshare --ipc bash
$ echo $$  # Note PID

# In NEW namespace
$ ipcs  # Empty! Isolated
$ ipcmk -M 1024  # Create shared memory
$ ipcs -m  # Only see our own

# In another terminal (host namespace)
$ ipcs -m  # Can't see the new one!
```

### **3. Docker Examples**
```bash
# Container WITHOUT IPC namespace sharing
$ docker run -d --name container1 alpine sleep 1000
$ docker run -d --name container2 alpine sleep 1000

# Each has own IPC space
$ docker exec container1 ipcs
# Empty or container1's objects only

# Container WITH shared IPC namespace
$ docker run -d --name container1 --ipc=shareable alpine sleep 1000
$ docker run -d --name container2 --ipc=container:container1 alpine sleep 1000

# Now container2 can see container1's IPC objects
```

### **4. Security Implications**
```c
// ATTACK without IPC namespace:
// Container A could do:
key_t key = ftok("/some/common/file", 'X');
int shmid = shmget(key, 1024, 0666);
// Might get host's or another container's shared memory!

// With IPC namespace:
// Always gets private objects, even with same key
```

---

## **Part 6: Real-World Use Cases**

### **Case 1: PostgreSQL in Container**
```bash
# PostgreSQL uses shared memory for buffer cache
docker run -d --name postgres \
  -e POSTGRES_PASSWORD=secret \
  postgres:14

# Inside container:
$ ipcs -m
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     
0x0052e2c1 0          postgres   600        1642496    6
# Private to this container, safe from others
```

### **Case 2: Multi-process Applications**
```python
# Python multiprocessing with shared memory
from multiprocessing import shared_memory
import numpy as np

# Creates /dev/shm/pyshm_*
shm = shared_memory.SharedMemory(
    create=True, 
    size=100, 
    name='my_shm'
)
# Only visible within container's IPC namespace
```

### **Case 3: Kubernetes Pods**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-ipc-pod
spec:
  shareProcessNamespace: false  # Default
  # Each container gets own IPC namespace
  
  containers:
  - name: app
    image: myapp
  - name: sidecar  
    image: sidecar
    # Cannot see app's IPC objects by default
```

---

## **Part 7: Deep Dive: Kernel Implementation**

### **Key Data Structures**
```c
// In Linux kernel source (simplified):

// Each IPC object has namespace-aware ID
struct ipc_ids {
    int in_use;
    unsigned short seq;  // Sequence counter
    struct kern_ipc_perm *p[IPCMNI];
    // IPCMNI = 32768 max objects per type per namespace
};

// The permission structure
struct kern_ipc_perm {
    spinlock_t lock;
    int deleted;
    key_t key;          // System V key
    uid_t uid;          // Owner UID  
    gid_t gid;          // Owner GID
    uid_t cuid;         // Creator UID
    gid_t cgid;         // Creator GID
    umode_t mode;       // Permission bits
    unsigned long seq;  // Sequence number
    void *security;     // LSM (SELinux/AppArmor) data
    // NO namespace field! Objects belong to namespace's table
};

// Process namespace proxy
struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns;
    struct net *net_ns;
    struct time_namespace *time_ns;
    struct cgroup_namespace *cgroup_ns;
};
```

### **How Objects Are Created & Found**
```c
// Simplified sequence:

// 1. Process calls shmget(key, size, flags)
SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
    struct ipc_namespace *ns = current->nsproxy->ipc_ns;
    
    // 2. Convert key to IPC ID within namespace
    if (key == IPC_PRIVATE) {
        // Always create new object
        return new_shm(ns, size, shmflg);
    } else {
        // Lookup in namespace's table
        id = ipc_findkey(ns->ids[IPC_SHM_IDX], key);
        if (id < 0) {
            // Not found, create if IPC_CREAT
            if (shmflg & IPC_CREAT)
                return new_shm(ns, size, shmflg);
            else
                return -ENOENT;
        }
        // Found existing in THIS namespace
        return id;
    }
}

// 3. Key collision across namespaces doesn't matter!
// Same key in different namespace = different object
```

---

## **Part 8: Common Issues & Debugging**

### **Problem 1: "Permission Denied" Across Containers**
```bash
# Container A creates shm with key 1234
# Container B tries to access same key 1234
# Gets different object or ENOENT!

# Solution: Share IPC namespace
docker run --ipc=container:<other_container> ...
```

### **Problem 2: `/dev/shm` Too Small**
```bash
# Default is 64MB
docker run --shm-size=256m myapp
# or
docker run --ipc=host  # Uses host's /dev/shm (DANGEROUS!)
```

### **Problem 3: POSIX IPC Persistence**
```c
// POSIX objects can survive process termination
sem_unlink("/mysem");  // Always clean up!
shm_unlink("/myshm");
mq_unlink("/myqueue");
```

### **Debugging Commands**
```bash
# 1. See namespace IDs
$ ls -la /proc/$$/ns/
lrwxrwxrwx 1 root root 0 Jan 1 12:00 ipc -> ipc:[4026531839]

# 2. Enter container's IPC namespace
$ nsenter --ipc=/proc/<container_pid>/ns/ipc bash
$ ipcs  # Now see container's IPC objects

# 3. Check IPC limits
$ sysctl kernel.shmall kernel.shmmax kernel.shmmni
kernel.shmall = 18446744073692774399
kernel.shmmax = 18446744073692774399  
kernel.shmmni = 4096

# 4. Find processes using IPC
$ lsof | grep /dev/shm
$ fuser /dev/shm/myshm
```

---

## **Part 9: Security Best Practices**

### **1. Never Use `--ipc=host`**
```bash
# BAD: Container can see/affect host IPC
docker run --ipc=host myapp

# BAD: Container can attack other containers
docker run --ipc=container:other_container attacker

# GOOD: Default isolation
docker run myapp  # Gets private IPC namespace
```

### **2. Clean Up IPC Resources**
```c
// In your application:
void cleanup() {
    // System V IPC
    shmctl(shmid, IPC_RMID, NULL);
    semctl(semid, 0, IPC_RMID);
    msgctl(msgid, IPC_RMID, NULL);
    
    // POSIX IPC  
    sem_close(sem);
    sem_unlink("/mysem");
    shm_unlink("/myshm");
}
```

### **3. Use Filesystem IPC When Possible**
```bash
# Instead of shared memory across containers:
# Use volume mounts
docker run -v /shared_data:/data app1
docker run -v /shared_data:/data app2

# Or use network communication
# (HTTP, gRPC, Redis, etc.)
```

---

## **Part 10: The Big Picture**

### **Why IPC Namespace Matters:**
1. **Security**: Prevents containers from stealing/interfering with IPC
2. **Predictability**: Same key always gives same object within namespace
3. **Cleanup**: When container dies, its IPC objects disappear
4. **Resource Control**: Limits per container, not global

### **Complete Isolation Example:**
```bash
# Start 3 containers
for i in {1..3}; do
  docker run -d --name "container$i" alpine sleep 1000
done

# Each has:
# - Own PID namespace (can't see others' processes)
# - Own IPC namespace (can't see others' shm/sem/msg)
# - Own network namespace (own IP, ports)
# - Own UTS namespace (own hostname)
# - Own mount namespace (own /dev/shm)

# But they CAN communicate via:
# - Network (if connected)
# - Volumes (if shared)
# - Signals (if PID namespace shared)
# - But NOT via System V/POSIX IPC!
```

The IPC namespace completes the isolation story, ensuring that even the most intimate form of inter-process communication (shared memory) remains private to each container, just like soundproof rooms in an apartment building.
