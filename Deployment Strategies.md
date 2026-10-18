OpenShift Learning Series | Deployment Strategies

✈️ Your application v1 is running perfectly.

Now v2 is ready for production.

The question isn’t just:

“How do I deploy v2?”

The better question is:

“How do I deploy v2 without breaking what’s already working?”

While exploring OpenShift deployments, I learned that there are several ways to approach a new application release — and each comes with different trade-offs around downtime, risk, infrastructure, and rollback.

I tried to visualize them as 5 different runways to production. ✈️

1️⃣ Recreate

Stop all v1 Pods first, then start v2.

v1 → STOP → v2

Simple, but there is a period of downtime.

Best suited when downtime is acceptable.

2️⃣ Rolling Update

Gradually replace old Pods with new ones.

🟢🟢🟢 → 🟢🟢🔵 → 🟢🔵🔵 → 🔵🔵🔵

This is the standard strategy for Kubernetes/OpenShift Deployments.

Settings such as maxSurge and maxUnavailable control how aggressively the rollout happens.

3️⃣ Blue-Green

Run two complete environments:

🔵 Blue → v1

🟢 Green → v2

Deploy and validate v2 separately, then switch production traffic to it.

One major advantage?

👉 If something goes wrong, traffic can potentially be switched back quickly.

The trade-off is additional infrastructure while both environments exist.

4️⃣ Canary

Don’t expose everyone to v2 immediately.

Start small:

90% → v1

10% → v2

Monitor the new version.

If everything looks healthy:

70/30 → 50/50 → 100% v2

This limits the blast radius of a bad release.

5️⃣ A/B Testing

This looks similar to Canary, but the goal is different.

Canary asks:

“Is v2 safe enough to release?”

A/B testing asks:

“Which version performs better for a particular group of users?”

Different user groups can intentionally receive different versions while metrics are compared.

⸻

💡 One distinction I found important:

Recreate and RollingUpdate are strategies directly supported by a Kubernetes Deployment.

Blue-Green, Canary and A/B are broader release/traffic-management patterns. In OpenShift, their implementation can involve multiple workloads plus Services/Routes, ingress or service-mesh/GitOps capabilities depending on the architecture.

Understanding that distinction is important in interviews as well as production design.

⸻

📌 My takeaway:

There is no single “best” deployment strategy.

The right strategy depends on risk tolerance, downtime requirements, rollback expectations, infrastructure cost, and how much control you need over traffic.

💬 What deployment strategy does your team use most often in production — Rolling Update, Blue-Green, Canary, or something else?
<img width="1149" height="1369" alt="image" src="https://github.com/user-attachments/assets/adb0ee59-f55d-41fb-b850-61680acd558c" />
