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

![Uploading image.png…]()
