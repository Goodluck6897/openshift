OpenShift Learning Series | OVN-Kubernetes

We often say:

“Pod A can communicate with Pod B.”

But what actually happens when those Pods are running on different worker nodes? 🤔

That question led me deeper into OVN-Kubernetes, the default networking implementation in current OpenShift releases.

For a simplified cross-node flow:

Pod A → OVS (Worker 1) → Geneve Overlay → OVS (Worker 2) → Pod B

Behind this flow, OVN maintains the logical network topology and programs the required flows on the nodes.

A few concepts that helped me connect the pieces:

🔹 OVN Northbound DB → Stores the desired logical network configuration
🔹 OVN Southbound DB → Represents the network state and logical flows used by the OVN system
🔹 OVS (Open vSwitch) → Provides the node-level data plane
🔹 Geneve → Carries overlay traffic between nodes
🔹 NetworkPolicy → Enforced through OVN/OVS networking rules

🚨 Production Scenario

Pods on the same worker node can communicate, but Pods on different nodes cannot.

Where would you investigate?

I’d start with:

Node connectivity → OVN components → Geneve traffic → OVS flows → MTU/firewall configuration

Rather than immediately blaming the application.

📌 Key takeaway:

OpenShift networking becomes much easier to troubleshoot once you understand the path a packet actually takes.

Follow the packet, and the architecture starts making sense.

💬 Have you ever troubleshot a cross-node Pod connectivity issue in OpenShift? What was the root cause?

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/3e07e668-2864-424b-8950-a22ea16ba512" />



<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/c557ff22-cc1b-4853-8a99-f2de9c11fa1a" />





OpenShift Learning Series | Networking Interview Q&A

Over the last few posts, I’ve covered:

Services → Routes → Ingress → NetworkPolicy → OVN-Kubernetes

Today, instead of learning another concept, I wanted to test whether I could actually troubleshoot these concepts in an interview scenario.

So I put together 10 scenario-based OpenShift networking questions covering situations like:

🔹 Pods are Running, but the application is unreachable
🔹 Service exists but has no endpoints
🔹 Pod IP works, but Service IP doesn’t
🔹 Service works, but the Route doesn’t
🔹 Edge vs Re-encrypt vs Passthrough
🔹 NetworkPolicy suddenly blocks communication
🔹 Same-node communication works, cross-node doesn’t
🔹 DNS resolution fails inside a Pod
🔹 Route returns 503
🔹 External user can’t reach the application

The biggest lesson for me wasn’t another command.

It was learning to follow the traffic path:

Internal:
Pod → Service → Endpoint → Pod

External:
User → DNS → LB → Router → Route → Service → Endpoint → Pod

Cross-node:
Pod → OVN/OVS → Geneve → OVN/OVS → Pod

📌 My takeaway:

Don’t troubleshoot OpenShift networking by guessing.
Follow the packet and find the first layer where communication breaks.

That approach makes even complex networking scenarios much easier to reason about.

💬 Interview question: Pods on the same worker node can communicate, but Pods on different worker nodes cannot. What would you check first?




Absolutely. The text you have is mixed up. Here is a **clean, interview-ready troubleshooting command sheet** for this exact problem:

> **Problem:** Pods on the same node can communicate, but Pods on different nodes cannot communicate.

---

# OpenShift OVN-Kubernetes Troubleshooting Commands

## 1. Check Pod Status & IPs

First, confirm where the pods are running and what IPs they have.

```bash
oc get pods -o wide -A
```

For a specific namespace:

```bash
oc get pods -n <namespace> -o wide
```

Example:

```bash
oc get pods -n myapp -o wide
```

Look at:

- `STATUS`
- `NODE`
- `IP`

You want to identify:

```text
Pod-A → Node-1 → 10.128.1.10
Pod-B → Node-2 → 10.128.2.20
```

Then test connectivity:

```bash
oc exec -it <pod-A> -- ping <pod-B-IP>
```

For example:

```bash
oc exec -it nginx-abc -- ping 10.128.2.20
```

---

# 2. Check OVN Components

Check the OVN pods:

```bash
oc get pods -n openshift-ovn-kubernetes -o wide
```

You should see components such as:

```text
ovnkube-node
ovnkube-master
```

Check specifically:

```bash
oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-node -o wide
```

And:

```bash
oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-master -o wide
```

### What am I checking?

Every node should have a healthy:

```text
ovnkube-node
```

And the OVN control-plane components should be healthy.

Check:

```bash
oc get pods -n openshift-ovn-kubernetes
```

You don't want:

```text
CrashLoopBackOff
Error
Pending
0/...
```

---

# 3. Check OVN Health

Check the OVN database pods:

```bash
oc get pods -n openshift-ovn-kubernetes
```

Then inspect logs if something looks wrong:

```bash
oc logs -n openshift-ovn-kubernetes <ovnkube-node-pod>
```

For example:

```bash
oc logs -n openshift-ovn-kubernetes ovnkube-node-xxxxx
```

For the master:

```bash
oc logs -n openshift-ovn-kubernetes <ovnkube-master-pod>
```

You can also check OVN database status from the relevant OVN pod.

```bash
oc exec -n openshift-ovn-kubernetes -it <ovnkube-master-pod> -- ovn-nbctl show
```

And:

```bash
oc exec -n openshift-ovn-kubernetes -it <ovnkube-master-pod> -- ovn-sbctl show
```

### What are these?

```text
ovn-nbctl
    ↓
Northbound Database
    ↓
Desired network configuration

ovn-sbctl
    ↓
Southbound Database
    ↓
Logical network state for OVN
```

If OVN databases are not reachable or have problems, investigate OVN control-plane health.

---

