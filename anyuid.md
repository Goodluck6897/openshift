markdown
# 📝 Advanced OpenShift SCC & Non-Root Best Practices

This repository section covers declarative GitOps configurations for OpenShift Security Context Constraints (SCC) and the standard patterns for building OpenShift-native images that eliminate the need for `anyuid`.

---

## 🗂️ 1. Complete GitOps Declarative Manifest (YAML)

When working with GitOps tools like **ArgoCD** or **Red Hat Advanced Cluster Management (RHACM)**, you should avoid using `oc adm` CLI commands. Instead, use this single, combined file to declare your ServiceAccount, ClusterRoleBinding (to access the cluster-scoped `anyuid` SCC), and Deployment.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: legacy-app-sa
  namespace: production-apps
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: legacy-app-anyuid-binding
  namespace: production-apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid # 👈 System ClusterRole that points to anyuid SCC
subjects:
- kind: ServiceAccount
  name: legacy-app-sa
  namespace: production-apps
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-app-deployment
  namespace: production-apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: legacy-workload
  template:
    metadata:
      labels:
        app: legacy-workload
    spec:
      serviceAccountName: legacy-app-sa # 👈 Attaches the permitted SA to the Pod
      containers:
      - name: web-server
        image: nginx:latest
        ports:
        - containerPort: 80
```

---

## 🛠️ 2. Designing OpenShift-Native Images (No `anyuid` Required)

The ultimate security goal in OpenShift is to make containers **independent of the UID** they run as. OpenShift generates an unpredictable UID dynamically, but it **always guarantees that this random user belongs to the Root Group (GID `0`)**.

### The OpenShift-Native Dockerfile Pattern
To ensure an image runs perfectly under the default restrictive `restricted-v2` SCC, configure your file permissions to support the root group (`GID 0`).

```dockerfile
FROM alpine:latest

# 1. Create your application directory
RUN mkdir -p /app

# 2. Copy your application assets
COPY server.sh /app/server.sh

# 3. CRITICAL STEP FOR OPENSHIFT:
# Change ownership to the root group (0) and grant read/write/execute permissions.
# This ensures that whatever random UID OpenShift picks can interact with the files.
RUN chown -R 1001:0 /app && \
    chmod -R g=u /app

# 4. Do NOT use standard ports under 1024 (like 80 or 443).
# Unprivileged random UIDs cannot bind to privileged system ports.
EXPOSE 8080

# 5. Define a non-root default user ID for non-OpenShift environments (like local Docker).
USER 1001

CMD ["/app/server.sh"]
```

### 🧠 The Core Golden Rule
> **`chmod -R g=u`** makes group permissions identical to user permissions. Because OpenShift assigns the dynamic user to the root group `0`, your container dynamically inherits its own required permissions seamlessly.

---

## 🔍 3. Debugging Storage & SELinux Permission Failures

Even if your pod runs fine using `anyuid`, it may still crash due to underlying host storage constraints or SELinux policies.

### Persistent Volume Permission Denied (NFS / SAN / EBS)
If a pod running with `anyuid` fails to write to a volume, the physical files on the storage system are likely locked to a specific external user or group. 

* **The Kubernetes Fix (`fsGroup`):**
  If your storage provisioner supports it, use the `securityContext` parameter inside your Deployment to force the storage mount to adjust its ownership dynamically:

```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: 2000 # 👈 Forces the volume to be readable/writable by GID 2000
      containers:
      - name: database
        image: postgres:latest
```

### Investigating SELinux Denials on the Node
`anyuid` keeps SELinux containment active. If your application attempts to access host directories or specific system calls, the Linux kernel on the worker node will block it.

1. **Locate the specific worker node** hosting your failing pod:
   ```bash
   oc get pod <pod-name> -o wide
   ```
2. **Access the node's system logs** to scan for Audit/SELinux alerts (`AVC denials`):
   ```bash
   oc debug node/<node-name>
   # Inside the debug shell:
   chroot /host
   journalctl -u auditd | grep -i avc
   ```
3. **The Remediation:** If you see an absolute system denial that blocks a vital daemon, `anyuid` is not enough. You will need to carefully evaluate upgrading the specific workload to the `restricted` or `hostmount-anyuid` SCC based on strict architectural alignment.
Use code with caution.
Would you like to add anything else to these documentation files, such as a custom SCC definition template or steps for configuring NetworkPolicies for non-root apps?
