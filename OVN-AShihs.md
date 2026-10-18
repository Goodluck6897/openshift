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
