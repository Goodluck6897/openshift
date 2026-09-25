Absolutely. I reviewed the **OpenShift Container Platform 4.18 OVN-Kubernetes documentation** and converted the material into notes aimed at **real project work + OpenShift administrator interviews**, rather than simply copying the Red Hat documentation. ([Red Hat Docs][1])

# OpenShift OVN-Kubernetes — Project-Ready Notes

## 1. What is OVN-Kubernetes?

**OVN-Kubernetes is the default network plugin/CNI for OpenShift Container Platform.**

It provides networking for:

* Pods
* Services
* Nodes
* East-west traffic
* North-south traffic
* NetworkPolicy
* Egress traffic
* Multicast
* IPv4/IPv6
* Egress IP
* Firewall functionality
* IPsec
* Hybrid networking

OVN-Kubernetes is built using:

```text
Kubernetes
    │
    ▼
OVN-Kubernetes
    │
    ▼
OVN
    │
    ▼
Open vSwitch (OVS)
    │
    ▼
Physical Network
```

The important point is:

> **OVN decides how the virtual network should behave, and OVS implements that networking on each node.**

OpenShift uses **Geneve** for the overlay network rather than VXLAN. ([Red Hat Docs][1])

---

# 2. The 3 technologies you must understand

Think of OVN-Kubernetes as three layers.

```text
┌─────────────────────────────────────────────┐
│              OVN-Kubernetes                 │
│     Kubernetes/OpenShift integration        │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                    OVN                      │
│ Logical switches / routers / flows / policy │
└──────────────────────┬──────────────────────┘
                       │
┌──────────────────────▼──────────────────────┐
│                    OVS                      │
│      Actual packet forwarding on node       │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
                Physical network
```

### OVN

**Open Virtual Network**

Responsible for the logical network.

It understands concepts such as:

* Logical switch
* Logical router
* Logical port
* Logical flows
* Network policy
* Routing

### OVS

**Open vSwitch**

Actually forwards packets on the node.

OVN programs OVS with flows.

### OVN-Kubernetes

Connects Kubernetes/OpenShift networking requirements with OVN.

For example:

```text
Pod created
   ↓
OVN-Kubernetes
   ↓
Create logical networking
   ↓
OVN
   ↓
Generate logical flows
   ↓
OVN Controller
   ↓
OpenFlow rules
   ↓
OVS
```

This distinction is **very important in interviews**. ([Red Hat Docs][2])

---

# 3. Why is it called an overlay network?

Pods can communicate even though their Pod IP addresses are different from the underlying physical network.

Example:

```text
Node 1
Physical IP: 10.10.1.10
Pod:         10.128.1.20

             │
             │ Geneve
             ▼

Node 2
Physical IP: 10.10.2.10
Pod:         10.129.2.30
```

The physical network doesn't necessarily need to understand:

```text
10.128.1.20 → 10.129.2.30
```

Instead, OVN-Kubernetes encapsulates the traffic using **Geneve** and sends it across the physical network.

```text
Original packet

Pod A
10.128.1.20
     │
     ▼
OVS
     │
     ▼
Geneve encapsulation
     │
     ▼
Physical network
     │
     ▼
Node 2 OVS
     │
     ▼
Geneve decapsulation
     │
     ▼
Pod B
10.129.2.30
```

Red Hat specifically documents Geneve as the overlay protocol used by OVN-Kubernetes. ([Red Hat Docs][1])

---

# 4. OVN-Kubernetes architecture

This is the architecture you should be able to draw in an interview.

```text
                 Kubernetes / OpenShift
                         │
                         ▼
                OVN-Kubernetes
                         │
                         ▼
               ┌─────────────────┐
               │ OVN Northbound  │
               │     DB          │
               └────────┬────────┘
                        │
                        ▼
                  ovn-northd
                        │
                        ▼
               ┌─────────────────┐
               │ OVN Southbound  │
               │      DB         │
               └────────┬────────┘
                        │
                        ▼
                 ovn-controller
                        │
                        ▼
                 OpenFlow rules
                        │
                        ▼
                       OVS
                        │
                        ▼
                 Physical Network
```

