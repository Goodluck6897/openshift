<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/02972326-392a-4bb3-8b94-b80b71ee3d21" />
OpenShift Learning Series | Understanding etcd Quorum

While learning more about etcd, I came across an important concept that every OpenShift Administrator should understand:

👉 Quorum

We often hear that a highly available OpenShift cluster commonly has 3 control plane nodes.

But why 3? 🤔

One important reason is etcd quorum.

Imagine a cluster with 3 etcd members:

🟢 etcd-1
🟢 etcd-2
🟢 etcd-3

All healthy → 3/3 available → Quorum ✅

Now, one control plane node goes down:

🟢 etcd-1
🟢 etcd-2
🔴 etcd-3

We still have 2 out of 3 etcd members.

➡️ Majority available
➡️ Quorum maintained ✅
➡️ etcd can continue making progress

But what if another etcd member goes down?

🟢 etcd-1
🔴 etcd-2
🔴 etcd-3

Now only 1 out of 3 members is available.

❌ No majority
❌ Quorum lost
❌ etcd cannot safely commit new state changes

Existing workloads on healthy worker nodes may continue running, but normal control-plane operations and cluster changes are severely impacted.

💡 The simple rule:

3 etcd members → 2 required for quorum → 1 failure tolerated

5 etcd members → 3 required for quorum → 2 failures tolerated

This helped me better understand that having 3 control plane nodes isn’t simply about having three servers.

It’s about building a highly available control plane where critical services—including etcd—can tolerate failures.

⸻

🎯 Interview Scenario

Q: You have a 3-control-plane-node OpenShift cluster. One control plane node goes down. Will the entire cluster stop working?

A: Not necessarily. If the remaining 2 etcd members are healthy and reachable, they still form a majority, so etcd quorum is maintained. The cluster can continue control-plane operations, assuming the remaining required components are healthy.

⸻

📌 My key takeaway:

Quorum is what allows etcd to stay consistent while tolerating member failures.

Understanding this concept is important not only for interviews but also for troubleshooting real OpenShift control-plane incidents.

💬 Question for you:

If an etcd cluster has 5 members, how many members can fail while still maintaining quorum? 🤔

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/d657aafb-5057-41d7-a845-499163c08af5" />
