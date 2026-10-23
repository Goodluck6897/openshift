
# 🔐 Running Privileged Containers in OpenShift

> **⚠️ Security Warning:** Running privileged containers bypasses OpenShift's default security model. Always follow the principle of least privilege.

---

## 📋 Table of Contents

- [Understanding SCCs](#understanding-sccs)
- [Built-in SCCs](#built-in-sccs)
- [Method 1: Grant Privileged SCC to a ServiceAccount](#method-1-grant-privileged-scc-to-a-serviceaccount)
- [Method 2: Deployment with Privileged Containers](#method-2-deployment-with-privileged-containers)
- [Method 3: Using anyuid SCC (Root Without Full Privileges)](#method-3-using-anyuid-scc-root-without-full-privileges)
- [Method 4: Creating a Custom SCC](#method-4-creating-a-custom-scc)
- [Method 5: RBAC Approach (ClusterRole + RoleBinding)](#method-5-rbac-approach-clusterrole--rolebinding)
- [Debugging Commands](#debugging-commands)
- [Common Errors and Fixes](#common-errors-and-fixes)
- [Security Best Practices](#security-best-practices)

---

## Understanding SCCs

**Security Context Constraints (SCCs)** control what actions a pod can perform and what resources it can access. By default, OpenShift restricts containers from running as privileged.

---

## Built-in SCCs

| SCC Name | Description |
|:--|:--|
| `restricted` | Default. Denies host features, requires non-root |
| `restricted-v2` | Updated `restricted` (OpenShift 4.11+) |
| `nonroot` | Allows any non-root UID |
| `nonroot-v2` | Updated `nonroot` |
| `anyuid` | Allows any UID including root (UID 0) |
| `hostaccess` | Allows host namespaces, filesystems, PIDs |
| `hostnetwork` | Allows host networking and ports |
| `hostnetwork-v2` | Updated `hostnetwork` |
| `privileged` | **Full unrestricted access** to all host features |

---

## Method 1: Grant Privileged SCC to a ServiceAccount

> ✅ **Recommended approach** — scope the SCC to a specific ServiceAccount, not a user.

### Step 1 — Create a dedicated ServiceAccount

```bash
oc create serviceaccount privileged-sa -n my-namespace
```

### Step 2 — Add the `privileged` SCC

```bash
oc adm policy add-scc-to-user privileged -z privileged-sa -n my-namespace
```

### Step 3 — Verify the assignment

```bash
oc adm policy who-can use scc privileged
```

### Step 4 — Deploy a privileged pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: my-namespace
spec:
  serviceAccountName: privileged-sa
  containers:
    - name: privileged-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        privileged: true
```

```bash
oc apply -f privileged-pod.yaml
```

### Step 5 — Verify the pod SCC

```bash
oc get pod privileged-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Expected output:

```
privileged
```

---

## Method 2: Deployment with Privileged Containers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: privileged-deployment
  namespace: my-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: privileged-app
  template:
    metadata:
      labels:
        app: privileged-app
    spec:
      serviceAccountName: privileged-sa
      containers:
        - name: privileged-container
          image: registry.access.redhat.com/ubi9/ubi:latest
          command: ["sleep", "infinity"]
          securityContext:
            privileged: true
            runAsUser: 0
          volumeMounts:
            - name: host-volume
              mountPath: /host
      volumes:
        - name: host-volume
          hostPath:
            path: /
            type: Directory
```

```bash
oc apply -f privileged-deployment.yaml
```

---

## Method 3: Using `anyuid` SCC (Root Without Full Privileges)

> 💡 Use this if you **only** need to run as root but **don't** need host-level access.

```bash
oc adm policy add-scc-to-user anyuid -z my-sa -n my-namespace
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: my-namespace
spec:
  serviceAccountName: my-sa
  containers:
    - name: root-container
      image: registry.access.redhat.com/ubi9/ubi:latest
      command: ["sleep", "infinity"]
      securityContext:
        runAsUser: 0
```

---

## Method 4: Creating a Custom SCC

> 🎯 For **fine-grained control** — grant only the minimum capabilities needed.

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-privileged-scc
allowPrivilegedContainer: true
allowHostNetwork: false
allowHostPorts: false
allowHostPID: false
allowHostIPC: false
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
users:
  - system:serviceaccount:my-namespace:custom-sa
```

```bash
oc apply -f custom-scc.yaml
```

---

## Method 5: RBAC Approach (ClusterRole + RoleBinding)

> 🔑 The modern, RBAC-native way to manage SCC access.

### Create a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: use-privileged-scc
rules:
  - apiGroups: ["security.openshift.io"]
    resources: ["securitycontextconstraints"]
    resourceNames: ["privileged"]
    verbs: ["use"]
```

### Bind it to the ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: privileged-scc-binding
  namespace: my-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: use-privileged-scc
subjects:
  - kind: ServiceAccount
    name: privileged-sa
    namespace: my-namespace
```

```bash
oc apply -f clusterrole.yaml
oc apply -f rolebinding.yaml
```

---

## Debugging Commands

```bash
# Check which SCC a running pod is using
oc get pod <pod-name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# List all SCCs
oc get scc

# Describe a specific SCC
oc describe scc privileged

# Check which ServiceAccounts have a specific SCC
oc get scc privileged -o jsonpath='{.users}'

# Validate which SCC would be assigned to a pod
oc adm policy scc-subject-review -f pod.yaml

# Check SCC review for a specific ServiceAccount
oc adm policy scc-review -z privileged-sa -n my-namespace
```

---

## Common Errors and Fixes

| Error | Cause | Fix |
|:--|:--|:--|
| `unable to validate against any security context constraint` | Pod requests capabilities not allowed by any SCC | Grant the appropriate SCC to the ServiceAccount |
| `forbidden: ...securityContext.privileged: Invalid value: true` | `privileged: true` set but SA lacks privileged SCC | `oc adm policy add-scc-to-user privileged -z <sa> -n <ns>` |
| `Error creating: pods "xxx" is forbidden` | Deployment's SA doesn't have the right SCC | Ensure `serviceAccountName` is set correctly in the Deployment spec |

---

## Security Best Practices

- ✅ **Scope to namespace + ServiceAccount** — never grant `privileged` SCC cluster-wide
- ✅ **Prefer `anyuid` over `privileged`** if you only need root access
- ✅ **Use custom SCCs** to grant minimum required capabilities
- ✅ **Audit SCC usage regularly:**

```bash
oc get pods --all-namespaces -o json | \
  jq '.items[] | {name: .metadata.name, namespace: .metadata.namespace, scc: .metadata.annotations["openshift.io/scc"]}'
```

- ✅ **Remove SCC grants when no longer needed:**

```bash
oc adm policy remove-scc-from-user privileged -z privileged-sa -n my-namespace
```

---

> 📌 **Key Takeaway:** Always bind SCCs to a **specific ServiceAccount** in a **specific namespace**. Never grant broad privileged access across the cluster.