# 4. Test Overlay Connectivity

Now determine whether the problem is actually between nodes.

Find the pod IPs:

```bash
oc get pods -o wide -A
```

Example:

```text
Pod-A
IP:   10.128.1.10
Node: worker-1

Pod-B
IP:   10.128.2.20
Node: worker-2
```

From Pod-A:

```bash
oc exec -it <pod-A> -- ping 10.128.2.20
```

If this fails, test the nodes themselves.

Find node IPs:

```bash
oc get nodes -o wide
```

Then test:

```bash
ping <remote-node-IP>
```

This helps separate:

```text
Pod → Pod problem
```

from:

```text
Node → Node problem
```

---

# 5. Check OVS

OVN-Kubernetes uses **Open vSwitch (OVS)** on the nodes.

Get a node shell:

```bash
oc debug node/<node-name>
```

Then:

```bash
chroot /host
```

Now you're effectively working on the node.

Check OVS:

```bash
ovs-vsctl show
```

Look for bridges such as:

```text
br-int
br-ex
```

Check OpenFlow flows:

```bash
ovs-ofctl dump-flows br-int
```

This is useful when you suspect that packets are entering OVS but are not being forwarded correctly.

---

# 6. Check Geneve Overlay

For OVN-Kubernetes, pod-to-pod traffic between nodes normally travels through the OVN overlay.

A key component is:

```text
Geneve
UDP 6081
```

Check whether Geneve interfaces exist:

```bash
ip link
```

You may see an interface such as:

```text
genev_sys_6081
```

You can specifically check:

```bash
ip link show | grep -i geneve
```

---

# 7. Check MTU

MTU problems are a **very common overlay-network troubleshooting area**.

Check node interfaces:

```bash
ip link
```

Or:

```bash
ip addr
```

Check the Geneve interface:

```bash
ip link show genev_sys_6081
```

Check pod interface from inside the pod:

```bash
oc exec -it <pod-name> -- ip link
```

You can also test different packet sizes:

```bash
oc exec -it <pod-name> -- ping -M do -s 1400 <remote-pod-IP>
```

If that fails, try:

```bash
oc exec -it <pod-name> -- ping -M do -s 1300 <remote-pod-IP>
```

If smaller packets work but larger packets fail, investigate **MTU/fragmentation**.

---

# 8. Check Firewall

On the node:

```bash
firewall-cmd --list-all
```

Check whether UDP 6081 is allowed:

```bash
firewall-cmd --list-ports
```

You can also check nftables:

```bash
nft list ruleset
```

For OVN Geneve traffic, investigate:

```text
UDP 6081
```

between the nodes.

---

# 9. Check Node Routes

On the node:

```bash
ip route
```

Also:

```bash
ip route get <remote-pod-IP>
```

Example:

```bash
ip route get 10.128.2.20
```

This tells you how the node intends to reach that destination.

---

# 10. Check Network Interfaces

On the affected nodes:

```bash
ip addr
```

Check:

```bash
ip link
```

And:

```bash
ip -s link
```

The last command is particularly useful for identifying interface errors/drops.

Look for:

```text
RX errors
TX errors
dropped
overruns
```

---

# 11. Check OVS Interfaces

```bash
ovs-vsctl show
```

Then:

```bash
ovs-vsctl list interface
```

You can also check OVS port/interface statistics:

```bash
ovs-ofctl dump-ports br-int
```

---

# 12. Packet Capture

If everything looks correct but traffic still fails, use `tcpdump`.

On the node:

```bash
tcpdump -ni any host <remote-node-IP>
```

For Geneve specifically:

```bash
tcpdump -ni any udp port 6081
```

For example:

```bash
tcpdump -ni any udp port 6081
```

Then generate traffic from the pod:

```bash
oc exec -it <pod-A> -- ping <pod-B-IP>
```

You should see Geneve packets on the underlying network.

---

# The Complete Troubleshooting Flow

For your interview, **don't randomly run commands**. Follow this order:

```text
Pod A
  |
  | ping
  ↓
Pod B
  |
  X
FAIL
  |
  ↓
1. Check Pod IPs
   oc get pods -o wide
  |
  ↓
2. Are pods on different nodes?
  |
  ↓
3. Check ovnkube-node
   oc get pods -n openshift-ovn-kubernetes -o wide
  |
  ↓
4. Check OVN health
   ovn-nbctl show
   ovn-sbctl show
  |
  ↓
5. Check node-to-node connectivity
   ping <remote-node-IP>
  |
  ↓
6. Check Geneve
   UDP 6081
  |
  ↓
7. Check OVS
   ovs-vsctl show
   ovs-ofctl dump-flows br-int
  |
  ↓
8. Check routes
   ip route
   ip route get <remote-pod-IP>
  |
  ↓
9. Check MTU
   ping -M do -s <size>
  |
  ↓
10. Check firewall
    firewall-cmd / nft
  |
  ↓
11. tcpdump
    tcpdump -ni any udp port 6081
```

### ⭐ Interview answer

If they ask:

> **"Pods on the same node communicate, but pods on different nodes cannot. How will you troubleshoot?"**

A strong answer is:

> "First, I would verify the pod IPs and confirm that the pods are actually on different nodes using `oc get pods -o wide`. Then I would verify that `ovnkube-node` is healthy on both nodes and check OVN Northbound and Southbound database health. Since same-node communication works, I would focus on the inter-node overlay path. I would verify node-to-node connectivity, Geneve UDP 6081, OVS `br-int` and its flows, node routes, MTU, and firewall rules. Finally, I would use `tcpdump` on the nodes to determine whether Geneve packets are leaving one node and arriving at the other."

That is the **clean troubleshooting story** you should remember.
