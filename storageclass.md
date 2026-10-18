OpenShift Learning Series | Persistent Storage

A Pod gets deleted. Does the application data disappear with it?

This question helped me understand why persistent storage is such an important part of OpenShift.

The basic storage flow is:

Pod → PVC → StorageClass → CSI Driver → PV → Storage Backend

The way I remember it:

📄 PVC → “I need storage”
🏷️ StorageClass → “What kind of storage?”
⚙️ CSI Driver → “I’ll provision/connect it”
🗄️ PV → Storage resource available to the cluster
💾 Backend → Where the data actually lives

And the important part:

Pod-1 writes data → Pod-1 dies 💥 → Pod-2 uses the same PVC → Data remains available ✅

🚨 Production Scenario

A new application is deployed, but the Pod remains:

Pending

Instead of focusing only on the Pod, follow the storage chain:

Pod → PVC → StorageClass → CSI → PV → Backend

Check where the provisioning or binding process is failing.

📌 Key takeaway:

Pods are ephemeral. Application data often shouldn’t be.

Understanding PV, PVC, StorageClass and CSI makes storage troubleshooting much easier than simply memorizing commands.

💬 Interview question: A PVC remains in Pending state. What would you check first?

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/acdb11af-6984-4b6e-8473-84179ea44eb9" />



OpenShift Storage Troubleshooting | PVC Stuck in Pending

A developer deploys an application requiring a 20Gi PVC.

But instead of starting:

Pod → Pending ❌
PVC → Pending ❌
PV → Not Created ❌

So where should an OpenShift Administrator start?

One thing I’m learning with troubleshooting is:

Don’t guess the problem. Follow the chain and find the first broken link.

For dynamic storage provisioning, I follow:

PVC → StorageClass → CSI Driver → PV → Storage Backend

And one of the most useful commands is:

oc describe pvc <pvc-name>

The Events often tell you whether the issue is related to:

→ StorageClass
→ CSI/provisioning
→ Access mode or capacity
→ Topology/scheduling
→ Permissions
→ Backend storage

🎯 Interview Scenario:

PVC is Pending even though a StorageClass exists. What would you check next?

My first step would be to inspect the PVC events rather than immediately changing the configuration.

📌 Key takeaway:

Good troubleshooting isn’t about running more commands.
It’s about finding the first broken link in the chain.


<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/b699e9f0-3d03-4abd-a489-576b655e85da" />





OpenShift Learning Series | Persistent Volume Reclaim Policies

A developer accidentally deletes a PVC.

Does that mean the application data is gone?

Not necessarily.

It all depends on the PersistentVolume Reclaim Policy.

🟢 Retain → Data is preserved. The administrator decides what happens next.

🔴 Delete → The PV and underlying storage are removed (if supported), and the data is deleted.

⚪ Recycle → Deprecated. Not used in modern OpenShift.

💡 Interview Question

Q: Does deleting a PVC always delete the data?

A: No. It depends on the PV’s reclaim policy.

📌 Key takeaway:

Deleting a PVC doesn’t always mean deleting your data.

Always check the reclaim policy before making storage changes in production.

💬 Which reclaim policy does your organization use for production databases?


<img width="1149" height="1369" alt="image" src="https://github.com/user-attachments/assets/7faaaf7b-07fc-47af-af5b-0648c110795c" />


OpenShift Learning Series | Storage Access Modes

A developer scales an application from 1 → 3 replicas.

Pod-1 → Running ✅
Pod-2 → Pending ❌
Pod-3 → Pending ❌

Could storage be the reason? 🤔

One thing worth checking is the PVC Access Mode.

I remember them like parking rules:

🟢 RWO → Read-write from a single Node
🔵 RWX → Read-write from multiple Nodes
🟠 ROX → Read-only from multiple Nodes
🟣 RWOP → Read-write by a single Pod

🎯 Interview trap:

Does RWO mean only one Pod can use the volume?

No. RWO means the volume can be mounted read-write by a single node. Multiple Pods on that node may potentially access it.

If you specifically need read-write access restricted to one Pod, that’s RWOP.

📌 Key takeaway:

Choose the access mode based on the application architecture AND what the storage backend supports.


<img width="1197" height="1314" alt="image" src="https://github.com/user-attachments/assets/bd49cd17-729f-488d-bbda-6ba3d3a2e58f" />

OpenShift Learning Series | Dynamic vs Static Provisioning

A developer creates a PersistentVolumeClaim (PVC) requesting 20Gi of storage.

A few seconds later…

✅ A PersistentVolume (PV) appears automatically.

But here’s the interview question:

Who actually created the PV? 🤔

The answer is:

❌ Not the developer.
❌ Not the administrator manually.

✅ The CSI Driver, guided by the StorageClass, dynamically provisions the storage and creates the PV.

Quick comparison:

🔹 Dynamic Provisioning

* Developer creates only the PVC.
* StorageClass + CSI Driver automatically create the PV.

🔹 Static Provisioning

* Administrator creates the PV first.
* The PVC binds to the existing PV.

💡 Key takeaway:

PVC is the request. PV is the storage. StorageClass + CSI Driver make the automation happen.

🎯 Interview Question:

When would you choose Static Provisioning instead of Dynamic Provisioning?

💬 I’d love to hear how your organization manages persistent storage in OpenShift.






<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/83696b36-c87f-4b9c-bdf7-207ffe4e4b91" />



OpenShift Learning Series | 15 Production Scenario-Based Storage Interview Questions

Over the last few days, I’ve been exploring OpenShift Storage—from PVCs and StorageClasses to CSI Drivers, Access Modes, Reclaim Policies, and Dynamic Provisioning.

To wrap up the storage section, I created 15 production-oriented interview questions that helped me connect the concepts instead of just memorizing them.

Some of the scenarios include:

🔹 Why is a PVC stuck in Pending?
🔹 Why wasn’t a PV created automatically?
🔹 Does deleting a PVC always delete the data?
🔹 Why do Pods fail after scaling from 1 to 3 replicas?
🔹 RWO vs RWX vs RWOP—when should each be used?
🔹 Static vs Dynamic Provisioning
🔹 Volume mount failures and expansion issues

The biggest lesson I learned is:

Don’t troubleshoot the Pod first. Follow the storage chain.

PVC → StorageClass → CSI Driver → PV → Storage Backend

Once you identify the first broken link, troubleshooting becomes much more structured and efficient.

💬 Interview Question:

A Pod is stuck in Pending because of an unbound PVC. What’s the first command you would run, and why?

📌 I hope these notes help anyone preparing for OpenShift or Kubernetes storage interviews.


<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/cf7a0e32-10a4-4c8b-89d8-7f6492e0e876" />