There are several components you need to understand.

---

# 5. OVN Northbound Database — NBDB

### Purpose

The **Northbound database** contains the **desired logical network configuration**.

Think:

> "What should the network look like?"

It contains logical objects such as:

* Logical switches
* Logical routers
* Logical ports
* Policies
* Network configuration

Example:

```text
NBDB

Logical Switch
     │
     ├── Pod A
     ├── Pod B
     └── Pod C

Logical Router
     │
     ├── Network A
     └── Network B
```

### Interview answer

> "The OVN Northbound database stores the logical network configuration or desired state."

---

# 6. ovn-northd

`ovn-northd` is the translator.

It reads:

```text
Northbound DB
```

and generates:

```text
logical flows
```

which are stored in:

```text
Southbound DB
```

Think:

```text
NBDB
 │
 │ Logical configuration
 ▼
ovn-northd
 │
 │ Logical flows
 ▼
SBDB
```

### Interview answer

> "ovn-northd translates the logical network configuration from the Northbound database into logical datapath flows stored in the Southbound database."

([Red Hat Docs][2])

---

# 7. OVN Southbound Database — SBDB

The **Southbound database** contains information required to implement the network on the physical nodes.

It knows things such as:

* Chassis information
* Physical/logical bindings
* Logical flows
* Node information
* How logical objects map to physical nodes

Think:

> NBDB = desired logical network

> SBDB = information required to implement that network

---

# 8. ovn-controller

`ovn-controller` runs on the nodes and communicates with:

```text
SBDB
```

It takes logical flows and converts them into:

```text
OpenFlow rules
```

which are installed into:

```text
Open vSwitch
```

So:

```text
SBDB
  │
  ▼
ovn-controller
  │
  ▼
OpenFlow
  │
  ▼
OVS
```

### Very important interview statement

> **ovn-controller translates OVN logical flows into OpenFlow rules and programs OVS on the node.**

([Red Hat Docs][2])

---

# 9. OVS — Open vSwitch

OVS is the actual virtual switch running on each node.

Simplified:

```text
              Node
┌─────────────────────────────────┐
│                                 │
│ Pod A ──┐                       │
│         │                       │
│ Pod B ──┼──► OVS ──► Physical NIC
│         │                       │
│ Pod C ──┘                       │
│                                 │
└─────────────────────────────────┘
```

OVN tells OVS:

> "If you receive this packet, send it here."

OVS then performs the actual forwarding.

---

# 10. What runs on OpenShift nodes?

One of the most important things to remember is:

```text
openshift-ovn-kubernetes
```

namespace.

You will commonly see:

```bash
oc get pods -n openshift-ovn-kubernetes
```

Typical important components include:

```text
ovnkube-control-plane
ovnkube-node
```

The `ovnkube-node` components are associated with the node-level OVN/OVS networking functionality. Red Hat also provides readiness probes and health information for these pods. ([Red Hat Docs][3])

---

# 11. Control-plane vs node networking

Conceptually:

```text
CONTROL PLANE
─────────────────────────────

OVN databases
     │
ovn-northd
     │
logical network state


WORKER NODE
─────────────────────────────

ovn-controller
     │
     ▼
    OVS
     │
     ▼
Pods / Services / Network
```

The databases and OVN control components maintain the network state, while node-level components program the local OVS datapath.

---

# 12. Pod-to-Pod communication

This is one of the most important project scenarios.

Suppose:

```text
Pod A
Node 1
IP = 10.128.1.10

       wants to communicate with

Pod B
Node 2
IP = 10.129.2.20
```

### Step 1 — Pod sends packet

```text
Pod A
10.128.1.10
      │
      ▼
OVS
```

