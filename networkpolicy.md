OpenShift Learning Series | NetworkPolicy

Your Pods can communicate with each other…

But should they? 🤔

Imagine a simple application:

Frontend → Backend → Database

Our requirement:

✅ Frontend → Backend
✅ Backend → Database
❌ Frontend → Database

This is where NetworkPolicy becomes important.

It allows us to control which Pods can communicate, on which ports, and in which direction.

Two concepts to remember:

➡️ Ingress = Traffic coming into a Pod
⬅️ Egress = Traffic leaving a Pod

🚨 Production Scenario

Pods are Running.
Service has Endpoints.
DNS is working.

But Frontend still can’t connect to Backend.

Before restarting anything, check:

oc get networkpolicy -n <namespace>

Then verify the selectors, labels, ports, ingress and egress rules.

💡 One important concept:

NetworkPolicies are additive.

Multiple policies can collectively allow traffic to a selected Pod.

📌 Key takeaway:

Running ≠ Reachable.
Reachable ≠ Should be allowed.

Use NetworkPolicy to allow only the communication your application actually needs.

💬 Do you use default-deny NetworkPolicies in your environment?

![Uploading image.png…]()
