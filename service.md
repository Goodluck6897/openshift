OpenShift Learning Series | Service Troubleshooting

Pods are Running. Service exists. But the application is still unreachable. 🤔

This is where simply checking whether resources are Running is not enough.

While exploring Services in OpenShift, I looked at a common troubleshooting scenario:

✅ Pods are Running
✅ Service exists
❌ Application traffic isn’t reaching the Pods

So where do we start?

Imagine the Service is configured with:

selector: app=frontend

But the Pods actually have:

app=web

The Service exists and the Pods are perfectly healthy — but they don’t match.

The result?

Service → No matching endpoints → No backend traffic ❌

This helped me understand an important point:

A Service doesn’t find Pods by their names or Pod IPs.
It selects Pods using labels and selectors, with endpoint data representing the eligible backends.

🔧 My troubleshooting flow

When a Service isn’t reaching its Pods, I’d start with:

oc get svc

→ Is the Service present and are the ports correct?

oc get endpoints <service-name>

→ Does the Service actually have backend endpoints?

oc describe svc <service-name>

→ What selector is configured?

oc get pods --show-labels

→ Do the Pod labels match that selector?

If the endpoint list is empty, label/selector mismatch is one of the first things worth checking—along with Pod readiness and Service port configuration.

Fix the selector or Pod labels:

Labels match → Endpoints populate → Traffic can reach the Pods ✅

💡 Why do we need a Service at all?

Pods are ephemeral.

A Pod might disappear and its replacement can receive a completely different IP address.

Instead of clients tracking individual Pod IPs, the Service provides a stable cluster IP and DNS identity while the backend Pods can change dynamically.

That gives us a simple model:

Stable Service → Dynamic Pods

⸻

🎯 Interview Scenario

Q: Your Pods are Running and the Service exists, but oc get endpoints <service> shows no endpoints. What would you investigate?

I would check:

Service selector → Pod labels → Pod readiness → Service targetPort/ports

rather than immediately troubleshooting the application itself.

⸻

📌 Key takeaway

“Service exists” doesn’t mean “Service has healthy backends.”

Always verify what is actually behind the Service.

Next, this naturally leads to another important OpenShift question:

A Service makes the application reachable inside the cluster — but how does an external user reach it?

That takes us to OpenShift Routes & the Ingress Controller. 🌐

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/df72d65b-5875-4c8e-a45b-848a407e568e" />
