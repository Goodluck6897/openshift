
# 🔓 Accessing Host PID / IPC in OpenShift SCC

> **⚠️ Security Warning:** Enabling `hostPID` or `hostIPC` breaks critical container isolation boundaries. Containers can see and interact with host processes and shared memory. Use only when absolutely necessary and with full understanding of the risks.

---

## 📋 Table of Contents

- [What are Host PID and Host IPC?](#what-are-host-pid-and-host-ipc)
- [How Isolation Works by Default](#how-isolation-works-by-default)
- [SCCs That Allow Host PID / IPC](#sccs-that-allow-host-pid--ipc)
- [Part 1: Host PID Namespace](#part-1-host-pid-namespace)
  - [Method 1: Using the `hostaccess` SCC](#method-1-using-the-hostaccess-scc)
  - [Method 2: Using the `privileged` SCC](#method-2-using-the-privileged-scc)
  - [Method 3: Custom SCC for Host PID Only](#method-3-custom-scc-for-host-pid-only)
  - [Method 4: RBAC Approach](#method-4-rbac-approach-hostpid)
- [Part 2: Host IPC Namespace](#part-2-host-ipc-namespace)
  - [Method 1: Using the `hostaccess` SCC](#method-1-using-the-hostaccess-scc-ipc)
  - [Method 2: Custom SCC for Host IPC Only](#method-2-custom-scc-for-host-ipc-only)
  - [Method 3: RBAC Approach](#method-3-rbac-approach-hostipc)
- [Part 3: Host PID + IPC Combined](#part-3-host-pid--ipc-combined)
- [Real-World Examples](#real-world-examples)
- [What Host PID / IPC Exposes](#what-host-pid--ipc-exposes)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Common Errors and Fixes](#common-errors-and-fixes)
- [Detection & Auditing](#detection--auditing)
- [Security Best Practices](#security-best-practices)

---

## What are Host PID and Host IPC?

### Host PID (`hostPID: true`)

Shares the **host's process ID namespace** with the container. The container can see **all processes** running on the node — not just its own.

```
┌──────────────────────────────────────────────────────┐
│                     Node (Host)                      │
│                                                      │
│  PID 1: systemd                                      │
│  PID 512: kubelet                                    │
│  PID 780: crio                                       │
│  PID 1200: etcd                                      │
│                                                      │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │  Normal Pod         │  │  hostPID Pod           │ │
│  │                     │  │                        │ │
│  │  Sees only:         │  │  Sees ALL:             │ │
│  │  PID 1: sleep       │  │  PID 1: systemd       │ │
│  │  (own process)      │  │  PID 512: kubelet     │ │
│  │                     │  │  PID 780: crio        │ │
│  │                     │  │  PID 1200: etcd       │ │
│  │                     │  │  PID 5001: sleep      │ │
│  │                     │  │  (own + all host)     │ │
│  └─────────────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Host IPC (`hostIPC: true`)

Shares the **host's Inter-Process Communication namespace**. The container can access shared memory segments, semaphores, and message queues used by host processes.

```
┌──────────────────────────────────────────────────────┐
│                     Node (Host)                      │
│                                                      │
│  Shared Memory: /dev/shm/db_cache (PostgreSQL)       │
│  Semaphore: sem_app_lock (App Server)                │
│  Message Queue: mq_worker (Worker Process)           │
│                                                      │
│  ┌─────────────────────┐  ┌────────────────────────┐ │
│  │  Normal Pod         │  │  hostIPC Pod           │ │
│  │                     │  │                        │ │
│  │  Own IPC only:      │  │  Sees ALL host IPC:    │ │
│  │  (isolated)         │  │  /dev/shm/db_cache     │ │
│  │                     │  │  sem_app_lock          │ │
│  │                     │  │  mq_worker             │ │
│  └─────────────────────┘  └────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## How Isolation Works by Default

| Namespace | Default (restricted SCC) | With hostPID/hostIPC |
|:--|:--|:--|
| **PID** | Isolated — container sees only its own processes | Shared — container sees all host processes |
| **IPC** | Isolated — container has its own shared memory | Shared — container accesses host shared memory |
| **Network** | Isolated — pod gets its own IP | Unchanged (unless `hostNetwork` also set) |
| **Mount** | Isolated — container has its own filesystem | Unchanged (unless `hostPath` volumes used) |
| **UTS** | Isolated — container has its own hostname | Unchanged |

---

## SCCs That Allow Host PID / IPC

| SCC | `allowHostPID` | `allowHostIPC` | Other Access |
|:--|:--|:--|:--|
| `restricted` | ❌ | ❌ | None |
| `restricted-v2` | ❌ | ❌ | None |
| `nonroot` | ❌ | ❌ | None |
| `anyuid` | ❌ | ❌ | Root UID only |
| `hostnetwork` | ❌ | ❌ | Host network only |
| `hostnetwork-v2` | ❌ | ❌ | Host network only |
| `hostaccess` | ✅ | ✅ | + Host network, host dirs |
| `privileged` | ✅ | ✅ | Everything |

> 💡 **Notice:** There is **no built-in SCC** that grants only `hostPID` or only `hostIPC`. You need `hostaccess`, `privileged`, or a **custom SCC**.

---

# Part 1: Host PID Namespace

---

## Method 1: Using the `hostaccess` SCC

> Grants `hostPID` + `hostIPC` + `hostNetwork` + `hostDirVolumes`. More than you may need.

### Step 1 — Create a ServiceAccount

```bash
oc create serviceaccount hostpid-sa -n my-namespace
```

### Step 2 — Grant the `hostaccess` SCC

```bash
oc adm policy add-scc-to-user hostaccess -z hostpid-sa -n my-namespace
```

### Step 3 — Verify

```bash
oc adm policy who-can use scc hostaccess
```

### Step 4 — Deploy a pod with hostPID

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-pod
  namespace: my-namespace
spec:
  serviceAccountName: hostpid-sa
  hostPID: true
  containers:
    - name: hostpid-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
```

```bash
oc apply -f hostpid-pod.yaml
```

### Step 5 — Verify host PID access

```bash
# You should see ALL host processes, not just the container's
oc exec hostpid-pod -- ps aux
```

Expected output (truncated):

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  2456  1234 ?        Ss   00:00   0:05 /sbin/init
root       512  1.2  2.3 45678 12345 ?        Ssl  00:00   1:23 /usr/bin/kubelet
root       780  0.5  1.1 34567  9876 ?        Ssl  00:00   0:45 /usr/bin/crio
...
```

### Step 6 — Confirm the SCC in use

```bash
oc get pod hostpid-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Expected output:

```
hostaccess
```

---

## Method 2: Using the `privileged` SCC

> ⚠️ Overkill for just hostPID. Use only if you also need privileged mode, host filesystem, etc.

```bash
oc create serviceaccount priv-sa -n my-namespace
oc adm policy add-scc-to-user privileged -z priv-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-privileged
  namespace: my-namespace
spec:
  serviceAccountName: priv-sa
  hostPID: true
  containers:
    - name: priv-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
        runAsUser: 0
```

---

## Method 3: Custom SCC for Host PID Only

> 🎯 **Best practice** — grants hostPID with the tightest possible restrictions on everything else.

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-hostpid-only
# --- Host access ---
allowHostPID: true
allowHostIPC: false
allowHostNetwork: false
allowHostPorts: false
allowHostDirVolumePlugin: false
# --- Container restrictions ---
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
readOnlyRootFilesystem: false
# --- User / SELinux ---
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
# --- Capabilities ---
requiredDropCapabilities:
  - ALL
allowedCapabilities:
  - SYS_PTRACE       # Needed for process inspection
defaultAddCapabilities: []
# --- Volumes ---
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
# --- Seccomp ---
seccompProfiles:
  - runtime/default
# --- Bind to ServiceAccount ---
users:
  - system:serviceaccount:my-namespace:hostpid-sa
```

```bash
oc apply -f custom-hostpid-only-scc.yaml
```

---

## Method 4: RBAC Approach (hostPID)

### Create a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-hostaccess-scc
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["hostaccess"]
    verbs: ["use"]
```

### Bind it to the ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hostaccess-scc-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-hostaccess-scc
subjects:
  - kind: ServiceAccount
    name: hostpid-sa
    namespace: my-namespace
```

```bash
oc apply -f clusterrole-hostaccess.yaml
oc apply -f rolebinding-hostaccess.yaml
```

---

# Part 2: Host IPC Namespace

---

## Method 1: Using the `hostaccess` SCC (IPC)

```bash
oc create serviceaccount hostipc-sa -n my-namespace
oc adm policy add-scc-to-user hostaccess -z hostipc-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostipc-pod
  namespace: my-namespace
spec:
  serviceAccountName: hostipc-sa
  hostIPC: true
  containers:
    - name: hostipc-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
```

```bash
oc apply -f hostipc-pod.yaml
```

### Verify host IPC access

```bash
# List shared memory segments visible to the container
oc exec hostipc-pod -- ipcs -m

# List semaphores
oc exec hostipc-pod -- ipcs -s

# List message queues
oc exec hostipc-pod -- ipcs -q

# Check /dev/shm (shared memory filesystem)
oc exec hostipc-pod -- ls -la /dev/shm/
```

---

## Method 2: Custom SCC for Host IPC Only

> 🎯 Grants hostIPC with the tightest possible restrictions.

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-hostipc-only
# --- Host access ---
allowHostIPC: true
allowHostPID: false
allowHostNetwork: false
allowHostPorts: false
allowHostDirVolumePlugin: false
# --- Container restrictions ---
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
readOnlyRootFilesystem: false
# --- User / SELinux ---
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
# --- Capabilities ---
requiredDropCapabilities:
  - ALL
allowedCapabilities: []
defaultAddCapabilities: []
# --- Volumes ---
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
# --- Seccomp ---
seccompProfiles:
  - runtime/default
# --- Bind to ServiceAccount ---
users:
  - system:serviceaccount:my-namespace:hostipc-sa
```

```bash
oc apply -f custom-hostipc-only-scc.yaml
```

---

## Method 3: RBAC Approach (hostIPC)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-custom-hostipc-scc
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["custom-hostipc-only"]
    verbs: ["use"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hostipc-scc-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-custom-hostipc-scc
subjects:
  - kind: ServiceAccount
    name: hostipc-sa
    namespace: my-namespace
```

```bash
oc apply -f rbac-hostipc.yaml
```

---

# Part 3: Host PID + IPC Combined

---

### Custom SCC for both hostPID and hostIPC

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-hostpid-ipc
# --- Host access ---
allowHostPID: true
allowHostIPC: true
allowHostNetwork: false
allowHostPorts: false
allowHostDirVolumePlugin: false
# --- Container restrictions ---
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
readOnlyRootFilesystem: false
# --- User / SELinux ---
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
# --- Capabilities ---
requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
allowedCapabilities:
  - SYS_PTRACE
defaultAddCapabilities: []
# --- Volumes ---
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
# --- Bind to ServiceAccount ---
users:
  - system:serviceaccount:my-namespace:hostpid-ipc-sa
```

### Pod using both hostPID and hostIPC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-ipc-pod
  namespace: my-namespace
spec:
  serviceAccountName: hostpid-ipc-sa
  hostPID: true
  hostIPC: true
  containers:
    - name: debug-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
        capabilities:
          add: ["SYS_PTRACE"]
```

```bash
oc create serviceaccount hostpid-ipc-sa -n my-namespace
oc apply -f custom-hostpid-ipc-scc.yaml
oc apply -f hostpid-ipc-pod.yaml
```

### Verify both namespaces are shared

```bash
# Verify hostPID — see all host processes
oc exec hostpid-ipc-pod -- ps aux | head -20

# Verify hostIPC — see host shared memory
oc exec hostpid-ipc-pod -- ipcs -a

# Verify hostIPC — check /dev/shm
oc exec hostpid-ipc-pod -- ls -la /dev/shm/
```

---

## Real-World Examples

### Example 1: Node Process Debugger (strace / gdb)

> Debug a misbehaving process on the node without SSH access.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-debugger
  namespace: debug-ns
spec:
  serviceAccountName: hostpid-sa
  hostPID: true
  nodeName: worker-node-01       # Pin to specific node
  containers:
    - name: debugger
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
        capabilities:
          add: ["SYS_PTRACE"]
```

```bash
# Enter the debugger pod
oc exec -it node-debugger -- bash

# Find the target process
ps aux | grep kubelet

# Trace system calls of the kubelet
strace -p <kubelet-pid> -f -e trace=network

# Read /proc info for any host process
cat /proc/<pid>/status
cat /proc/<pid>/environ
cat /proc/<pid>/cmdline | tr '\0' ' '
ls -la /proc/<pid>/fd/
```

---

### Example 2: Shared Memory Database Sidecar

> A sidecar that needs to access a database's shared memory on the host for monitoring.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-shm-monitor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-shm-monitor
  template:
    metadata:
      labels:
        app: db-shm-monitor
    spec:
      serviceAccountName: hostipc-sa
      hostIPC: true
      containers:
        - name: shm-monitor
          image: registry.access.redhat.com/ubi9/ubi:latest
          command:
            - /bin/sh
            - -c
            - |
              while true; do
                echo "=== Shared Memory Segments ==="
                ipcs -m
                echo "=== /dev/shm Contents ==="
                ls -la /dev/shm/
                echo "=== Semaphores ==="
                ipcs -s
                sleep 30
              done
```

---

### Example 3: Process Monitoring DaemonSet

> Monitor all processes across every node in the cluster.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: process-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: proc-monitor
  template:
    metadata:
      labels:
        app: proc-monitor
    spec:
      serviceAccountName: hostpid-sa
      hostPID: true
      containers:
        - name: monitor
          image: registry.access.redhat.com/ubi9/ubi:latest
          command:
            - /bin/sh
            - -c
            - |
              while true; do
                echo "=== $(hostname) - $(date) ==="
                echo "--- Top CPU Processes ---"
                ps aux --sort=-%cpu | head -10
                echo "--- Top Memory Processes ---"
                ps aux --sort=-%mem | head -10
                echo "--- Zombie Processes ---"
                ps aux | awk '$8 ~ /Z/ {print}'
                sleep 60
              done
```

---

### Example 4: Signal Sender (Graceful Process Restart)

> Send signals to host processes from within a container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: signal-sender
  namespace: ops
spec:
  serviceAccountName: hostpid-sa
  hostPID: true
  containers:
    - name: signaler
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
```

```bash
oc exec -it signal-sender -- bash

# Find the target process
ps aux | grep nginx

# Send SIGHUP to reload config
kill -HUP <nginx-pid>

# Send SIGUSR1 to reopen log files
kill -USR1 <nginx-pid>

# Send SIGTERM for graceful shutdown
kill -TERM <pid>
```

---

### Example 5: nsenter — Enter Any Process Namespace

> The most powerful use of hostPID — enter any process's full namespace context.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nsenter-pod
  namespace: debug-ns
spec:
  serviceAccountName: priv-sa
  hostPID: true
  containers:
    - name: nsenter
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
        runAsUser: 0
```

```bash
oc exec -it nsenter-pod -- bash

# Enter PID 1's full namespace (host root shell)
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash

# Enter only the mount namespace of a specific process
nsenter --target <pid> --mount -- ls /

# Enter only the network namespace of a specific process
nsenter --target <pid> --net -- ip addr show

# Enter a specific container's namespace from the host
# Find the container's PID first
crictl inspect <container-id> | grep pid
nsenter --target <container-pid> --all -- /bin/sh
```

---

### Example 6: IPC Between Pods via Host Shared Memory

> Two pods communicating through host shared memory.

**Writer Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shm-writer
  namespace: my-namespace
spec:
  serviceAccountName: hostipc-sa
  hostIPC: true
  containers:
    - name: writer
      image: registry.access.redhat.com/ubi9/ubi:latest
      command:
        - /bin/sh
        - -c
        - |
          # Write data to host shared memory
          echo "Hello from writer pod at $(date)" > /dev/shm/shared_data
          echo "Data written to /dev/shm/shared_data"
          sleep infinity
```

**Reader Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shm-reader
  namespace: my-namespace
spec:
  serviceAccountName: hostipc-sa
  hostIPC: true
  containers:
    - name: reader
      image: registry.access.redhat.com/ubi9/ubi:latest
      command:
        - /bin/sh
        - -c
        - |
          # Read data from host shared memory
          while true; do
            echo "Reading: $(cat /dev/shm/shared_data 2>/dev/null || echo 'No data yet')"
            sleep 5
          done
```

---

## What Host PID / IPC Exposes

### With `hostPID: true`, a container CAN:

```bash
# See ALL processes on the node
ps aux

# Read environment variables of any process (may contain secrets!)
cat /proc/<pid>/environ | tr '\0' '\n'

# Read command-line arguments of any process
cat /proc/<pid>/cmdline | tr '\0' ' '

# Read file descriptors of any process
ls -la /proc/<pid>/fd/

# Read memory maps
cat /proc/<pid>/maps

# Read process status and limits
cat /proc/<pid>/status
cat /proc/<pid>/limits

# Send signals to any process (if running as root)
kill -TERM <pid>
kill -9 <pid>

# Trace processes (with SYS_PTRACE capability)
strace -p <pid>

# Enter any process's namespace (with nsenter + privileges)
nsenter --target <pid> --all -- /bin/sh
```

### With `hostIPC: true`, a container CAN:

```bash
# List all shared memory segments
ipcs -m

# List all semaphores
ipcs -s

# List all message queues
ipcs -q

# Read/write host shared memory
ls -la /dev/shm/
cat /dev/shm/<segment_name>

# Attach to existing shared memory segments
# (programmatically via shmget/shmat)

# Potentially read sensitive data from database shared buffers
# (PostgreSQL, Oracle, etc. use shared memory heavily)
```

### Comparison of risk levels

| Capability | Risk Level | What's Exposed |
|:--|:--|:--|
| `hostPID` only | 🟠 HIGH | Process list, env vars, cmdline, /proc |
| `hostPID` + `runAsUser: 0` | 🔴 CRITICAL | + Signal processes, read all /proc entries |
| `hostPID` + `SYS_PTRACE` | 🔴 CRITICAL | + Trace/debug any process, read memory |
| `hostPID` + `privileged` | 🔴 CRITICAL | + nsenter into any namespace (full escape) |
| `hostIPC` only | 🟡 MEDIUM | Shared memory, semaphores, message queues |
| `hostIPC` + `runAsUser: 0` | 🟠 HIGH | + Read/write all IPC segments |
| `hostPID` + `hostIPC` | 🔴 CRITICAL | Full process + IPC visibility |

---

## Debugging & Troubleshooting

### Verify hostPID is active

```bash
# Should show host processes (systemd, kubelet, crio, etc.)
oc exec <pod> -- ps aux | head -20

# PID 1 should be systemd/init, NOT your container's entrypoint
oc exec <pod> -- cat /proc/1/cmdline | tr '\0' ' '
# Expected: /sbin/init  or  /usr/lib/systemd/systemd
```

### Verify hostIPC is active

```bash
# Should show host IPC segments (if any exist)
oc exec <pod> -- ipcs -a

# Check host shared memory
oc exec <pod> -- ls -la /dev/shm/

# Compare with node's IPC
oc debug node/<node-name> -- chroot /host ipcs -a
```

### Check SCC assignment

```bash
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

### Check which SCCs allow hostPID / hostIPC

```bash
# SCCs allowing hostPID
oc get scc -o json | \
  jq -r '.items[] | select(.allowHostPID == true) | .metadata.name'

# SCCs allowing hostIPC
oc get scc -o json | \
  jq -r '.items[] | select(.allowHostIPC == true) | .metadata.name'
```

### Inspect the SCC details

```bash
oc describe scc hostaccess
oc get scc hostaccess -o yaml
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|:--|:--|:--|
| `unable to validate against any security context constraint` | SA lacks an SCC with `allowHostPID` or `allowHostIPC` | Grant `hostaccess` or a custom SCC |
| `hostPID: Invalid value: true` | `restricted` SCC rejects `hostPID` | `oc adm policy add-scc-to-user hostaccess -z <sa> -n <ns>` |
| `hostIPC: Invalid value: true` | `restricted` SCC rejects `hostIPC` | Grant an SCC with `allowHostIPC: true` |
| `ps aux` shows only container process | `hostPID: true` not set or SCC not applied | Verify pod spec has `hostPID: true` and check SCC annotation |
| `ipcs` shows empty output | No IPC segments on the host, or `hostIPC` not active | Verify with `oc debug node/<node> -- chroot /host ipcs -a` |
| `strace: attach: Operation not permitted` | Missing `SYS_PTRACE` capability | Add `capabilities: add: ["SYS_PTRACE"]` and allow in SCC |
| `kill: Operation not permitted` | Container not running as root | Set `runAsUser: 0` and use `anyuid` or `hostaccess` SCC |
| `nsenter: failed to execute` | Missing `privileged: true` or insufficient capabilities | Use `privileged` SCC for full nsenter access |

---

## Detection & Auditing

### Find all pods using hostPID

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostPID == true) | [.metadata.namespace, .metadata.name, .spec.nodeName] | @tsv' | \
  column -t
```

### Find all pods using hostIPC

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostIPC == true) | [.metadata.namespace, .metadata.name, .spec.nodeName] | @tsv' | \
  column -t
```

### Find pods using hostPID OR hostIPC OR both

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostPID == true or .spec.hostIPC == true) | [
    .metadata.namespace,
    .metadata.name,
    "hostPID=\(.spec.hostPID // false)",
    "hostIPC=\(.spec.hostIPC // false)",
    .metadata.annotations["openshift.io/scc"]
  ] | @tsv' | \
  column -t
```

### Find ServiceAccounts with dangerous SCCs

```bash
for scc in hostaccess privileged; do
  echo "=== SCC: $scc ==="
  oc get scc $scc -o jsonpath='{.users}' 2>/dev/null | jq -r '.[]'
  echo ""
done
```

### Audit custom SCCs for hostPID / hostIPC

```bash
oc get scc -o json | \
  jq -r '.items[] | select(.allowHostPID == true or .allowHostIPC == true) | [
    .metadata.name,
    "hostPID=\(.allowHostPID)",
    "hostIPC=\(.allowHostIPC)"
  ] | @tsv' | \
  column -t
```

### Full cluster security audit

```bash
echo "=== Pods with Host Namespace Access ==="
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostPID == true or .spec.hostIPC == true or .spec.hostNetwork == true) | [
    .metadata.namespace,
    .metadata.name,
    "PID=\(.spec.hostPID // false)",
    "IPC=\(.spec.hostIPC // false)",
    "NET=\(.spec.hostNetwork // false)",
    "SCC=\(.metadata.annotations["openshift.io/scc"])"
  ] | @tsv' | \
  column -t
```

---

## Security Best Practices

- ✅ **Create custom SCCs** — never use `hostaccess` or `privileged` when you only need `hostPID` or `hostIPC`
- ✅ **Scope to specific ServiceAccount + namespace** — never grant cluster-wide
- ✅ **Use a dedicated ServiceAccount** — never grant to the `default` SA
- ✅ **Drop all unnecessary capabilities:**

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["SYS_PTRACE"]   # Only if you need process tracing
```

- ✅ **Avoid combining hostPID + privileged** — this enables full container escape via `nsenter`
- ✅ **Use Pod Security Admission (PSA) labels** to enforce restrictions at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: baseline    # Blocks hostPID/hostIPC
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

- ✅ **Use Kyverno / OPA Gatekeeper** as a second line of defense:

```yaml
# Kyverno policy to block hostPID and hostIPC
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-host-pid-ipc
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-host-pid
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "hostPID is not allowed."
        pattern:
          spec:
            =(hostPID): false
    - name: deny-host-ipc
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "hostIPC is not allowed."
        pattern:
          spec:
            =(hostIPC): false
```

- ✅ **Consider alternatives:**

| Need | Alternative to hostPID/hostIPC |
|:--|:--|
| Debug a node process | Use `oc debug node/<node>` instead |
| Monitor processes | Use Prometheus + node-exporter |
| Share memory between pods | Use `emptyDir` with `medium: Memory` |
| Send signals to processes | Use Kubernetes lifecycle hooks |
| Inspect container processes | Use `oc exec` or `crictl` on the node |

- ✅ **Audit regularly and remove grants when no longer needed:**

```bash
oc adm policy remove-scc-from-user hostaccess -z hostpid-sa -n my-namespace
oc delete serviceaccount hostpid-sa -n my-namespace
```

---

## Quick Cheat Sheet

```bash
# === SETUP (hostPID) ===
oc create serviceaccount hostpid-sa -n my-namespace
oc adm policy add-scc-to-user hostaccess -z hostpid-sa -n my-namespace
# Pod spec: hostPID: true

# === SETUP (hostIPC) ===
oc create serviceaccount hostipc-sa -n my-namespace
oc adm policy add-scc-to-user hostaccess -z hostipc-sa -n my-namespace
# Pod spec: hostIPC: true

# === VERIFY ===
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
oc exec <pod> -- ps aux                  # hostPID: should see host processes
oc exec <pod> -- ipcs -a                 # hostIPC: should see host IPC segments
oc exec <pod> -- cat /proc/1/cmdline     # hostPID: should show systemd/init

# === AUDIT ===
oc get pods -A -o json | jq -r '.items[] | select(.spec.hostPID == true) | [.metadata.namespace, .metadata.name] | @tsv'
oc get pods -A -o json | jq -r '.items[] | select(.spec.hostIPC == true) | [.metadata.namespace, .metadata.name] | @tsv'

# === CLEANUP ===
oc adm policy remove-scc-from-user hostaccess -z hostpid-sa -n my-namespace
oc delete serviceaccount hostpid-sa -n my-namespace
```

---

> 📌 **Key Takeaway:** There is no built-in SCC for hostPID-only or hostIPC-only — always create a **custom SCC** scoped to the minimum access needed. Avoid combining `hostPID` with `privileged` mode, as this enables full container escape via `nsenter`. Use `oc debug node/<node>` as a safer alternative for one-off debugging.