### Step 2 — OVS checks flows

OVS has flows programmed by OVN.

```text
OVS
 │
 ├── destination local?
 │
 └── destination remote?
```

### Step 3 — Remote node

If Pod B is on another node:

```text
OVS
 │
 ▼
Geneve encapsulation
 │
 ▼
Physical network
 │
 ▼
Node 2
 │
 ▼
OVS
```

### Step 4 — Destination pod

```text
OVS
 │
 ▼
Pod B
```

So the conceptual packet path is:

```text
Pod
 ↓
OVS
 ↓
Geneve
 ↓
Physical Network
 ↓
OVS
 ↓
Pod
```

---

# 13. What is Geneve?

**Geneve = Generic Network Virtualization Encapsulation**

OVN-Kubernetes uses Geneve to create the overlay network.

Important interview comparison:

| Technology              | OpenShift OVN-Kubernetes         |
| ----------------------- | -------------------------------- |
| Overlay                 | Yes                              |
| VXLAN                   | Not the primary overlay protocol |
| Geneve                  | Yes                              |
| Default Geneve UDP port | 6081                             |
| Purpose                 | Encapsulate overlay traffic      |

The Geneve port is configurable through OVN-Kubernetes networking configuration; Red Hat documents UDP **6081** as the default. ([Red Hat Docs][4])

---

# 14. Logical Switch

A logical switch is a virtual Layer-2 switching construct.

Example:

```text
Logical Switch
      │
 ┌────┼────┐
 │    │    │
Pod A Pod B Pod C
```

It is logical rather than a physical Ethernet switch.

OVN creates the logical network, and OVS implements the forwarding.

---

# 15. Logical Router

OVN also provides logical routing.

Example:

```text
             Logical Router
             /            \
            /              \
           ▼                ▼
     Pod Network A     Pod Network B
```

This allows traffic to move between logical networks.

OVN provides distributed virtual routing, which is one of its important capabilities. ([Red Hat Docs][1])

---

# 16. Distributed routing

This is an important difference from thinking about a traditional centralized router.

Instead of:

```text
Pod
 ↓
Central Router
 ↓
Destination
```

OVN can implement routing in a distributed fashion:

```text
Node 1                         Node 2

Pod A                          Pod B
 │                              ▲
 ▼                              │
OVS ─── distributed routing ─── OVS
```

This helps avoid sending every packet through a centralized router.

---

# 17. Join subnet

OVN-Kubernetes uses an internal **join network**.

The join switch connects logical routers in the OVN architecture.

Default IPv4 join subnet in OCP 4.18:

```text
100.64.0.0/16
```

Default IPv6:

```text
fd98::/64
```

The subnet must not overlap with other cluster or host networks. ([Red Hat Docs][3])

### Easy way to remember

```text
Join subnet
     ↓
Connects OVN logical routing components
```

---

# 18. Transit subnet

OVN-Kubernetes also uses a **transit switch**.

Default IPv4:

```text
100.88.0.0/16
```

Default IPv6:

```text
fd97::/64
```

The transit subnet is used by the distributed transit switch and supports east-west traffic between nodes. ([Red Hat Docs][3])

Remember:

```text
JOIN
→ connects logical routing components

TRANSIT
→ distributed transit network / east-west traffic
```

---

# 19. Masquerade subnet

OVN-Kubernetes also has an internal masquerade subnet.

For new OCP 4.17+ clusters, Red Hat documents the IPv4 default as:

```text
169.254.0.0/17
```

Older clusters upgraded to 4.17 can retain:

```text
169.254.169.0/29
```

The important operational rule is:

> **Internal OVN subnets must not overlap with your existing host/OpenShift network ranges.**

([Red Hat Docs][3])

---

# 20. br-ex

You will frequently hear about:

```text
br-ex
```

Think of it as the OVS bridge associated with the node's external network connectivity.

