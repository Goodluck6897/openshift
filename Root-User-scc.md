
# 🔑 Running as Root User in OpenShift

> **⚠️ Important:** By default, OpenShift **does not allow** containers to run as root (UID 0). This is enforced by the `restricted` / `restricted-v2` Security Context Constraint (SCC). You must explicitly grant permissions to run as root.

---

## 📋 Table of Contents

- [Why OpenShift Blocks Root by Default](#why-openshift-blocks-root-by-default)
- [Quick Reference: SCC Options for Root Access](#quick-reference-scc-options-for-root-access)
- [Method 1: Using the `anyuid` SCC (Recommended)](#method-1-using-the-anyuid-scc-recommended)
- [Method 2: Using the `privileged` SCC (Full Host Access)](#method-2-using-the-privileged-scc-full-host-access)
- [Method 3: Custom SCC for Root Only](#method-3-custom-scc-for-root-only)
- [Method 4: RBAC Approach (ClusterRole + RoleBinding)](#method-4-rbac-approach-clusterrole--rolebinding)
- [Real-World Examples](#real-world-examples)
- [Debugging & Troubleshooting](#debugging--troubleshooting)
- [Common Errors and Fixes](#common-errors-and-fixes)
- [Security Best Practices](#security-best-practices)

---

## Why OpenShift Blocks Root by Default

OpenShift assigns a **random UID** from a namespace-specific range to every container. This prevents:

- Containers from modifying host filesystems
- Privilege escalation attacks
- Containers from binding to privileged ports (< 1024)

```bash
# Check the UID range assigned to a namespace
oc describe namespace my-namespace | grep "sa.scc.uid-range"
```

Example output:

```
openshift.io/sa.scc.uid-range=1000700000/10000
```

> This means containers in `my-namespace` will run with UIDs between `1000700000` and `1000710000`.

---

## Quick Reference: SCC Options for Root Access

| SCC | Runs as Root? | Host Access? | Use Case |
|:--|:--|:--|:--|
| `restricted` | ❌ No | ❌ No | Default — random UID |
| `nonroot` | ❌ No | ❌ No | Any non-root UID |
| `anyuid` | ✅ Yes | ❌ No | Root inside container only |
| `hostaccess` | ✅ Yes | ✅ Yes | Root + host filesystem |
| `privileged` | ✅ Yes | ✅ Yes | Full unrestricted access |

> 💡 **Rule of thumb:** Use `anyuid` if you just need root. Use `privileged` only if you also need host-level access.

---

## Method 1: Using the `anyuid` SCC (Recommended)

> ✅ **Best for:** Running legacy images or apps that require UID 0 but don't need host access.

### Step 1 — Create a ServiceAccount

```bash
oc create serviceaccount root-sa -n my-namespace
```

### Step 2 — Grant `anyuid` SCC to the ServiceAccount

```bash
oc adm policy add-scc-to-user anyuid -z root-sa -n my-namespace
```

### Step 3 — Verify

```bash
oc adm policy who-can use scc anyuid
```

### Step 4 — Deploy a pod as root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: my-namespace
spec:
  serviceAccountName: root-sa
  containers:
    - name: root-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
```

```bash
oc apply -f root-pod.yaml
```

### Step 5 — Verify the container is running as root

```bash
oc exec root-pod -- id
```

Expected output:

```
uid=0(root) gid=0(root) groups=0(root)
```

### Step 6 — Confirm the SCC in use

```bash
oc get pod root-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Expected output:

```
anyuid
```

---

## Method 2: Using the `privileged` SCC (Full Host Access)

> ⚠️ Use only when you need root **and** host-level access (networking, devices, filesystems).

```bash
oc create serviceaccount priv-sa -n my-namespace
oc adm policy add-scc-to-user privileged -z priv-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-root-pod
  namespace: my-namespace
spec:
  serviceAccountName: priv-sa
  containers:
    - name: priv-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
        runAsUser: 0
```

```bash
oc apply -f privileged-root-pod.yaml
```

---

## Method 3: Custom SCC for Root Only

> 🎯 **Best for:** Allowing root but restricting everything else (no host network, no host PID, etc.)

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-root-only
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPorts: false
allowHostPID: false
allowHostIPC: false
allowHostDirVolumePlugin: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
allowedCapabilities: []
defaultAddCapabilities: []
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
users:
  - system:serviceaccount:my-namespace:root-sa
```

```bash
oc apply -f custom-root-only-scc.yaml
```

---

## Method 4: RBAC Approach (ClusterRole + RoleBinding)

> 🔑 Modern, auditable way to manage SCC access via RBAC.

### Create a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-anyuid-scc
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["anyuid"]
    verbs: ["use"]
```

### Bind it to the ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: anyuid-scc-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-anyuid-scc
subjects:
  - kind: ServiceAccount
    name: root-sa
    namespace: my-namespace
```

```bash
oc apply -f clusterrole-anyuid.yaml
oc apply -f rolebinding-anyuid.yaml
```

---

## Real-World Examples

### Example 1: Running NGINX as Root

> NGINX's default image requires root to bind to port 80.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-root
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: root-sa
      containers:
        - name: nginx
          image: docker.io/library/nginx:latest
          ports:
            - containerPort: 80
          securityContext:
            runAsUser: 0
```

### Example 2: Running a Database (MySQL) as Root

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-root
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      serviceAccountName: root-sa
      containers:
        - name: mysql
          image: docker.io/library/mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          ports:
            - containerPort: 3306
          securityContext:
            runAsUser: 0
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
```

### Example 3: Running a Jenkins Agent as Root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: ci-cd
spec:
  serviceAccountName: root-sa
  containers:
    - name: jenkins-agent
      image: jenkins/inbound-agent:latest
      securityContext:
        runAsUser: 0
      env:
        - name: JENKINS_URL
          value: "http://jenkins-master:8080"
        - name: JENKINS_SECRET
          valueFrom:
            secretKeyRef:
              name: jenkins-agent-secret
              key: secret
```

---

## Debugging & Troubleshooting

```bash
# Check which SCC a pod is using
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Check the UID the container is running as
oc exec <pod-name> -- id

# List all SCCs available
oc get scc

# Describe a specific SCC
oc describe scc anyuid

# Check which ServiceAccounts have a specific SCC
oc get scc anyuid -o jsonpath='{.users}'

# Simulate which SCC would be assigned to a pod spec
oc adm policy scc-subject-review -f pod.yaml

# Check SCC review for a specific ServiceAccount
oc adm policy scc-review -z root-sa -n my-namespace

# View all pods and their SCCs across namespaces
oc get pods --all-namespaces -o json | \
  jq '.items[] | {name: .metadata.name, ns: .metadata.namespace, scc: .metadata.annotations["openshift.io/scc"], uid: .spec.containers[0].securityContext.runAsUser}'
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|:--|:--|:--|
| `unable to validate against any security context constraint` | SA lacks an SCC that allows `runAsUser: 0` | Grant `anyuid` SCC: `oc adm policy add-scc-to-user anyuid -z <sa> -n <ns>` |
| `runAsUser: Invalid value: 0` | `restricted` SCC rejects UID 0 | Assign `anyuid` or `privileged` SCC to the SA |
| Container runs but app fails with permission errors | App needs root but container got a random UID | Verify SCC with `oc get pod <name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'` |
| `CrashLoopBackOff` on database containers | Image expects root but gets random UID | Add `securityContext.runAsUser: 0` and grant `anyuid` SCC |
| `mkdir: cannot create directory: Permission denied` | Volume mount owned by root, container running as non-root | Use `anyuid` SCC + `runAsUser: 0`, or fix `fsGroup` in pod spec |

---

## Security Best Practices

- ✅ **Always use `anyuid` over `privileged`** unless you explicitly need host access
- ✅ **Scope SCC grants to a specific ServiceAccount + namespace** — never cluster-wide
- ✅ **Use a dedicated ServiceAccount** — never grant root to the `default` SA
- ✅ **Prefer custom SCCs** to grant only the minimum capabilities
- ✅ **Audit root pods regularly:**

```bash
oc get pods --all-namespaces -o json | \
  jq '.items[] | select(.spec.containers[].securityContext.runAsUser == 0) | {name: .metadata.name, ns: .metadata.namespace}'
```

- ✅ **Remove SCC grants when no longer needed:**

```bash
oc adm policy remove-scc-from-user anyuid -z root-sa -n my-namespace
```

- ✅ **Consider alternatives to root:**

```bash
# Many images have non-root variants
# Example: Use nginx-unprivileged instead of nginx
image: docker.io/nginxinc/nginx-unprivileged:latest
```

---

## Quick Cheat Sheet

```bash
# === SETUP ===
oc create serviceaccount root-sa -n my-namespace
oc adm policy add-scc-to-user anyuid -z root-sa -n my-namespace

# === VERIFY ===
oc adm policy who-can use scc anyuid
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
oc exec <pod> -- id

# === CLEANUP ===
oc adm policy remove-scc-from-user anyuid -z root-sa -n my-namespace
oc delete serviceaccount root-sa -n my-namespace
```

---

> 📌 **Key Takeaway:** Use `anyuid` SCC + a dedicated ServiceAccount to run containers as root. Reserve `privileged` SCC only when host-level access is truly required. Always follow the principle of least privilege.
