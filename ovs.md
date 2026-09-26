In OpenShift, **Open vSwitch (OVS)** is a software-based network switch used to connect Pods, Nodes, and the external network.

The important point for interviews is:

> **OVS is the switching layer; OVN-Kubernetes is the networking/control-plane system that configures and uses OVS.**

### Simple picture

```text
                    External Network
                           |
                      Physical NIC
                           |
                    +-------------+
                    |   Node      |
                    |             |
                    |   OVS       |
                    |  Switch     |
                    +-------------+
                     /     |     \
                   Pod1   Pod2   Pod3
```

In an OpenShift cluster using **OVN-Kubernetes**:

```text
              OVN-Kubernetes
                    |
        +-----------+-----------+
        |                       |
   OVN control plane       OVS datapath
        |                       |
  Logical switches       Actual packet
  Logical routers        forwarding
  ACL / policies
```

### What does OVS actually do?

Think of OVS as a **virtual Ethernet switch** running on every node.

It can:

* Forward packets between Pods
* Connect Pod networks to the node
* Forward traffic between nodes
* Handle VLAN/VXLAN/Geneve-related networking
* Apply OpenFlow rules
* Provide the datapath used by OVN-Kubernetes

For example:

```text
Pod A
10.128.1.10
    |
    | veth
    |
  OVS
    |
    | Geneve tunnel
    |
  OVS on Node2
    |
    |
Pod B
10.128.2.20
```

If Pod A and Pod B are on different nodes, **OVN-Kubernetes programs the networking configuration**, while OVS participates in forwarding the packets.

### OVS vs OVN — very important

Don't confuse these in an interview:

| Component          | Think of it as                             |
| ------------------ | ------------------------------------------ |
| **OVS**            | Virtual switch / packet forwarding         |
| **OVN**            | Network virtualization/control plane       |
| **OVN-Kubernetes** | Integrates OVN networking with Kubernetes  |
| **Geneve**         | Tunnel used to carry traffic between nodes |
| **OpenFlow**       | Rules used by OVS for packet processing    |

A simplified flow:

```text
Kubernetes API
      |
      v
OVN-Kubernetes
      |
      v
OVN logical network
      |
      v
OVS on each node
      |
      v
Packets
```

### One interview question you should know

**Q: Does OpenShift use OVS directly to configure the network?**

A good answer:

> "In an OVN-Kubernetes based OpenShift cluster, OVN-Kubernetes manages the logical networking configuration. OVN translates that logical network configuration into flows and datapath configuration that are implemented through Open vSwitch on the nodes. OVS handles the actual packet forwarding."

### One important historical point

Older OpenShift networking commonly used:

```text
OpenShift SDN
      |
      v
Open vSwitch
```

Modern OpenShift uses:

```text
OVN-Kubernetes
      |
      v
Open vSwitch
```

So if you're preparing for an **OpenShift interview**, you should understand **OVS → OVN → OVN-Kubernetes → Geneve → Pod-to-Pod traffic** as one complete topic.


Absolutely. For your OpenShift interview preparation, let's look at a **realistic OVN-Kubernetes node** and understand what each command/output means.

## 1. Check OVS installation and services

On an OpenShift worker node:

```bash
systemctl status ovs-vswitchd
systemctl status ovsdb-server
```

Typical output:

```text
● ovs-vswitchd.service - Open vSwitch Forwarding Unit
   Loaded: loaded
   Active: active (running)
     ...
```

And:

```text
● ovsdb-server.service - Open vSwitch Database Server
   Loaded: loaded
   Active: active (running)
     ...
```

However, in OpenShift, **don't depend only on these systemd commands**. OVS is managed as part of the OVN-Kubernetes deployment.

Check the OVN pods:

```bash
oc get pods -n openshift-ovn-kubernetes -o wide
```

Example:

```text
NAME                                     READY   STATUS    NODE
ovnkube-node-abc12                       6/6     Running   worker01
ovnkube-node-def34                       6/6     Running   worker02
ovnkube-control-plane-xyz12              2/2     Running   master01
```

---

# 2. Check OVS configuration

The most important command:

```bash
ovs-vsctl show
```

Example output:

```text
8e4a1d2b-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    Bridge br-int
        fail_mode: secure
        datapath_type: system
        Port ovn-k8s-mp0
            Interface ovn-k8s-mp0
        Port patch-br-int-to-br-ex
            Interface patch-br-int-to-br-ex
        Port "8a3c..."
            Interface "8a3c..."
        Port "pod123..."
            Interface "pod123..."
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "ens192"
            Interface "ens192"
    ovs_version: "3.x.x"
```

The important pieces are:

```text
br-int
br-ex
```

---

# 3. What is `br-int`?

`br-int` means:

> **Integration Bridge**

This is the main OVS bridge used by OVN-Kubernetes for Pod networking.

Think:

```text
                br-int
                  |
       +----------+----------+
       |          |          |
      Pod1       Pod2       Pod3
```

It participates in forwarding traffic between:

* Pods
* Nodes
* OVN logical networking
* Other OVS components

---

# 4. What is `br-ex`?

`br-ex` means:

> **External Bridge**

It connects the OVN/OVS networking to the node's external network.

Simplified:

```text
Pod
 |
br-int
 |
br-ex
 |
ens192
 |
Physical Network
 |
External Network
```

So when you're troubleshooting **Pod → external network**, `br-ex` becomes particularly important.

---

# 5. List OVS bridges

Command:

