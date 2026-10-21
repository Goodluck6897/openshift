🚀 Kubernetes/OpenShift Pod Scheduling: How does the Scheduler actually decide WHERE your Pod runs?
We often say:
“The scheduler schedules the Pod on a node.”
But that one sentence hides a surprisingly intelligent decision-making process.
When a Pod is created and doesn’t yet have a node assigned, kube-scheduler starts looking for the best place to run it.
Think of the process in two major stages:
🔎 1. Filtering — Which nodes CAN run this Pod?
The scheduler eliminates nodes that don't satisfy the Pod's requirements.
This is where things like these become important:
🔹 nodeSelector — “Run me only on nodes with these labels.”
🔹 Node Affinity — A more expressive version of nodeSelector.
requiredDuringScheduling... acts as a hard requirement, while preferredDuringScheduling... expresses a preference.
🔹 Pod Affinity — “Place me near Pods matching these labels.”
Useful when workloads benefit from being co-located.
🔹 Pod Anti-Affinity — “Keep me away from Pods matching these labels.”
Very useful for spreading replicas across nodes or failure domains for better availability.
🔹 Taints & Tolerations — Nodes can repel Pods using taints, while Pods with matching tolerations are allowed onto those nodes.
And of course, the scheduler also considers things such as available resources and node conditions.
📊 2. Scoring — Which eligible node is the BEST node?
Passing the filters doesn't automatically mean a node gets the Pod.
The remaining nodes are scored based on scheduling preferences and configured scheduler plugins.
Imagine:
Node-1 ❌ Filtered
Node-2 ❌ Filtered
Node-3 ✅ Score: 72
Node-4 ✅ Score: 95
🏆 Node-4 wins.
The scheduler then communicates the decision through the API Server and the Pod is bound to the selected node.
That's the part I find interesting:
Pod scheduling isn't simply about finding a node with enough CPU and memory.
It's about expressing intent.
nodeSelector → Where can I run?
nodeAffinity → Where should I run?
podAffinity → Who should I stay close to?
podAntiAffinity → Who should I stay away from?
taints/tolerations → Which workloads are allowed on this node?
Once you understand that distinction, Kubernetes scheduling becomes much easier to reason about — especially when troubleshooting a Pod sitting stubbornly in Pending. 😄
The diagram below captures the complete journey:
Pod → Filter → Score → Select → Bind → Node
Understanding the why behind scheduling is much more valuable than simply memorizing the YAML.

<img width="1222" height="1287" alt="image" src="https://github.com/user-attachments/assets/ee8ecdd1-23e4-4848-aee0-84a4dc447a1f" />
