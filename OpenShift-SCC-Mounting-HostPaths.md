Here are your GitHub-ready markdown notes on Mounting Host Paths in OpenShift SCCs, Venkatadry:


# 📂 Mounting Host Paths in OpenShift SCC — Complete Guide

> **What it means, why it's dangerous, how SCCs protect you, and real-world production examples.**

---

## 📌 Table of Contents

- [What Does Mounting Host Paths Mean?](#-what-does-mounting-host-paths-mean)
- [How It Works Technically](#-how-it-works-technically)
- [Why Is This Dangerous?](#-why-is-this-dangerous)
- [How SCCs Protect Against This](#-how-sccs-protect-against-this)
- [Real-World Production Examples](#-real-world-production-examples)
  - [Legitimate Use — Log Collection Agent (Fluentd)](#-legitimate-use--log-collection-agent-fluentd)
  - [Legitimate Use — Prometheus Node Exporter](#-legitimate-use--prometheus-node-exporter)
  - [What Gets Blocked (Default Behavior)](#-what-gets-blocked-default-behavior)
- [Custom SCC for hostPath Only](#-custom-scc-for-hostpath-only)
- [Best Practices](#-best-practices)
- [Audit Commands](#-audit-commands)
- [Quick Summary](#-quick-summary)
- [References](#-references)

---

## 🔍 What Does Mounting Host Paths Mean?

**Mounting host paths** means a container inside a pod gets **direct access to a directory or file on the underlying worker node's filesystem** using a volume type called `hostPath`.

### Simple Analogy

> Your container is a **locked room** inside a building (the node). Normally, the container can only see what's inside its own room. **Mounting a host path is like cutting a window or door into the building's hallway** — the container can now read/write files that belong to the host machine itself.

### Visual Representation


Normal Pod (restricted SCC):
┌─────────────────────────────────┐
│  Worker Node                    │
│  ┌───────────────────┐          │
│  │  Container         │          │
│  │  [own filesystem]  │  ✖ No   │
│  │                    │──access──│──→ Node's /var/log
│  └───────────────────┘          │
└─────────────────────────────────┘

Pod with hostPath (hostaccess/privileged SCC):
┌─────────────────────────────────┐
│  Worker Node                    │
│  ┌───────────────────┐          │
│  │  Container         │          │
│  │  [/host-logs]      │──────────│──→ Node's /var/log ✅
│  │                    │  direct  │
│  └───────────────────┘  access  │
└─────────────────────────────────┘


---

## ⚙️ How It Works Technically

In a pod spec, a `hostPath` volume mount looks like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  volumes:
    - name: host-logs
      hostPath:
        path: /var/log          # ← Directory on the WORKER NODE
        type: Directory
  containers:
    - name: my-app
      image: my-image
      volumeMounts:
        - name: host-logs
          mountPath: /host-logs  # ← Appears inside the container here

What Happens

ComponentPathDescriptionNode (Host)/var/logThe actual directory on the worker node's filesystemContainer/host-logsThe mount point inside the containerResult—Container can read/write all files in the node's /var/log

ComponentPathDescription

Component

Path

Description

Node (Host)/var/logThe actual directory on the worker node's filesystem

Node (Host)

/var/log

The actual directory on the worker node's filesystem

Container/host-logsThe mount point inside the container

Container

/host-logs

The mount point inside the container

Result—Container can read/write all files in the node's /var/log

Result

—

Container can read/write all files in the node's /var/log

hostPath Types

TypeBehavior"" (empty)No checks before mountingDirectoryOrCreateCreates directory if it doesn't existDirectoryDirectory must already existFileOrCreateCreates file if it doesn't existFileFile must already existSocketUNIX socket must existCharDeviceCharacter device must existBlockDeviceBlock device must exist

TypeBehavior

Type

Behavior

"" (empty)No checks before mounting

"" (empty)

No checks before mounting

DirectoryOrCreateCreates directory if it doesn't exist

DirectoryOrCreate

Creates directory if it doesn't exist

DirectoryDirectory must already exist

Directory

Directory must already exist

FileOrCreateCreates file if it doesn't exist

FileOrCreate

Creates file if it doesn't exist

FileFile must already exist

File

File must already exist

SocketUNIX socket must exist

Socket

UNIX socket must exist

CharDeviceCharacter device must exist

CharDevice

Character device must exist

BlockDeviceBlock device must exist

BlockDevice

Block device must exist

🚨 Why Is This Dangerous?

Risk Matrix

RiskWhat Could HappenSeverityData exfiltrationCompromised container reads /etc/shadow, /etc/kubernetes🔴 CriticalHost compromiseContainer writes malicious cron jobs, SSH keys to host🔴 CriticalContainer escapeMounting / (root filesystem) = full host access🔴 CriticalCross-tenant data leakOne tenant's pod reads another tenant's data on same node🔴 CriticalCluster takeoverMounting /etc/kubernetes/pki exposes cluster certs & keys🔴 CriticalMalware persistenceMalware written to host survives container restarts🟠 HighLog tamperingContainer modifies or deletes host audit logs🟠 High

RiskWhat Could HappenSeverity

Risk

What Could Happen

Severity

Data exfiltrationCompromised container reads /etc/shadow, /etc/kubernetes🔴 Critical

Data exfiltration

Compromised container reads /etc/shadow, /etc/kubernetes

🔴 Critical

Host compromiseContainer writes malicious cron jobs, SSH keys to host🔴 Critical

Host compromise

Container writes malicious cron jobs, SSH keys to host

🔴 Critical

Container escapeMounting / (root filesystem) = full host access🔴 Critical

Container escape

Mounting / (root filesystem) = full host access

🔴 Critical

Cross-tenant data leakOne tenant's pod reads another tenant's data on same node🔴 Critical

Cross-tenant data leak

One tenant's pod reads another tenant's data on same node

🔴 Critical

Cluster takeoverMounting /etc/kubernetes/pki exposes cluster certs & keys🔴 Critical

Cluster takeover

Mounting /etc/kubernetes/pki exposes cluster certs & keys

🔴 Critical

Malware persistenceMalware written to host survives container restarts🟠 High

Malware persistence

Malware written to host survives container restarts

🟠 High

Log tamperingContainer modifies or deletes host audit logs🟠 High

Log tampering

Container modifies or deletes host audit logs

🟠 High

Examples of Dangerous Mounts

# ⚠️ CRITICAL — Mounts the ENTIRE host filesystem
volumes:
  - name: root-fs
    hostPath:
      path: /

# ⚠️ CRITICAL — Exposes cluster PKI certificates
volumes:
  - name: k8s-certs
    hostPath:
      path: /etc/kubernetes/pki

# ⚠️ HIGH — Exposes all host credentials
volumes:
  - name: etc
    hostPath:
      path: /etc

# ⚠️ HIGH — Access to container runtime socket
volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock

🛡️ How SCCs Protect Against This

The restricted SCC (default in OpenShift) blocks hostPath volumes entirely.

SCC vs hostPath Permissions

SCCAllows hostPath?Allows Privileged?Typical Use Caserestricted❌ No❌ NoDefault — most workloadsnonroot❌ No❌ NoNon-root appsanyuid❌ No❌ NoRoot UID apps (no host access)hostaccess✅ Yes❌ NoMonitoring/logging agentshostmount-anyuid✅ Yes❌ NoHost mounts with any UIDprivileged✅ Yes✅ YesSystem-level workloads

SCCAllows hostPath?Allows Privileged?Typical Use Case

SCC

Allows hostPath?

Allows Privileged?

Typical Use Case

restricted❌ No❌ NoDefault — most workloads

restricted

❌ No

❌ No

Default — most workloads

nonroot❌ No❌ NoNon-root apps

nonroot

❌ No

❌ No

Non-root apps

anyuid❌ No❌ NoRoot UID apps (no host access)

anyuid

❌ No

❌ No

Root UID apps (no host access)

hostaccess✅ Yes❌ NoMonitoring/logging agents

hostaccess

✅ Yes

❌ No

Monitoring/logging agents

hostmount-anyuid✅ Yes❌ NoHost mounts with any UID

hostmount-anyuid

✅ Yes

❌ No

Host mounts with any UID

privileged✅ Yes✅ YesSystem-level workloads

privileged

✅ Yes

✅ Yes

System-level workloads

Verify Allowed Volumes for an SCC

oc describe scc restricted | grep -i volumes

Output:

Volumes:    configMap,downwardAPI,emptyDir,persistentVolumeClaim,projected,secret

☝️ Notice — no hostPath in the list. That's the protection.

☝️ Notice — no hostPath in the list. That's the protection.

oc describe scc hostaccess | grep -i volumes

Output:

Volumes:    configMap,downwardAPI,emptyDir,hostPath,persistentVolumeClaim,projected,secret

☝️ hostPath is present — this SCC allows host mounts.

☝️ hostPath is present — this SCC allows host mounts.

🏭 Real-World Production Examples

✅ Legitimate Use — Log Collection Agent (Fluentd)

Fluentd runs as a DaemonSet on every node and needs to read container logs from the host.

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd-sa
      containers:
        - name: fluentd
          image: fluent/fluentd:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true              # ← Best practice: read-only
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
            type: Directory
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
            type: Directory

Grant the required SCC:

# Create service account
oc create sa fluentd-sa -n logging

# Grant hostaccess SCC
oc adm policy add-scc-to-user hostaccess -z fluentd-sa -n logging

Output:

clusterrole.rbac.authorization.k8s.io/system:openshift:scc:hostaccess added: "fluentd-sa"

✅ Legitimate Use — Prometheus Node Exporter

Needs /proc and /sys from the host for hardware and OS metrics.

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
      serviceAccountName: node-exporter-sa
      hostNetwork: true
      hostPID: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              mountPropagation: HostToContainer
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /

Grant the required SCC:

oc create sa node-exporter-sa -n monitoring
oc adm policy add-scc-to-user hostaccess -z node-exporter-sa -n monitoring

❌ What Gets Blocked (Default Behavior)

# A regular pod trying to mount hostPath — will be DENIED
oc run test --image=nginx --restart=Never \
  --overrides='{
    "spec":{
      "volumes":[{"name":"host","hostPath":{"path":"/var/log"}}],
      "containers":[{
        "name":"test",
        "image":"nginx",
        "volumeMounts":[{"name":"host","mountPath":"/logs"}]
      }]
    }
  }'

Error Output:

Error from server (Forbidden): pods "test" is forbidden: unable to validate against
any security context constraint: [spec.volumes: Invalid value: "hostPath":
hostPath volumes are not allowed to be used]

✅ OpenShift's restricted SCC blocked the hostPath mount — cluster is protected.

✅ OpenShift's restricted SCC blocked the hostPath mount — cluster is protected.

🔧 Custom SCC for hostPath Only

When you need hostPath access without granting full privileged or hostaccess permissions:

hostpath-only-scc.yaml:

apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: hostpath-only
  annotations:
    kubernetes.io/description: "Allows hostPath volumes only — no privileged, no host network"
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
allowHostDirVolumePlugin: true
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
readOnlyRootFilesystem: false
requiredDropCapabilities:
  - ALL
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - hostPath              # ← Only addition vs restricted
  - persistentVolumeClaim
  - projected
  - secret

Apply and bind:

# Create the custom SCC
oc create -f hostpath-only-scc.yaml

# Verify
oc get scc hostpath-only
oc describe scc hostpath-only | grep -i volumes

# Bind to a service account
oc adm policy add-scc-to-user hostpath-only -z myapp-sa -n myproject

Output:

NAME            PRIV    CAPS   SELINUX     RUNASUSER            FSGROUP     SUPGROUP
hostpath-only   false   []     MustRunAs   MustRunAsNonRoot     MustRunAs   RunAsAny

Volumes:    configMap,downwardAPI,emptyDir,hostPath,persistentVolumeClaim,projected,secret

✅ Best Practices

Do's ✅

#PracticeWhy1Always mount as readOnly: truePrevents container from writing to host2Use the most specific path possibleMount /var/log not /3Create dedicated service accountsNever use default SA for hostPath pods4Prefer hostaccess over privilegedGrants host filesystem without full privileges5Create custom SCCsGrant only hostPath without other dangerous permissions6Scope to specific namespacesNever grant hostPath access cluster-wide7Audit regularlyCheck which pods use hostPath mounts8Document every exceptionMaintain a registry of apps needing host access

#PracticeWhy

#

Practice

Why

1Always mount as readOnly: truePrevents container from writing to host

1

Always mount as readOnly: true

Prevents container from writing to host

2Use the most specific path possibleMount /var/log not /

2

Use the most specific path possible

Mount /var/log not /

3Create dedicated service accountsNever use default SA for hostPath pods

3

Create dedicated service accounts

Never use default SA for hostPath pods

4Prefer hostaccess over privilegedGrants host filesystem without full privileges

4

Prefer hostaccess over privileged

Grants host filesystem without full privileges

5Create custom SCCsGrant only hostPath without other dangerous permissions

5

Create custom SCCs

Grant only hostPath without other dangerous permissions

6Scope to specific namespacesNever grant hostPath access cluster-wide

6

Scope to specific namespaces

Never grant hostPath access cluster-wide

7Audit regularlyCheck which pods use hostPath mounts

7

Audit regularly

Check which pods use hostPath mounts

8Document every exceptionMaintain a registry of apps needing host access

8

Document every exception

Maintain a registry of apps needing host access

Don'ts ❌

#Anti-PatternRisk1Mounting / (root filesystem)Complete host compromise2Mounting /var/run/docker.sockContainer can control the container runtime3Mounting /etc/kubernetes/pkiCluster certificate theft4Using privileged SCC just for hostPathOver-permissive — use hostaccess or custom SCC5Granting hostPath to default SAEvery pod in the namespace gets host access6Mounting with write access unnecessarilyAlways use readOnly: true when possible

#Anti-PatternRisk

#

Anti-Pattern

Risk

1Mounting / (root filesystem)Complete host compromise

1

Mounting / (root filesystem)

Complete host compromise

2Mounting /var/run/docker.sockContainer can control the container runtime

2

Mounting /var/run/docker.sock

Container can control the container runtime

3Mounting /etc/kubernetes/pkiCluster certificate theft

3

Mounting /etc/kubernetes/pki

Cluster certificate theft

4Using privileged SCC just for hostPathOver-permissive — use hostaccess or custom SCC

4

Using privileged SCC just for hostPath

Over-permissive — use hostaccess or custom SCC

5Granting hostPath to default SAEvery pod in the namespace gets host access

5

Granting hostPath to default SA

Every pod in the namespace gets host access

6Mounting with write access unnecessarilyAlways use readOnly: true when possible

6

Mounting with write access unnecessarily

Always use readOnly: true when possible

🔎 Audit Commands

# Find ALL pods using hostPath volumes across the cluster
oc get pods --all-namespaces -o json | \
  jq '.items[] | select(.spec.volumes[]?.hostPath != null) | 
  {namespace: .metadata.namespace, pod: .metadata.name, 
   hostPaths: [.spec.volumes[] | select(.hostPath != null) | .hostPath.path]}'

# Check which SCCs allow hostPath
oc get scc -o json | \
  jq '.items[] | select(.volumes[] == "hostPath" or .volumes[] == "*") | .metadata.name'

# Check which service accounts have hostaccess SCC
oc get scc hostaccess -o yaml | grep -A20 users

# Verify a specific pod's SCC
oc get pod <pod-name> -n <namespace> -o yaml | grep openshift.io/scc

# List all pods with their SCC in a namespace
oc get pods -n <namespace> -o custom-columns=\
'NAME:.metadata.name,SCC:.metadata.annotations.openshift\.io/scc'

📊 Quick Summary

┌──────────────────────────────────────────────────────────────┐
│                    hostPath in OpenShift                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  What:   Container gets direct access to node's filesystem   │
│                                                              │
│  Risk:   Data theft, host compromise, container escape       │
│                                                              │
│  Default: BLOCKED by restricted SCC ✅                       │
│                                                              │
│  Allow:   hostaccess, hostmount-anyuid, privileged SCCs      │
│                                                              │
│  Best:    Custom SCC with hostPath only + readOnly mounts    │
│                                                              │
│  Audit:   oc get pods -o json | jq '..hostPath...'           │
│                                                              │
└──────────────────────────────────────────────────────────────┘

OpenShift Docs — Managing SCCs (OCP 4.19) https://docs.openshift.com/pt-br/documentation/openshift_container_platform/4.19/html/authentication_and_authorization/managing-pod-security-policies

Kubernetes Docs — Volumes: hostPath https://kubernetes.io/docs/concepts/storage/volumes/#hostpath

Kubernetes — Pod Security Standards https://kubernetes.io/docs/concepts/security/pod-security-standards/

Red Hat Blog — Introduction to Security Contexts and SCCs https://www.redhat.com/en/blog/introduction-to-security-contexts-and-sccs

Red Hat Blog — Understanding Service Accounts and SCCs https://www.redhat.com/it/blog/understanding-service-accounts-sccs

Red Hat Blog — Managing SCCs in OpenShift https://www.redhat.com/ko/blog/managing-sccs-in-openshift

Red Hat Blog — How to Manage Service Accounts and SCCs https://www.redhat.com/it/blog/security-context-constraint-configuration

Red Hat Blog — How to Fix Permission Errors Using Service Accounts https://www.redhat.com/en/blog/security-context-constraint-permissions

Red Hat Developers — Understanding OpenShift SCCs https://developers.redhat.com/blog/2016/10/21/understanding-openshift-security-context-constraints
