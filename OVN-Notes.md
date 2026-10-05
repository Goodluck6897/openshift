
# OVN — Open Virtual Network

![Uploading image.png…]()
![Uploading image.png…]()


## What is OVN?

OVN (Open Virtual Network) is a set of **daemons** (background services) that work on top of **Open vSwitch (OVS)**. Its main job is to take **virtual network configurations** and translate them into **OpenFlow rules** that OVS can understand and execute.

> **Think of it this way:** OVS handles the low-level traffic forwarding (flows), while OVN sits on top and lets you work with **logical networking concepts** like routers and switches — making network management much simpler.

---

## License

- OVN is **open source**, licensed under the **Apache 2.0 License**.

---

## How OVN Differs from Open vSwitch

| Aspect | Open vSwitch (OVS) | OVN |
|---|---|---|
| **Abstraction Level** | Low-level — works with **flows** | High-level — works with **logical routers & switches** |
| **Target User** | Network engineers managing flow rules | Cloud Management Software (CMS) platforms |
| **Role** | Forwards packets based on OpenFlow rules | Translates logical network configs into OVS flows |

---

## Key Features

- **Distributed Virtual Routers** — Route traffic between different virtual networks across multiple hosts, without a single bottleneck.
- **Distributed Logical Switches** — Create virtual Layer 2 switches that span across many physical machines.
- **Access Control Lists (ACLs)** — Define security rules to allow or deny traffic at the logical level.
- **DHCP** — Automatically assign IP addresses to virtual machines without needing an external DHCP server.
- **DNS Server** — Built-in DNS resolution for virtual networks.

---

## Architecture Overview

- OVN is designed to be used by **Cloud Management Software (CMS)** such as OpenStack, Kubernetes (via OVN-Kubernetes), or your **OpenShift cluster**.
- For full architecture details, refer to the `ovn-architecture` manpage:
  ```bash
  man ovn-architecture
  ```

---

## Technical Details

- **Runs in Userspace:** OVN runs **entirely in userspace** — **no kernel modules** need to be installed.
  - This makes installation and upgrades simpler and safer.

---

## Quick Summary

```
OVN = Higher-level abstraction over Open vSwitch
         ↓
Logical Routers + Logical Switches
         ↓
Translated into OpenFlow rules for OVS
         ↓
OVS handles actual packet forwarding
```

---

## Useful Links

- [OVN GitHub Repository](https://github.com/ovn-org/ovn)
- [Open vSwitch Official Site](https://www.openvswitch.org/)
- [OVN Architecture Manpage](https://man7.org/linux/man-pages/man7/ovn-architecture.7.html)

- https://man7.org/linux/man-pages/man7/ovn-architecture.7.html

- <img width="1187" height="602" alt="image" src="https://github.com/user-attachments/assets/def69c38-f451-429a-a8bf-010133f2001f" />

