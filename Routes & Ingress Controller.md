

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/44ac38dc-e016-418d-bc73-b74dd28ec5e9" />

OpenShift Learning Series | Routes & Ingress Controller

🌐 Your Service works perfectly inside the cluster… but how does an external user reach it?

Yesterday, I explored Services in OpenShift and how they provide stable access to dynamic Pods.

But a ClusterIP Service is primarily reachable inside the cluster.

So the next question was:

How does traffic from the internet actually reach an application running inside OpenShift?

This is where Routes and the Ingress Controller come into the picture.

A simplified request flow looks like this:

User → DNS → Load Balancer → Router → Route → Service → Pod

Here’s how I understand each part:

🔹 DNS resolves the application hostname to the external entry point.

🔹 Load Balancer sends incoming traffic toward the OpenShift router endpoints.

🔹 Ingress Controller manages the router deployment and configuration.

🔹 Router Pods receive application traffic and evaluate configured Routes.

🔹 Route defines which hostname/path should reach which Service.

🔹 Service provides the stable backend abstraction.

🔹 Pods finally run the application and process the request.

One distinction that helped me:

The Ingress Controller manages the router infrastructure.
Router Pods handle the actual incoming application traffic.
Routes define where that traffic should go.

⸻

🔐 Then comes TLS termination

OpenShift Routes commonly support three TLS termination modes:

EDGE

Client — HTTPS → Router — HTTP → Service/Pod

TLS terminates at the router.

RE-ENCRYPT

Client — HTTPS → Router — HTTPS → Backend

The router terminates the client TLS connection and establishes another encrypted connection to the backend.

PASSTHROUGH

Client — HTTPS → Router → Backend

The router doesn’t terminate TLS; encrypted traffic continues to the backend, which handles TLS.

Understanding where TLS terminates makes these three much easier to remember.

⸻

🚨 Production troubleshooting scenario

Imagine:

✅ Route exists
✅ Service exists
✅ Pods are Running

But…

❌ https://app.example.com isn’t working.

Instead of randomly restarting components, I’d follow the request path:

1. Route — hostname, target Service, port and TLS configuration
2. DNS — does the hostname resolve to the correct ingress endpoint?
3. Service — correct port/targetPort?
4. Endpoints/EndpointSlices — are healthy Pod backends present?
5. Pods — Running and Ready?
6. Router/Ingress — router health, logs, certificates and configuration
7. External path — load balancer/firewall/network connectivity if required

⸻

🎯 Interview Question

What is the difference between an OpenShift Route, Ingress Controller and Router Pod?

My short answer:

Route = traffic rule
Ingress Controller = manages ingress/router infrastructure
Router Pod = processes application ingress traffic