Simplified:

```text
                Pod
                 │
                 ▼
                OVS
                 │
                 ▼
              br-ex
                 │
                 ▼
             Node NIC
                 │
                 ▼
        Physical Network
```

This is particularly important when troubleshooting:

* External connectivity
* Egress
* Node gateway
* LoadBalancer traffic
* North-south traffic

---

# 21. Egress traffic

Suppose a pod wants to access:

```text
google.com
```

Conceptually:

```text
Pod
 │
 ▼
OVS
 │
 ▼
OVN routing
 │
 ▼
Node gateway / br-ex
 │
 ▼
Physical network
 │
 ▼
Firewall
 │
 ▼
Internet
```

OVN-Kubernetes supports different gateway behaviors for egress traffic. Red Hat documents `Shared` and `Local` gateway configurations. ([Red Hat Docs][5])

---

# 22. NetworkPolicy

OVN-Kubernetes implements Kubernetes NetworkPolicy.

Example:

```text
Namespace A
   │
   │ allowed
   ▼
Namespace B

Namespace C
   │
   │ denied
   ▼
Namespace B
```

Policies can control:

```text
Ingress
Egress
```

OVN translates the policy requirements into network flows that are enforced in the datapath. OVN-Kubernetes also supports network-policy logging. ([Red Hat Docs][1])

---

# 23. Egress IP

Egress IP allows traffic from selected workloads to appear externally with a specific source IP.

Example:

```text
Pod
10.128.x.x
    │
    ▼
OVN
    │
    ▼
Egress IP
192.168.10.50
    │
    ▼
External application
```

External systems therefore see:

```text
192.168.10.50
```

rather than the Pod IP.

OVN-Kubernetes supports egress IP functionality. ([Red Hat Docs][1])

---

# 24. NetworkPolicy vs EgressFirewall vs EgressIP

These are often confused.

| Feature        | Purpose                                      |
| -------------- | -------------------------------------------- |
| NetworkPolicy  | Control allowed/denied pod traffic           |
| EgressFirewall | Restrict where workloads can send traffic    |
| EgressIP       | Give selected workloads a specific source IP |
| Gateway        | Control how traffic leaves the cluster       |

Think:

```text
NetworkPolicy
    ↓
"Can Pod A talk to Pod B?"

EgressFirewall
    ↓
"Can this namespace talk to that external network?"

EgressIP
    ↓
"What source IP should external systems see?"
```

---

# 25. OVN-Kubernetes troubleshooting

This is where project knowledge becomes important.

Start here:

```bash
oc get pods -n openshift-ovn-kubernetes
```

Then:

```bash
oc get pods -n openshift-ovn-kubernetes -o wide
```

Check:

```text
ovnkube-control-plane
ovnkube-node
```

---

# 26. Check OVN daemon status

```bash
oc get pods -n openshift-ovn-kubernetes
```

Look for:

```text
Running
Ready
Restart count
CrashLoopBackOff
```

Then:

```bash
oc get daemonsets -n openshift-ovn-kubernetes
```

Red Hat also recommends checking the OVN namespace and logs when determining OVN-Kubernetes status. ([Red Hat Docs][6])

---

# 27. Check Network Operator

Very important:

```bash
oc get co/network
```

More detailed:

```bash
oc get co/network -o json | jq '.status.conditions[]'
```

You want to understand whether:

```text
Available
Progressing
Degraded
```

is healthy.

Red Hat specifically recommends checking the Network ClusterOperator conditions when troubleshooting OVN-Kubernetes readiness. ([Red Hat Docs][3])

---

# 28. Check OVN events

```bash
oc get events -n openshift-ovn-kubernetes
```

Specific pod:

```bash
oc describe pod <ovnkube-pod> \
  -n openshift-ovn-kubernetes
```

This can reveal:

* Readiness failures
* Container failures
* Mount problems
* Configuration issues
* Restart causes

