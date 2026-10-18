OpenShift Learning Series | Inside an OpenShift Pod

🚨 Your Pod is running… until it isn’t.

As I continue exploring OpenShift, I wanted to go beyond the basic question:

“What is a Pod?”

The more interesting question is:

👉 What actually happens when something goes wrong inside a Pod?

In production, we often see states and errors like:

⏳ Pending
📦 ContainerCreating
❌ ImagePullBackOff
🔄 CrashLoopBackOff
💾 OOMKilled
⚠️ Readiness probe failures

But each one tells a different story.

For example:

🔹 Pending → The Pod may not have been scheduled yet. Check resources, taints/tolerations, affinity, or unbound PVCs.

🔹 ContainerCreating → The Pod is scheduled, but Kubelet is still setting up the container, storage, or networking.

🔹 ImagePullBackOff → Check the image name, registry access, and image pull secrets.

🔹 CrashLoopBackOff → The container starts but repeatedly crashes. Start with application logs, events, configuration, and probes.

🔹 OOMKilled → The container exceeded its memory limit.

🔹 Readiness Probe Failed → The application may be running, but the Pod isn’t ready to receive traffic.

This made me realize something important:

Knowing that a Pod has failed isn’t enough. Understanding its state tells you where to start troubleshooting.

🛠️ My basic troubleshooting workflow:

1️⃣ Check Pod status
oc get pods

2️⃣ Describe the Pod
oc describe pod <pod-name>

3️⃣ Check Events

4️⃣ Check container logs
oc logs <pod-name> -c <container>

5️⃣ If needed, investigate the Worker Node

Another important concept: Pods are ephemeral.

If a Pod is recreated, it normally gets a new Pod IP. But if the application uses persistent storage through a PVC, the persistent data can survive Pod recreation.

⸻

🎯 Interview Scenario

Q: A Pod is in CrashLoopBackOff. How would you troubleshoot it?

My approach would be:

👉 Check Pod events
👉 Review current and previous container logs
👉 Check exit codes and termination reasons
👉 Validate ConfigMaps and Secrets
👉 Check liveness/startup probes
👉 Review CPU and memory limits
👉 Verify application dependencies

The key is not to guess—follow the evidence from Pod → Events → Logs → Node.

⸻

💡 Technical Takeaway

A good OpenShift Administrator doesn’t just know what a Pod is.

They understand how a Pod behaves throughout its lifecycle—and where to look when that lifecycle breaks.

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/16dd1a5b-8a97-4798-9422-321defe73f6e" />
