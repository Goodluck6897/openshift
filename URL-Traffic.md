How does traffic reach your app in OpenShift? 🚀
Every request to your application passes through the HAProxy-based Ingress Router. Here is the journey:
1️⃣ Client requests https://lnkd.in/gkuRMZ2C
2️⃣ DNS resolves the wildcard *.apps domain to the Load Balancer VIP
3️⃣ The Load Balancer forwards traffic to the Router nodes
4️⃣ HAProxy terminates TLS and matches the Route by host and path
5️⃣ The Service picks a healthy Endpoint
6️⃣ The App Pod serves the response
💡 Quick tips for admins:
• Router pods run in the openshift-ingress namespace
• Check them with oc get pods -n openshift-ingress
• Route types: edge, passthrough, re-encrypt
• Tune timeouts with route annotations when needed
Swipe through the animation to see the full flow. Which route type do you use most in production? 👇


<img width="1280" height="634" alt="image" src="https://github.com/user-attachments/assets/8f4d68c1-ab00-4d60-92f5-067832dcb685" />