([Red Hat Docs][3])

---

# 29. Check logs

First identify the pod:

```bash
oc get pods -n openshift-ovn-kubernetes
```

Then:

```bash
oc logs <pod> -n openshift-ovn-kubernetes
```

If the pod has multiple containers:

```bash
oc logs <pod> \
  -n openshift-ovn-kubernetes \
  -c <container>
```

For example, investigate the relevant OVN components individually.

---

# 30. Check which node has the OVN pod

```bash
oc get pods \
  -n openshift-ovn-kubernetes \
  -o wide
```

You might see:

```text
NAME             NODE
ovnkube-node-x   worker01
ovnkube-node-y   worker02
ovnkube-node-z   worker03
```

This is very useful when troubleshooting a problem affecting a particular worker.

---

# 31. Project troubleshooting scenario

### Problem

Application Pod A cannot communicate with Pod B.

Don't immediately blame the application.

Follow:

```text
1. Pod status
      ↓
2. Pod IP
      ↓
3. Node placement
      ↓
4. Service / endpoint if applicable
      ↓
5. NetworkPolicy
      ↓
6. ovnkube-node
      ↓
7. OVS
      ↓
8. Geneve/node connectivity
      ↓
9. Physical network
```

Commands:

```bash
oc get pod -o wide -n <namespace>
```

Then:

```bash
oc get networkpolicy -n <namespace>
```

Then:

```bash
oc get pods -n openshift-ovn-kubernetes -o wide
```

Then:

```bash
oc get co/network
```

Then:

```bash
oc logs <ovnkube-node-pod> \
  -n openshift-ovn-kubernetes
```

---

# 32. If Pod-to-Pod communication fails

Use this mental model:

```text
             Pod A
               │
               ▼
             CNI
               │
               ▼
              OVS
               │
        ┌──────┴──────┐
        │             │
     local          remote
        │             │
        ▼             ▼
      Pod B         Geneve
                      │
                      ▼
                Physical NIC
                      │
                      ▼
                   Node B
                      │
                      ▼
                     OVS
                      │
                      ▼
                    Pod B
```

Check each layer independently.

---

# 33. If external connectivity fails

Example:

```text
Pod → Internet
```

Check:

```text
Pod IP
   ↓
OVN
   ↓
Gateway
   ↓
br-ex
   ↓
Node NIC
   ↓
Physical network
   ↓
Firewall
   ↓
Internet
```

Useful commands:

```bash
oc exec -it <pod> -- ping <destination>
```

```bash
oc exec -it <pod> -- curl <destination>
```

Check node:

```bash
oc debug node/<node>
```

Then inspect networking from the node environment as appropriate.

---

# 34. Important OVN-Kubernetes features

You should know these without looking them up:

```text
OVN-Kubernetes
│
├── Overlay networking
├── Geneve
├── Open vSwitch
├── Logical switches
├── Logical routers
├── Distributed routing
├── NetworkPolicy
├── Egress IP
├── Egress firewall
├── Multicast
├── IPv4
├── IPv6
├── Dual-stack
├── IPsec
├── Hardware offload
└── Hybrid networking
```

Red Hat lists these capabilities in the OCP 4.18 documentation. ([Red Hat Docs][1])

---

# 35. OVN vs OVS — don't confuse them

This is probably the **most important distinction**.

| Component      | Responsibility                                  |
| -------------- | ----------------------------------------------- |
| OVN            | Defines/manages virtual network                 |
| NBDB           | Logical desired state                           |
| ovn-northd     | Converts logical configuration to logical flows |
| SBDB           | Stores logical/physical state and flows         |
| ovn-controller | Converts logical flows to OpenFlow              |
| OVS            | Performs packet forwarding                      |
| Geneve         | Encapsulates overlay traffic                    |
| OVN-Kubernetes | Integrates OVN networking with Kubernetes       |

### Easy memory trick