```bash
ovs-vsctl list-br
```

Example:

```text
br-ex
br-int
```

Very simple interview question:

**Q: How do you find OVS bridges?**

```bash
ovs-vsctl list-br
```

---

# 6. List ports on `br-int`

```bash
ovs-vsctl list-ports br-int
```

Example:

```text
ovn-k8s-mp0
patch-br-int-to-br-ex
pod1234
pod5678
```

This tells you what ports are attached to the integration bridge.

---

# 7. List ports on `br-ex`

```bash
ovs-vsctl list-ports br-ex
```

Example:

```text
ens192
patch-br-ex-to-br-int
```

So you can visualize:

```text
                 br-int
                   |
          patch-br-int-to-br-ex
                   |
                 br-ex
                   |
                 ens192
                   |
             Physical NIC
```

---

# 8. Check OVS interfaces

```bash
ovs-vsctl list interface
```

You may see:

```text
_uuid               : ...
name                : "ens192"
type                : ""

_uuid               : ...
name                : "br-ex"
type                : "internal"

_uuid               : ...
name                : "patch-br-int-to-br-ex"
type                : "patch"
```

The important field is:

```text
type
```

For example:

```text
type: patch
```

means it is a **patch port connecting OVS bridges**.

---

# 9. Check OVS flows

This is extremely useful during troubleshooting.

```bash
ovs-ofctl dump-flows br-int
```

Example:

```text
cookie=0x0, duration=1200s, table=0,
priority=100,in_port=5,actions=output:10

cookie=0x0, duration=1200s, table=0,
priority=100,in_port=10,actions=output:5

cookie=0x0, duration=1200s, table=0,
priority=50,actions=NORMAL
```

Don't try to memorize individual flow entries for an interview.

Understand the concept:

```text
Packet
  |
  v
OVS
  |
  v
Flow rules
  |
  v
Determine where packet goes
```

---

# 10. Check OVS version

```bash
ovs-vsctl --version
```

Example:

```text
ovs-vsctl (Open vSwitch) 3.x.x
DB Schema 8.x
```

---

# 11. Check OVS database

You can check the OVS database directly:

```bash
ovsdb-client list-dbs
```

Example:

```text
Open_vSwitch
```

Then:

```bash
ovs-vsctl list bridge
```

Example:

```text
_uuid : ...
name  : "br-int"
ports : [...]
```

---

# 12. Check OVN southbound database

With OVN-Kubernetes, you'll also encounter:

```bash
ovn-sbctl show
```

Example:

```text
Chassis "worker01"
    hostname: worker01
    Encap geneve
        ip: "10.10.10.21"
        options: {csum="true"}
    Port_Binding ...
```

This is important because now we're moving from **OVS** into **OVN**.

Think:

```text
             OVN
              |
      Logical networking
              |
              v
            OVS
              |
       Actual packets
```

---

# 13. Check OVN logical switches

```bash
ovn-nbctl show
```

Example:

```text
switch 1234
    switch 1234 (name=transit_switch)
        port ...
        port ...
```

You may also see logical routers:

```bash
ovn-nbctl list logical_router
```

---

# 14. OpenShift troubleshooting workflow

This is the part I recommend remembering for interviews.

Suppose:

> **Pod cannot communicate with another Pod.**

I would troubleshoot roughly like this:

```text
1. Check Pod
       |
       v
2. Check Pod IP
       |
       v
3. Check Pod interface
       |
       v
4. Check OVS
       |
       v
5. Check OVN
       |
       v
6. Check node-to-node network
```

Commands:

### Step 1 — Pod

```bash
oc get pod -o wide
```

Example:

```text
NAME     READY   STATUS    IP            NODE
nginx    1/1     Running   10.128.2.15   worker01
```

### Step 2 — Node

```bash
oc debug node/worker01
```

Then:

```bash
chroot /host
```

### Step 3 — OVS

```bash
ovs-vsctl show
```

### Step 4 — Bridges

```bash
ovs-vsctl list-br
```

### Step 5 — Ports

```bash
ovs-vsctl list-ports br-int
```

### Step 6 — Flows

```bash
ovs-ofctl dump-flows br-int
```

### Step 7 — OVN

```bash
ovn-sbctl show
```

---

# The architecture to remember

For your OpenShift interview, remember this picture:

```text
                    Kubernetes API
                          |
                          v
                  OVN-Kubernetes
                          |
             +------------+------------+
             |                         |
             v                         v
       OVN Northbound             OVN Southbound
       Logical network             Chassis state
             |                         |
             +------------+------------+
                          |
                          v
                       OVS
                 +--------+--------+
                 |                 |
              br-int             br-ex
                 |                 |
              Pod NIC            ens192
                 |                 |
                Pod          Physical Network
```

### Interview answer in 30 seconds

> **"Open vSwitch is the virtual switch used in OpenShift's OVN-Kubernetes networking. OVN-Kubernetes creates the logical network configuration, and OVS provides the datapath that forwards packets on each node. I can use `ovs-vsctl show` to inspect bridges, ports and interfaces, `ovs-vsctl list-br` to list bridges, and `ovs-ofctl dump-flows br-int` to inspect forwarding flows. The key bridges are typically `br-int` for integration and `br-ex` for external connectivity."**

The **next important topic** is understanding exactly how a packet travels **Pod A → br-int → Geneve tunnel → br-int on Node 2 → Pod B**. That is where OVN-Kubernetes, OVS, Geneve, logical switches and logical routers all come together.





