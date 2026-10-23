
# 🌐 Using Host Network in OpenShift SCC

> **⚠️ Security Warning:** Enabling host networking removes network isolation between the container and the node. The container shares the node's network stack, IP address, and can see all network traffic. Use only when absolutely necessary.

---

## 📋 Table of Contents

- [What is Host Networking?](#what-is-host-networking)
- [How Host Network Differs from Pod Network](#how-host-network-differs-from-pod-network)
- [SCCs That Allow Host Networking](#sccs-that-allow-host-networking)
- [Method 1: Using the Built-in `hostnetwork` SCC](#method-1-using-the-built-in-hostnetwork-scc)
- [Method 2: Using the `hostnetwork-v2` SCC](#method-2-using-the-hostnetwork-v2-scc)
- [Method 3: Using the `privileged` SCC](#method-3-using-the-privileged-scc)
- [Method 4: Custom SCC for Host Network Only](#method-4-custom-scc-for-host-network-only)
- [Method 5: RBAC Approach (ClusterRole + RoleBinding)](#method-5-rbac-approach-clusterrole--rolebinding)
- [Real-World Examples](#real-world-examples)
- [Host Network + Host Ports](#host-network--host-ports)
- [DNS Considerations](#dns-considerations)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Common Errors and Fixes](#common-errors-and-fixes)
- [Security Implications](#security-implications)
- [Security Best Practices](#security-best-practices)

---

## What is Host Networking?

When `hostNetwork: true` is set on a pod, the container **shares the node's network namespace** instead of getting its own isolated network stack.

```
┌─────────────────────────────────────────────────────┐
│                   Node (Host)                       │
│                                                     │
│   IP: 10.0.1.50                                     │
│   Interfaces: eth0, ens192, lo                      │
│                                                     │
│   ┌──────────────────────┐  ┌────────────────────┐  │
│   │  Normal Pod          │  │  hostNetwork Pod   │  │
│   │  ┌────────────────┐  │  │                    │  │
│   │  │ veth interface │  │  │  Uses node's eth0  │  │
│   │  │ IP: 10.128.x.x │  │  │  IP: 10.0.1.50   │  │
│   │  │ (Pod Network)  │  │  │  (Host Network)   │  │
│   │  └────────────────┘  │  │                    │  │
│   └──────────────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## How Host Network Differs from Pod Network

| Feature | Pod Network (default) | Host Network |
|:--|:--|:--|
| **IP Address** | Unique pod IP from cluster CIDR | Node's IP address |
| **Network Namespace** | Isolated per pod | Shared with host |
| **Port Binding** | Pod ports (no conflict) | Directly on node ports (can conflict) |
| **DNS** | Cluster DNS (`cluster.local`) | Node's `/etc/resolv.conf` |
| **Network Policies** | ✅ Enforced | ❌ Bypassed |
| **Service Discovery** | Via ClusterIP / DNS | Must use node IP or manual config |
| **Traffic Visibility** | Own traffic only | All node traffic visible |
| **OVN-Kubernetes Isolation** | ✅ Enforced | ❌ Bypassed |
| **Performance** | Slight overhead (veth + bridge) | Native performance (no overlay) |

---

## SCCs That Allow Host Networking

| SCC | `allowHostNetwork` | `allowHostPorts` | Other Access |
|:--|:--|:--|:--|
| `restricted` | ❌ `false` | ❌ `false` | None |
| `nonroot` | ❌ `false` | ❌ `false` | None |
| `anyuid` | ❌ `false` | ❌ `false` | Root UID only |
| `hostnetwork` | ✅ `true` | ✅ `true` | Host network only |
| `hostnetwork-v2` | ✅ `true` | ✅ `true` | Host network only (updated) |
| `hostaccess` | ✅ `true` | ✅ `true` | + Host dirs, PID, IPC |
| `privileged` | ✅ `true` | ✅ `true` | Everything |

> 💡 **Rule of thumb:** Use `hostnetwork` or `hostnetwork-v2` if you only need host networking. Avoid `privileged` or `hostaccess` unless you need more.

---

## Method 1: Using the Built-in `hostnetwork` SCC

> ✅ **Recommended** — grants host network access without additional host privileges.

### Step 1 — Create a ServiceAccount

```bash
oc create serviceaccount hostnet-sa -n my-namespace
```

### Step 2 — Grant the `hostnetwork` SCC

```bash
oc adm policy add-scc-to-user hostnetwork -z hostnet-sa -n my-namespace
```

### Step 3 — Verify

```bash
oc adm policy who-can use scc hostnetwork
```

### Step 4 — Deploy a pod with host networking

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod
  namespace: my-namespace
spec:
  serviceAccountName: hostnet-sa
  hostNetwork: true
  containers:
    - name: hostnet-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
```

```bash
oc apply -f hostnet-pod.yaml
```

### Step 5 — Verify host network is active

```bash
# Check the pod IP matches the node IP
oc get pod hostnet-pod -o wide

# Verify inside the container
oc exec hostnet-pod -- ip addr show
oc exec hostnet-pod -- hostname -I
```

Expected: The pod IP will be the **node's IP**, not a pod network IP.

### Step 6 — Confirm the SCC in use

```bash
oc get pod hostnet-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Expected output:

```
hostnetwork
```

---

## Method 2: Using the `hostnetwork-v2` SCC

> 🆕 Available in OpenShift 4.11+. Updated security defaults with the same host network access.

```bash
oc create serviceaccount hostnet-v2-sa -n my-namespace
oc adm policy add-scc-to-user hostnetwork-v2 -z hostnet-v2-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-v2-pod
  namespace: my-namespace
spec:
  serviceAccountName: hostnet-v2-sa
  hostNetwork: true
  containers:
    - name: hostnet-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
```

### Differences between `hostnetwork` and `hostnetwork-v2`

| Feature | `hostnetwork` | `hostnetwork-v2` |
|:--|:--|:--|
| `seccompProfiles` | Not set | `runtime/default` |
| `allowPrivilegeEscalation` | `true` | `false` |
| `requiredDropCapabilities` | `- KILL; - MKNOD; - SETUID; - SETGID` | `- ALL` |

> 💡 Prefer `hostnetwork-v2` on OpenShift 4.11+ for tighter defaults.

---

## Method 3: Using the `privileged` SCC

> ⚠️ **Overkill for just host networking.** Use only if you also need host PID, host IPC, privileged mode, etc.

```bash
oc create serviceaccount priv-sa -n my-namespace
oc adm policy add-scc-to-user privileged -z priv-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-privileged
  namespace: my-namespace
spec:
  serviceAccountName: priv-sa
  hostNetwork: true
  containers:
    - name: priv-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
```

---

## Method 4: Custom SCC for Host Network Only

> 🎯 **Best for:** Granting host network with the tightest possible restrictions on everything else.

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-hostnetwork-only
# --- Host access ---
allowHostNetwork: true
allowHostPorts: true
allowHostPID: false
allowHostIPC: false
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
  - NET_BIND_SERVICE
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
  - system:serviceaccount:my-namespace:hostnet-sa
```

```bash
oc apply -f custom-hostnetwork-only-scc.yaml
```

> 💡 This SCC allows host networking but **drops all capabilities** except `NET_BIND_SERVICE`, enforces non-root, and blocks host filesystem access.

---

## Method 5: RBAC Approach (ClusterRole + RoleBinding)

> 🔑 Modern, auditable way to manage SCC access.

### Create a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-hostnetwork-scc
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["hostnetwork-v2"]
    verbs: ["use"]
```

### Bind it to the ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hostnetwork-scc-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-hostnetwork-scc
subjects:
  - kind: ServiceAccount
    name: hostnet-sa
    namespace: my-namespace
```

```bash
oc apply -f clusterrole-hostnetwork.yaml
oc apply -f rolebinding-hostnetwork.yaml
```

---

## Real-World Examples

### Example 1: NGINX Ingress Controller with Host Network

> Ingress controllers often need host networking to bind to ports 80/443 directly on the node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress
  namespace: ingress-ns
spec:
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      serviceAccountName: hostnet-sa
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: nginx-ingress
          image: registry.k8s.io/ingress-nginx/controller:v1.9.4
          ports:
            - name: http
              containerPort: 80
              hostPort: 80
            - name: https
              containerPort: 443
              hostPort: 443
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
```

> ⚠️ Note `dnsPolicy: ClusterFirstWithHostNet` — this is **critical** for DNS to work correctly (see [DNS Considerations](#dns-considerations)).

---

### Example 2: Network Monitoring / Packet Capture (DaemonSet)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: network-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: net-monitor
  template:
    metadata:
      labels:
        app: net-monitor
    spec:
      serviceAccountName: hostnet-sa
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: tcpdump
          image: docker.io/nicolaka/netshoot:latest
          command:
            - /bin/sh
            - -c
            - "tcpdump -i any -w /captures/$(hostname)_$(date +%Y%m%d).pcap"
          securityContext:
            capabilities:
              add: ["NET_RAW", "NET_ADMIN"]
          volumeMounts:
            - name: captures
              mountPath: /captures
      volumes:
        - name: captures
          emptyDir: {}
```

---

### Example 3: MetalLB Speaker (BGP/ARP on Host Network)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: metallb-speaker
  namespace: metallb-system
spec:
  selector:
    matchLabels:
      app: metallb-speaker
  template:
    metadata:
      labels:
        app: metallb-speaker
    spec:
      serviceAccountName: metallb-speaker-sa
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      terminationGracePeriodSeconds: 2
      containers:
        - name: speaker
          image: quay.io/metallb/speaker:v0.13.12
          ports:
            - name: monitoring
              containerPort: 7472
              hostPort: 7472
          env:
            - name: METALLB_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          securityContext:
            capabilities:
              add: ["NET_RAW"]
              drop: ["ALL"]
```

---

### Example 4: Prometheus Node Exporter

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      serviceAccountName: hostnet-sa
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: node-exporter
          image: quay.io/prometheus/node-exporter:v1.7.0
          ports:
            - name: metrics
              containerPort: 9100
              hostPort: 9100
          args:
            - "--path.rootfs=/host"
          volumeMounts:
            - name: rootfs
              mountPath: /host
              readOnly: true
      volumes:
        - name: rootfs
          hostPath:
            path: /
```

---

### Example 5: CoreDNS / Custom DNS Server

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-dns
  namespace: dns-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: custom-dns
  template:
    metadata:
      labels:
        app: custom-dns
    spec:
      serviceAccountName: hostnet-sa
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      containers:
        - name: coredns
          image: docker.io/coredns/coredns:1.11.1
          ports:
            - name: dns
              containerPort: 53
              hostPort: 53
              protocol: UDP
            - name: dns-tcp
              containerPort: 53
              hostPort: 53
              protocol: TCP
          volumeMounts:
            - name: config
              mountPath: /etc/coredns
      volumes:
        - name: config
          configMap:
            name: coredns-config
```

---

## Host Network + Host Ports

When using `hostNetwork: true`, you can also bind specific ports on the node.

### Using `hostPort` (with or without `hostNetwork`)

```yaml
spec:
  containers:
    - name: app
      ports:
        # With hostNetwork: true, containerPort IS the hostPort
        - containerPort: 8080
          hostPort: 8080    # Binds directly on the node
          protocol: TCP
```

### Port conflict prevention

```bash
# Check which ports are already in use on a node
oc debug node/<node-name> -- chroot /host ss -tlnp

# Check for hostPort conflicts across pods
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].ports[]?.hostPort != null) | [.metadata.namespace, .metadata.name, (.spec.containers[].ports[] | select(.hostPort != null) | "\(.hostPort)/\(.protocol)")] | @tsv' | \
  column -t
```

> ⚠️ **Important:** With `hostNetwork: true`, if two pods on the same node try to bind the same port, the second pod will **fail to start**. Use a `DaemonSet` to ensure one pod per node.

---

## DNS Considerations

> 🔴 **This is the #1 gotcha with host networking.**

When `hostNetwork: true`, the pod uses the **node's DNS** (`/etc/resolv.conf`) instead of the cluster DNS. This means:

- ❌ `my-service.my-namespace.svc.cluster.local` **will NOT resolve**
- ❌ Kubernetes service discovery **breaks**
- ❌ Short names like `my-service` **won't work**

### Fix: Use `dnsPolicy: ClusterFirstWithHostNet`

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet   # ← ALWAYS SET THIS
```

### DNS Policy comparison

| `dnsPolicy` | Behavior with `hostNetwork: true` |
|:--|:--|
| `Default` | Uses node's `/etc/resolv.conf` — ❌ no cluster DNS |
| `ClusterFirst` | Ignored when `hostNetwork: true` — falls back to `Default` |
| `ClusterFirstWithHostNet` | ✅ Uses cluster DNS first, then node DNS as fallback |
| `None` | Manual DNS config via `dnsConfig` |

### Verify DNS resolution

```bash
# Inside a hostNetwork pod
oc exec hostnet-pod -- cat /etc/resolv.conf

# Test cluster DNS resolution
oc exec hostnet-pod -- nslookup kubernetes.default.svc.cluster.local

# Test external DNS resolution
oc exec hostnet-pod -- nslookup google.com
```

### Manual DNS config (advanced)

```yaml
spec:
  hostNetwork: true
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 172.30.0.10          # Cluster DNS service IP
      - 10.0.0.2             # Node DNS
    searches:
      - my-namespace.svc.cluster.local
      - svc.cluster.local
      - cluster.local
    options:
      - name: ndots
        value: "5"
```

---

## Debugging & Troubleshooting

### Verify host network is active

```bash
# Pod IP should match node IP
oc get pod hostnet-pod -o wide
oc get nodes -o wide

# Compare interfaces
oc exec hostnet-pod -- ip addr show
oc debug node/<node-name> -- chroot /host ip addr show
```

### Check SCC assignment

```bash
oc get pod hostnet-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

### Check which SCC allows hostNetwork

```bash
oc get scc -o json | \
  jq -r '.items[] | select(.allowHostNetwork == true) | .metadata.name'
```

### Check for port conflicts

```bash
# List all hostPort usage across the cluster
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostNetwork == true) | [.metadata.namespace, .metadata.name, .spec.nodeName, (.spec.containers[].ports[]? | "\(.containerPort)/\(.protocol // "TCP")")] | @tsv' | \
  sort | column -t
```

### Test network connectivity

```bash
oc exec hostnet-pod -- curl -s --connect-timeout 3 http://10.128.0.1:8080
oc exec hostnet-pod -- ping -c 3 <other-node-ip>
oc exec hostnet-pod -- ss -tlnp
```

### Check OVN-Kubernetes behavior

```bash
# Host network pods bypass OVN — verify
oc exec hostnet-pod -- ip route show
oc exec hostnet-pod -- iptables -L -n -t nat
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|:--|:--|:--|
| `unable to validate against any security context constraint` | SA lacks an SCC with `allowHostNetwork: true` | `oc adm policy add-scc-to-user hostnetwork -z <sa> -n <ns>` |
| `hostNetwork: Invalid value: true` | `restricted` SCC rejects `hostNetwork` | Assign `hostnetwork` or `hostnetwork-v2` SCC |
| Pod stuck in `Pending` | Port conflict — another pod uses the same `hostPort` on the node | Change the port or use a DaemonSet (1 pod per node) |
| DNS resolution fails for `*.svc.cluster.local` | Missing `dnsPolicy: ClusterFirstWithHostNet` | Add `dnsPolicy: ClusterFirstWithHostNet` to pod spec |
| `bind: address already in use` | Node process or another pod already using the port | Check with `ss -tlnp` on the node; pick a different port |
| Network policies not working | Host network pods bypass NetworkPolicy | Use firewall rules on the node instead, or avoid host networking |
| Service not reachable from hostNetwork pod | Pod uses node DNS, can't resolve cluster services | Set `dnsPolicy: ClusterFirstWithHostNet` |

---

## Security Implications

> Understanding what you expose when enabling host networking.

### What a hostNetwork pod CAN do

```bash
# See all node network interfaces
ip addr show

# See all connections on the node
ss -tlnp

# Access the cloud metadata API
curl http://169.254.169.254/latest/meta-data/

# Access node-local services (kubelet, etcd, etc.)
curl -k https://localhost:10250/pods
curl -k https://localhost:2379/version

# Sniff network traffic (if NET_RAW capability is granted)
tcpdump -i eth0

# Bypass NetworkPolicy (host network is outside OVN)
curl http://<any-pod-ip>:<port>

# Reach other nodes directly
curl http://<other-node-ip>:<port>
```

### What a hostNetwork pod CANNOT do (with `hostnetwork` SCC)

- ❌ Access host filesystem (no `hostPath` volumes)
- ❌ See host processes (no `hostPID`)
- ❌ Run as privileged (no `privileged: true`)
- ❌ Use shared memory with host (no `hostIPC`)

---

## Security Best Practices

- ✅ **Use `hostnetwork-v2` over `hostnetwork`** on OpenShift 4.11+ for tighter defaults
- ✅ **Scope SCC grants to a specific ServiceAccount + namespace** — never cluster-wide
- ✅ **Always set `dnsPolicy: ClusterFirstWithHostNet`** to preserve cluster DNS
- ✅ **Use DaemonSets** for hostNetwork pods to avoid port conflicts (1 pod per node)
- ✅ **Use a custom SCC** to grant host network without other host access
- ✅ **Drop all unnecessary capabilities:**

```yaml
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]   # Only if binding to ports < 1024
```

- ✅ **Consider alternatives before using host networking:**

| Need | Alternative to hostNetwork |
|:--|:--|
| Expose service externally | Use `LoadBalancer` or `NodePort` Service |
| Bind to port 80/443 | Use OpenShift Router / Ingress |
| High-performance networking | Use SR-IOV or Multus CNI |
| Access node network | Use `hostPort` without full `hostNetwork` |

- ✅ **Audit hostNetwork pods regularly:**

```bash
oc get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.hostNetwork == true) | [.metadata.namespace, .metadata.name, .spec.nodeName] | @tsv' | \
  column -t
```

- ✅ **Remove SCC grants when no longer needed:**

```bash
oc adm policy remove-scc-from-user hostnetwork -z hostnet-sa -n my-namespace
```

---

## Quick Cheat Sheet

```bash
# === SETUP ===
oc create serviceaccount hostnet-sa -n my-namespace
oc adm policy add-scc-to-user hostnetwork-v2 -z hostnet-sa -n my-namespace

# === DEPLOY ===
# Always include these two lines in your pod spec:
#   hostNetwork: true
#   dnsPolicy: ClusterFirstWithHostNet

# === VERIFY ===
oc get pod <pod> -o wide                                                    # Pod IP = Node IP
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'    # SCC = hostnetwork-v2
oc exec <pod> -- ip addr show                                               # Node interfaces visible
oc exec <pod> -- nslookup kubernetes.default.svc.cluster.local              # Cluster DNS works

# === AUDIT ===
oc get pods -A -o json | jq -r '.items[] | select(.spec.hostNetwork == true) | [.metadata.namespace, .metadata.name] | @tsv'

# === CLEANUP ===
oc adm policy remove-scc-from-user hostnetwork-v2 -z hostnet-sa -n my-namespace
oc delete serviceaccount hostnet-sa -n my-namespace
```

---

> 📌 **Key Takeaway:** Always pair `hostNetwork: true` with `dnsPolicy: ClusterFirstWithHostNet`. Use the `hostnetwork-v2` SCC for the tightest defaults. Consider alternatives like `NodePort`, `LoadBalancer`, or SR-IOV before reaching for host networking.