```text
NB
↓
Northd
↓
SB
↓
Controller
↓
OpenFlow
↓
OVS
↓
Packet
```

**NB → Northd → SB → Controller → OVS**

Memorize this sequence.

---

# 36. OpenShift networking troubleshooting commands — cheat sheet

```bash
# Check network operator
oc get co/network

# Detailed network operator status
oc get co/network -o yaml

# Check OVN pods
oc get pods -n openshift-ovn-kubernetes

# Check OVN pods + nodes
oc get pods -n openshift-ovn-kubernetes -o wide

# Check daemonsets
oc get daemonsets -n openshift-ovn-kubernetes

# OVN events
oc get events -n openshift-ovn-kubernetes

# Describe OVN pod
oc describe pod <pod> -n openshift-ovn-kubernetes

# OVN logs
oc logs <pod> -n openshift-ovn-kubernetes

# Specific container
oc logs <pod> \
  -n openshift-ovn-kubernetes \
  -c <container>

# Check application pod IP/node
oc get pods -o wide -n <namespace>

# Check network policies
oc get networkpolicy -A

# Check cluster network configuration
oc get network.config.openshift.io cluster -o yaml

# Check OVN network configuration
oc get network.operator.openshift.io cluster -o yaml
```

The Red Hat documentation specifically recommends the OVN namespace, readiness probes, events, Network ClusterOperator status, and OVN logs as key health/troubleshooting sources. ([Red Hat Docs][3])

---

# 37. Interview questions you should prepare

### Basic

**Q1. What is OVN-Kubernetes?**

> OVN-Kubernetes is OpenShift's default CNI/network plugin. It uses OVN to manage virtual networking and Open vSwitch to implement the networking on cluster nodes.

---

**Q2. What is the difference between OVN and OVS?**

> OVN provides the logical networking and generates logical flows, while OVS is the virtual switch that actually forwards packets.

---

**Q3. What protocol does OVN-Kubernetes use for the overlay?**

> Geneve.

---

**Q4. What is ovn-northd?**

> It translates the logical network configuration in the OVN Northbound database into logical flows stored in the Southbound database.

---

**Q5. What does ovn-controller do?**

> It reads logical flows from the Southbound database, converts them into OpenFlow rules, and programs OVS.

---

**Q6. What is NBDB?**

> Northbound database containing logical network configuration.

---

**Q7. What is SBDB?**

> Southbound database containing physical/logical network state, bindings, and logical flows used to program the nodes.

---

# 38. Scenario question

### "A pod on worker01 cannot communicate with a pod on worker02. How do you troubleshoot?"

A strong project answer:

> "First I verify both pods are running and get their IP addresses and node placement using `oc get pods -o wide`. Then I check NetworkPolicies. If the policy is not the issue, I check the `ovnkube-node` pod on both nodes, its readiness and logs, and the Network ClusterOperator status. Then I investigate the OVN/OVS datapath and Geneve connectivity between the nodes. Finally, if OVN looks healthy, I check the underlying node network and physical network."

That demonstrates **layer-by-layer troubleshooting** rather than just knowing commands.

---

# 39. Scenario: Pod cannot access Internet

Your troubleshooting flow:

```text
             Pod
              │
              ▼
         Pod networking
              │
              ▼
             OVS
              │
              ▼
        OVN gateway
              │
              ▼
            br-ex
              │
              ▼
          Node NIC
              │
              ▼
       Physical network
              │
              ▼
          Firewall
              │
              ▼
          Internet
```

Check:

```bash
oc exec <pod> -- curl -v https://example.com
```

Then investigate:

```text
DNS
↓
routing
↓
OVN
↓
gateway
↓
br-ex
↓
node
↓
firewall
```

---

# 40. The architecture you should memorize

For your **OpenShift administrator interviews**, I would memorize this diagram:

```text
                       Kubernetes
                           │
                           ▼
                   OVN-Kubernetes
                           │
                           ▼
                  ┌────────────────┐
                  │      NBDB      │
                  │ Northbound DB  │
                  └───────┬────────┘
                          │
                          ▼
                     ovn-northd
                          │
                          ▼
                  ┌────────────────┐
                  │      SBDB      │
                  │ Southbound DB  │
                  └───────┬────────┘
                          │
                          ▼
                   ovn-controller
                          │
                    OpenFlow rules
                          │
                          ▼
                       OVS
                    ┌─────┴─────┐
                    │           │
                  Pod          br-ex
                    │           │
                    │           ▼
                    │      Physical NIC
                    │           │
                    │       Network
                    │
                    ▼
                   Pod
```

And for remote Pod traffic:

```text
Pod A
 │
 ▼
OVS
 │
 ▼
Geneve
 │
 ▼
Physical Network
 │
 ▼
OVS
 │
 ▼
Pod B
```

---

# 41. One-page project memory

If you remember only this:

```text
OVN-Kubernetes
        │
        ├── Default OpenShift CNI
        │
        ├── Uses OVN
        │
        ├── Uses OVS on nodes
        │
        ├── Geneve overlay
        │
        ├── Logical switches
        │
        ├── Logical routers
        │
        ├── Distributed routing
        │
        ├── NetworkPolicy
        │
        ├── Egress IP
        │
        ├── Egress Firewall
        │
        ├── IPv4/IPv6/Dual-stack
        │
        └── IPsec/Multicast/etc.


Traffic control flow:

NBDB
 ↓
ovn-northd
 ↓
SBDB
 ↓
ovn-controller
 ↓
OpenFlow
 ↓
OVS
 ↓
Packet


Remote Pod:

Pod
 ↓
OVS
 ↓
Geneve
 ↓
Physical Network
 ↓
OVS
 ↓
Pod
```

**This is the level of OVN-Kubernetes knowledge I would consider the foundation for project-ready OpenShift administration.**

For the next step, the most valuable thing is to learn **OVN-Kubernetes practically**: **Pod → OVS → Geneve → remote Pod**, **Pod → Service**, **Pod → external network**, and **NetworkPolicy**, using real `oc`, `ovs-vsctl`, `ovn-nbctl`, `ovn-sbctl`, and troubleshooting commands. The Red Hat 4.18 guide also has dedicated architecture and troubleshooting sections that support this operational approach. ([Red Hat Docs][2])

[Red Hat OCP 4.18 — OVN-Kubernetes documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ovn-kubernetes_network_plugin/about-ovn-kubernetes?utm_source=chatgpt.com)

[1]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ovn-kubernetes_network_plugin/about-ovn-kubernetes "Chapter 1. About the OVN-Kubernetes network plugin | OVN-Kubernetes network plugin | OpenShift Container Platform | 4.18 | Red Hat Documentation"
[2]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/ovn-kubernetes_network_plugin/ovn-kubernetes-architecture-assembly?utm_source=chatgpt.com "Chapter 2. OVN-Kubernetes architecture | OVN-Kubernetes network plugin | OpenShift Container Platform | 4.18 | Red Hat Documentation"
[3]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html-single/ovn-kubernetes_network_plugin/index?utm_source=chatgpt.com "OVN-Kubernetes network plugin | OpenShift Container Platform | 4.18 | Red Hat Documentation"
[4]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html-single/networking/index?utm_source=chatgpt.com "Networking | OpenShift Container Platform | 4.16 | Red Hat Documentation"
[5]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/installing_on_bare_metal/user-provisioned-infrastructure?utm_source=chatgpt.com "Chapter 2. User-provisioned infrastructure | Installing on bare metal | OpenShift Container Platform | 4.18 | Red Hat Documentation"
[6]: https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/pdf/support/getting-support?utm_source=chatgpt.com "Support | OpenShift Container Platform | 4.18 | Red Hat Documentation"
