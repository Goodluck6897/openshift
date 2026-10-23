Here are your GitHub-ready markdown notes on OpenShift SCCs, Venkatadry:


# 🔐 Security Context Constraints (SCC) in OpenShift — Complete Guide

> **One-stop reference** for understanding, troubleshooting, and implementing SCCs in production OpenShift clusters.

---

## 📌 Table of Contents

- [What Are SCCs?](#-what-are-sccs)
- [Why SCCs Matter](#-why-sccs-matter)
- [Key SCC Attributes](#-key-scc-attributes)
- [Default SCCs in OpenShift](#-default-sccs-in-openshift)
- [Practical Scenarios with Outputs](#-practical-scenarios-with-outputs)
  - [Scenario 1: List All SCCs](#scenario-1-list-all-sccs-in-the-cluster)
  - [Scenario 2: Pod Fails Due to Restricted SCC](#scenario-2-pod-fails-due-to-restricted-scc)
  - [Scenario 3: Fix with anyuid SCC](#scenario-3-fix-it--grant-anyuid-scc-to-a-service-account)
  - [Scenario 4: Check SCC of a Running Pod](#scenario-4-check-which-scc-a-running-pod-is-using)
  - [Scenario 5: Who Can Use an SCC](#scenario-5-validate-which-scc-a-usersa-can-use)
  - [Scenario 6: Create a Custom SCC](#scenario-6-create-a-custom-scc)
  - [Scenario 7: Remove SCC from a Service Account](#scenario-7-remove-scc-from-a-service-account)
- [Real-Time Production Implementations](#-real-time-production-implementations)
- [Best Practices](#-production-best-practices)
- [Troubleshooting Flowchart](#-troubleshooting-flowchart)
- [Quick Reference Commands](#-quick-reference-commands)

---

## 🔍 What Are SCCs?

Security Context Constraints (SCCs) are **OpenShift's security enforcement layer** that controls what a pod/container is allowed to do at runtime.

- They define a set of **rules** governing what actions a pod or container can perform
- They are an **RBAC + security enforcement layer**, unique to OpenShift
- Built to enforce **strong multi-tenancy** and **non-root container execution**
- Makes OpenShift **more secure than vanilla Kubernetes** by default


┌──────────────────────────────────────────────┐
│              Pod Creation Request             │
│                     │                         │
│                     ▼                         │
│         ┌───────────────────┐                 │
│         │  Admission Control │                │
│         │   (SCC Validation) │                │
│         └─────────┬─────────┘                 │
│                   │                           │
│          ┌────────┴────────┐                  │
│          ▼                 ▼                  │
│    ✅ Allowed         ❌ Denied               │
│   (Pod Created)    (Pod Rejected)             │
└──────────────────────────────────────────────┘


---

## 🚧 Why SCCs Matter

By default, OpenShift enforces **non-root, least privilege** principles. SCCs prevent containers from:

| Blocked Action | Risk If Allowed |
|---|---|
| Running as root user | Full host compromise |
| [Mounting host paths](https://github.com/Goodluck6897/openshift/blob/main/OpenShift-SCC-Mounting-HostPaths.md) | Data exfiltration |
| [Running privileged containers](https://github.com/Goodluck6897/openshift/blob/main/Running%20privileged%20containers.md) | Kernel-level access |
| Escaping container isolation | Lateral movement |
| Using host network | Network sniffing |
| Accessing host PID/IPC | Process manipulation |

> 💡 **Key Insight:** In vanilla Kubernetes, pods run as root by default. In OpenShift, they run as **non-root by default** — this is the #1 security difference.

---

## 🧱 Key SCC Attributes

| Attribute | What It Controls | Example Values |
|---|---|---|
| `runAsUser` | Whether pod runs as root or specific UID | `MustRunAsRange`, `RunAsAny`, `MustRunAsNonRoot` |
| `seLinuxContext` | SELinux labels applied to containers | `MustRunAs`, `RunAsAny` |
| `allowPrivilegedContainer` | Whether privileged mode is allowed | `true` / `false` |
| `volumes` | Which volume types are permitted | `configMap`, `hostPath`, `PVC`, `secret` |
| `allowHostNetwork` | Access to host's network namespace | `true` / `false` |
| `allowHostPID` | Access to host PID namespace | `true` / `false` |
| `allowHostIPC` | Access to host IPC namespace | `true` / `false` |
| `readOnlyRootFilesystem` | Forces read-only root filesystem | `true` / `false` |
| `fsGroup` | Supplemental group IDs for volumes | `MustRunAs`, `RunAsAny` |
| `requiredDropCapabilities` | Linux capabilities that MUST be dropped | `ALL`, `KILL`, `MKNOD` |
| `allowedCapabilities` | Linux capabilities that CAN be requested | `NET_BIND_SERVICE`, `NET_ADMIN` |

---

## 🧩 Default SCCs in OpenShift

| SCC Name | Use Case | Run as Root? | Privileged? | Priority |
|---|---|---|---|---|
| `restricted` | Default for most users | ❌ No | ❌ No | Default |
| `anyuid` | Workloads requiring root UID | ✅ Yes | ❌ No | 10 |
| `privileged` | System-level workloads | ✅ Yes | ✅ Yes | — |
| `hostaccess` | Host network and volumes | ✅ Yes | ❌ No | — |
| `nonroot` | Explicit non-root enforcement | ❌ No | ❌ No | — |
| `hostmount-anyuid` | Host mounts with any UID | ✅ Yes | ❌ No | — |
| `hostnetwork` | Host network access | ❌ (Range) | ❌ No | — |
| `baseline` (v4.x+) | Common unprivileged workloads | ❌ No | ❌ No | — |

> **SCC Priority:** When multiple SCCs match, OpenShift picks the **most restrictive** one that satisfies the pod's requirements. Higher priority number = preferred when tied.

---

## 🛠 Practical Scenarios with Outputs

### Scenario 1: List All SCCs in the Cluster

```bash
oc get scc

Output:

NAME                 PRIV    CAPS   SELINUX     RUNASUSER          FSGROUP     SUPGROUP    PRIORITY
anyuid               false   []     MustRunAs   RunAsAny           RunAsAny    RunAsAny    10
hostaccess           false   []     MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <none>
hostmount-anyuid     false   []     MustRunAs   RunAsAny           RunAsAny    RunAsAny    <none>
hostnetwork          false   []     MustRunAs   MustRunAsRange     MustRunAs   MustRunAs   <none>
nonroot              false   []     MustRunAs   MustRunAsNonRoot   RunAsAny    RunAsAny    <none>
privileged           true    [*]    RunAsAny    RunAsAny           RunAsAny    RunAsAny    <none>
restricted           false   []     MustRunAs   MustRunAsRange     MustRunAs   RunAsAny    <none>

Inspect a specific SCC:

oc describe scc restricted

Scenario 2: Pod Fails Due to Restricted SCC

🔥 This is the #1 issue new OpenShift users face

🔥 This is the #1 issue new OpenShift users face

Deploy nginx (runs as root by default):

oc new-project scc-demo
oc run nginx --image=nginx --restart=Never

Check pod status:

oc get pods

Output:

NAME    READY   STATUS             RESTARTS   AGE
nginx   0/1     CrashLoopBackOff   2          45s

Check the error:

oc describe pod nginx | grep -A5 "Warning"

Output:

Warning  Failed  ...  Error: container has runAsNonRoot and image will run as root

Root Cause: The restricted SCC (default) blocks root execution. Nginx's official image runs as root (UID 0).

Scenario 3: Fix It — Grant anyuid SCC to a Service Account

# Step 1: Create a dedicated service account
oc create sa nginx-sa -n scc-demo

# Step 2: Grant anyuid SCC to the service account
oc adm policy add-scc-to-user anyuid -z nginx-sa -n scc-demo

Output:

clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid added: "nginx-sa"

Verify the binding:

oc get scc anyuid -o yaml | grep -A5 users

Output:

users:
- system:serviceaccount:scc-demo:nginx-sa

Redeploy with the service account:

oc run nginx --image=nginx --restart=Never \
  --overrides='{"spec":{"serviceAccountName":"nginx-sa"}}'

Check pod status:

oc get pods

Output:

NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          10s

✅ Pod now runs successfully under anyuid SCC.

Scenario 4: Check Which SCC a Running Pod Is Using

oc get pod nginx -o yaml | grep scc

Output:

openshift.io/scc: anyuid

Alternative — check all pods in a namespace:

oc get pods -o custom-columns='NAME:.metadata.name,SCC:.metadata.annotations.openshift\.io/scc'

Output:

NAME    SCC
nginx   anyuid
app1    restricted
app2    nonroot

Scenario 5: Validate Which SCC a User/SA Can Use

oc adm policy who-can use scc anyuid

Output:

Namespace: scc-demo
Verb:      use
Resource:  securitycontextconstraints

Users:  system:admin
        system:serviceaccount:scc-demo:nginx-sa
Groups: system:cluster-admins
        system:masters

Check which SCCs a specific service account can use:

oc adm policy scc-subject-review -z nginx-sa -n scc-demo

Scenario 6: Create a Custom SCC

custom-scc.yaml:

apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-restricted
  annotations:
    kubernetes.io/description: "Custom SCC - non-root with NET_BIND_SERVICE"
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
allowHostDirVolumePlugin: false
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
requiredDropCapabilities:
  - ALL
allowedCapabilities:
  - NET_BIND_SERVICE
readOnlyRootFilesystem: false

Apply and verify:

oc create -f custom-scc.yaml
oc get scc custom-restricted

Output:

NAME                PRIV    CAPS                SELINUX     RUNASUSER            FSGROUP     SUPGROUP
custom-restricted   false   [NET_BIND_SERVICE]  MustRunAs   MustRunAsNonRoot     MustRunAs   RunAsAny

Bind it to a service account:

oc adm policy add-scc-to-user custom-restricted -z myapp-sa -n myproject

Scenario 7: Remove SCC from a Service Account

oc adm policy remove-scc-from-user anyuid -z nginx-sa -n scc-demo

Output:

clusterrole.rbac.authorization.k8s.io/system:openshift:scc:anyuid removed: "nginx-sa"

🏭 Real-Time Production Implementations

#Use CaseSCC UsedWhy It's Needed1Legacy apps (Oracle DB, IBM MQ, WebSphere)anyuidImages require root UID to start processes2Monitoring agents (Prometheus Node Exporter, Datadog, Dynatrace)privileged / hostaccessNeed host metrics, PID namespace, host filesystem3Istio / Service Mesh sidecar injectionanyuid + NET_ADMINInit containers modify iptables rules4Logging agents (Fluentd, Fluent Bit, Splunk)privilegedMust mount /var/log from host, read all container logs5CI/CD build pods (Jenkins, Tekton, ArgoCD)anyuid / customBuild tools need root for package installs6NFS-backed PVCsCustom (fsGroup: RunAsAny)NFS requires specific group ownership for writes7GPU workloads (NVIDIA)privilegedGPU device plugin needs host device access8Nginx / Apache Ingressanyuid + NET_BIND_SERVICEBinding to port 80/443 requires root or capability9Security scanning (Twistlock, ACS, Prisma)privilegedFull host access to scan images and runtime10Multi-tenant SaaS platformrestricted (default)Enforces strict isolation between tenant namespaces11Elasticsearch / Kafkaanyuid + customRequires vm.max_map_count and specific UIDs12Backup agents (Velero, OADP)privilegedNeeds host filesystem and snapshot access

#Use CaseSCC UsedWhy It's Needed

#

Use Case

SCC Used

Why It's Needed

1Legacy apps (Oracle DB, IBM MQ, WebSphere)anyuidImages require root UID to start processes

1

Legacy apps (Oracle DB, IBM MQ, WebSphere)

anyuid

Images require root UID to start processes

2Monitoring agents (Prometheus Node Exporter, Datadog, Dynatrace)privileged / hostaccessNeed host metrics, PID namespace, host filesystem

2

Monitoring agents (Prometheus Node Exporter, Datadog, Dynatrace)

privileged / hostaccess

Need host metrics, PID namespace, host filesystem

3Istio / Service Mesh sidecar injectionanyuid + NET_ADMINInit containers modify iptables rules

3

Istio / Service Mesh sidecar injection

anyuid + NET_ADMIN

Init containers modify iptables rules

4Logging agents (Fluentd, Fluent Bit, Splunk)privilegedMust mount /var/log from host, read all container logs

4

Logging agents (Fluentd, Fluent Bit, Splunk)

privileged

Must mount /var/log from host, read all container logs

5CI/CD build pods (Jenkins, Tekton, ArgoCD)anyuid / customBuild tools need root for package installs

5

CI/CD build pods (Jenkins, Tekton, ArgoCD)

anyuid / custom

Build tools need root for package installs

6NFS-backed PVCsCustom (fsGroup: RunAsAny)NFS requires specific group ownership for writes

6

NFS-backed PVCs

Custom (fsGroup: RunAsAny)

NFS requires specific group ownership for writes

7GPU workloads (NVIDIA)privilegedGPU device plugin needs host device access

7

GPU workloads (NVIDIA)

privileged

GPU device plugin needs host device access

8Nginx / Apache Ingressanyuid + NET_BIND_SERVICEBinding to port 80/443 requires root or capability

8

Nginx / Apache Ingress

anyuid + NET_BIND_SERVICE

Binding to port 80/443 requires root or capability

9Security scanning (Twistlock, ACS, Prisma)privilegedFull host access to scan images and runtime

9

Security scanning (Twistlock, ACS, Prisma)

privileged

Full host access to scan images and runtime

10Multi-tenant SaaS platformrestricted (default)Enforces strict isolation between tenant namespaces

10

Multi-tenant SaaS platform

restricted (default)

Enforces strict isolation between tenant namespaces

11Elasticsearch / Kafkaanyuid + customRequires vm.max_map_count and specific UIDs

11

Elasticsearch / Kafka

anyuid + custom

Requires vm.max_map_count and specific UIDs

12Backup agents (Velero, OADP)privilegedNeeds host filesystem and snapshot access

12

Backup agents (Velero, OADP)

privileged

Needs host filesystem and snapshot access

Production Example: Deploying Elasticsearch on OpenShift

# Create namespace and service account
oc new-project elastic
oc create sa elastic-sa -n elastic

# Grant anyuid (Elasticsearch needs to run as UID 1000)
oc adm policy add-scc-to-user anyuid -z elastic-sa -n elastic

# Deploy with the service account
# In your Deployment spec:
#   spec:
#     serviceAccountName: elastic-sa
#     securityContext:
#       runAsUser: 1000
#       fsGroup: 1000

Production Example: Deploying Prometheus Node Exporter (DaemonSet)

# Node exporter needs host access for metrics
oc create sa node-exporter-sa -n monitoring
oc adm policy add-scc-to-user hostaccess -z node-exporter-sa -n monitoring

# In DaemonSet spec:
#   spec:
#     serviceAccountName: node-exporter-sa
#     hostNetwork: true
#     hostPID: true
#     volumes:
#       - name: proc
#         hostPath:
#           path: /proc

✅ Production Best Practices

Do's ✅

- Start with restricted — escalate only when absolutely needed

- Create dedicated service accounts per app — never use default SA

- Create custom SCCs with only the exact capabilities needed

- Document every SCC exception — maintain a registry of which apps need elevated SCCs and why

- Prefer nonroot over anyuid when the app just needs a specific non-root UID

- Label namespaces with Pod Security Admission (PSA) for defense-in-depth:oc label namespace myns pod-security.kubernetes.io/enforce=restricted

- Audit SCC bindings regularly:oc get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name | startswith("system:openshift:scc"))'

Don'ts ❌

- Never grant privileged SCC cluster-wide

- Never bind SCCs to the default service account

- Never use anyuid when nonroot would suffice

- Never skip SCC review during CI/CD pipeline design

- Never grant SCCs to groups when individual SA binding is possible

🔧 Troubleshooting Flowchart

Pod CrashLoopBackOff / CreateContainerConfigError?
│
├─ Step 1: Check events
│  oc describe pod <name> | grep -A10 Events
│
├─ Step 2: Identify the error
│  │
│  ├─ "runAsNonRoot and image will run as root"
│  │   → App needs root → Grant anyuid to SA
│  │
│  ├─ "permission denied" on port < 1024
│  │   → Add NET_BIND_SERVICE capability via custom SCC
│  │
│  ├─ "hostPath" volume denied
│  │   → Grant hostaccess or custom SCC with hostPath
│  │
│  ├─ "privileged container not allowed"
│  │   → Grant privileged SCC (last resort!)
│  │
│  ├─ "unable to validate against any SCC"
│  │   → SA doesn't have any matching SCC → check bindings
│  │
│  └─ "fsGroup" related errors
│      → Create custom SCC with appropriate fsGroup policy
│
└─ Step 3: Verify fix
   oc get pod <name> -o yaml | grep openshift.io/scc

📋 Quick Reference Commands

# List all SCCs
oc get scc

# Describe a specific SCC
oc describe scc <scc-name>

# Check which SCC a pod is using
oc get pod <pod-name> -o yaml | grep openshift.io/scc

# Check all pods' SCCs in a namespace
oc get pods -o custom-columns='NAME:.metadata.name,SCC:.metadata.annotations.openshift\.io/scc'

# Grant SCC to a service account
oc adm policy add-scc-to-user <scc-name> -z <sa-name> -n <namespace>

# Remove SCC from a service account
oc adm policy remove-scc-from-user <scc-name> -z <sa-name> -n <namespace>

# Check who can use an SCC
oc adm policy who-can use scc <scc-name>

# Review which SCC a service account would get
oc adm policy scc-subject-review -z <sa-name> -n <namespace>

# Audit all SCC bindings in the cluster
oc get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name | startswith("system:openshift:scc")) | {name: .metadata.name, scc: .roleRef.name, subjects: .subjects}'

# Export an SCC to YAML (for backup/modification)
oc get scc <scc-name> -o yaml > scc-backup.yaml

📚 References

### OpenShift Docs — Managing SCCs
- [Managing Security Context Constraints (OCP 4.x - English)](https://docs.openshift.com/en/documentation/openshift_container_platform/4.17/html/security_apis/securitycontextconstraints-security-openshift-io-v1) [1]
- [Managing Security Context Constraints (OCP 4.19)](https://docs.openshift.com/pt-br/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies) [2]
- [Red Hat Docs — Managing SCCs (OpenShift Dedicated)](https://docs.redhat.com/en/documentation/openshift_dedicated/4/html/authentication_and_authorization/managing-pod-security-policies) [29]
