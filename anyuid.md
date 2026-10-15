


📝 OpenShift SCC & anyuid Quick Reference


🚀 Overview: Why do we need anyuid?

By default, OpenShift Engine forces all pods to run using a randomly generated, high-range User ID (UID) (e.g., 1000620000) unique to each namespace. This provides exceptional multi-tenant security but breaks:
• Standard Docker Hub images (e.g., Nginx, Redis) that expect to run as root (UID 0) or a specific fixed UID (e.g., 1001).
• Pods accessing legacy persistent volumes with rigid UNIX user ownership.
The anyuid SCC is the safest mechanism to let a pod bypass dynamic allocation and run as the exact user specified inside the Docker container image definition.

🛠️ CLI Operations (Cheatsheet)


1. View Available SCCs in the Cluster

bash
oc get scc
Use code with caution.

2. Inspect the Details of the anyuid SCC

bash
oc describe scc anyuid
Use code with caution.

3. Identify Which SCC a Running Pod is Using

If an application is crashing or you want to verify it picked up the permission change:
bash
oc get pod <pod_name> -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
Use code with caution.

🔒 Production Implementation Workflow

Never apply anyuid directly to the default service account of a namespace, as it exposes all pods in that namespace to potential container escape vulnerabilities. Always isolate it via a dedicated ServiceAccount.

Step 1: Create a Dedicated Service Account

bash
oc create sa custom-app-sa -n <your-namespace>
Use code with caution.

Step 2: Bind anyuid to the Service Account

bash
oc adm policy add-scc-to-user anyuid -z custom-app-sa -n <your-namespace>
Use code with caution.

Step 3: Reference the Service Account in your Deployment

Apply this to your deployment manifest (deployment.yaml):
yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-nginx-app
  namespace: my-app-namespace
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: legacy-nginx
    spec:
      serviceAccountName: custom-app-sa # 👈 Connects the pod to the anyuid SCC
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
Use code with caution.

🔍 Troubleshooting Common SCC Issues


Symptom: CrashLoopBackOff or Permission Denied in Pod Logs

If an off-the-shelf image fails immediately with file permissions, it's likely hitting the dynamic UID restriction.
• Verification Command:bash
oc get events --sort-by='.metadata.creationTimestamp'
Use code with caution.
• Resolution: Implement the anyuid binding workflow outlined above.

How to Roll Back / Remove anyuid Permissions

If a workload is decommissioned or updated to a non-root design, strip the security bypass immediately:
bash
oc adm policy remove-scc-from-user anyuid -z custom-app-sa -n <your-namespace>
Use code with caution.

⚠️ Security Warning Rules

• Principle of Least Privilege: anyuid allows containerized root (UID 0). While isolated by SELinux, a vulnerability in the container runtime could allow a malicious user to exploit host kernel structures.
• Alternative to anyuid: Whenever possible, modify the application's Dockerfile to grant the group 0 (root group) directories access permissions (CHMOD 775). OpenShift runs random UIDs inside the root group by default, removing the need for anyuid entirely.
If you would like to expand your GitHub notes, let me know if you want to add:
• A complete YAML manifest version combining the ServiceAccount, RoleBinding, and Deployment.
• A section on how to build OpenShift-native Dockerfiles that don't need anyuid.
• Steps for debugging SELinux or storage volume permission errors.



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
