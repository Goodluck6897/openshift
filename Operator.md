OpenShift Learning Series | Operators – The AutoPilot of OpenShift

Imagine flying an aircraft.

Without AutoPilot, the pilot constantly adjusts the engine, altitude, speed, and direction.

With AutoPilot, you set the destination, and the system continuously keeps the aircraft on course.

That’s exactly how I think about OpenShift Operators.

Instead of manually managing an application’s lifecycle, an Operator continuously watches the desired state and reconciles the actual state whenever something drifts.

🎯 Deployment vs Operator

🔹 Deployment → Manages Pods and ReplicaSets.

🔹 Operator → Manages the entire application lifecycle using Custom Resources and reconciliation logic.

📌 Key takeaway:

Define the desired state once. The Operator continuously works to keep the application on course.

💬 Interview Question:

If a Deployment can create Pods, why do we need an Operator?

I’d love to hear how you explain this concept during interviews.

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/519c1c92-f462-4f05-9e59-511e6bfbee32" />

OpenShift Operators are powerful… but who manages the Operators?

👉 That’s where Operator Lifecycle Manager (OLM) comes in.

Think of it like a Control Tower 🗼 for your Operators:

* 📦 Installs Operators from OperatorHub
* 🔄 Manages upgrades automatically
* 🔍 Handles dependencies & versions
* ❤️ Keeps everything healthy

👉 You don’t manage Operators manually anymore —
OLM does it for you.

⸻

💡 Simple Flow:
Admin → Subscription → InstallPlan → Operator → Running App

⸻

🎯 Interview Tip:

OLM automates the entire lifecycle of Operators — install, upgrade, and health management.

⸻

📌 Key Takeaway:
👉 Operator = Autopilot
👉 OLM = Control Tower for those Autopilots



<img width="1280" height="853" alt="image" src="https://github.com/user-attachments/assets/c3ed0d54-2a5f-49a1-ac60-f265e2db9eaa" />

CRD & Custom Resource — The Heart of Operators

Most people understand Operators…
But the real magic happens here 👇

👉 CRD (Custom Resource Definition)
= Defines a new resource type (like a blueprint)

👉 CR (Custom Resource)
= Actual instance created by the user

⸻

💡 Simple Flow:
CRD → CR → Operator → Pods / PVC / Services

⸻

🤖 What does Operator do?

* 👀 Watches the CR
* ⚖️ Compares desired vs actual state
* 🔧 Creates / fixes resources automatically

⸻

🎯 Interview Tip:

Operator doesn’t manage apps directly —
it manages Custom Resources (CRs).

⸻

📌 Key Takeaway:
👉 CRD = Blueprint
👉 CR = Real Object
👉 Operator = Automation Engine

<img width="1280" height="853" alt="image" src="https://github.com/user-attachments/assets/819058ad-ac16-42df-9c62-291dc6aef106" />

OpenShift Operator Reconciliation Loop — How does an Operator actually work?

An Operator doesn’t just install an application and stop.

It continuously checks whether the actual state matches the desired state.

Think of it like a doctor monitoring a patient:

📝 CR → Treatment Plan / Desired State
👀 Watch → Monitor changes
⚖️ Compare → Desired vs Actual State
🔧 Act → Create / Update / Delete resources
✅ Reconcile → Keep the application healthy

For example, if the desired state is 3 replicas but only 1 Pod is running, the Operator detects the drift and takes corrective action.

🎯 Interview takeaway:

Operators continuously reconcile the actual state of the cluster with the desired state defined by the Custom Resource.

This is the core idea behind self-healing and automation in Operator-based applications.

<img width="1280" height="853" alt="image" src="https://github.com/user-attachments/assets/59ad994c-da1c-4cb0-984c-96f57892e874" />

OpenShift Operator Troubleshooting — Follow the Flow, Not Guesswork

When an Operator isn’t working, don’t jump straight to the Pod.

Follow the lifecycle step by step:

Subscription → InstallPlan → CSV → Operator Pod → CR → Managed Resources

At each stage, ask:

🔹 Is it present and healthy?
🔹 What do the Events say?
🔹 What do the logs say?
🔹 Is there a dependency, RBAC, configuration, or resource issue?

🎯 Production mindset:

Find the first broken link in the chain — that’s usually where the root cause starts.

💡 One issue in OLM can prevent the entire Operator lifecycle from progressing.

Interview question:
A Subscription exists, but the Operator is not installing. What would you check first?

<img width="1280" height="853" alt="image" src="https://github.com/user-attachments/assets/b5914921-0961-4cc7-90f9-eee414f12b19" />
CSV — The Operator’s Identity Card in OpenShift

Think of CSV as the Operator’s identity card.

It tells OLM:

🔹 Which Operator version to install
🔹 What CRDs it provides
🔹 What permissions it needs
🔹 What dependencies/capabilities it has
🔹 Whether the installation succeeded or failed

The installation chain:

Subscription → InstallPlan → CSV → Operator Pod → Managed Resources

🎯 Interview Tip:

If an Operator isn’t installing, check the CSV status and Events. They often tell you exactly where the installation is stuck.

Remember:

Subscription = What I want
InstallPlan = How to install it
CSV = What Operator is being installed

<img width="1280" height="853" alt="image" src="https://github.com/user-attachments/assets/19d23b1a-2db0-4233-b610-7e8a46c89950" />


Subscription vs InstallPlan vs CSV — A simple way to understand OLM

These three OpenShift objects can be confusing when you’re starting with Operators.

Here’s the easiest way I remember them:

🎫 Subscription → What do I want?
📋 InstallPlan → How will it be installed?
🪪 CSV → What exactly is being installed?

The flow:

Subscription → InstallPlan → CSV → Operator Pod → Managed Resources

If troubleshooting an Operator installation, follow this chain from the top and find the first point where things stop progressing.

🎯 Interview takeaway:

Subscription expresses the intent.
InstallPlan defines the installation plan.
CSV represents the specific Operator version being installed.

One broken link can stop the entire Operator installation.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/84a63637-e5d3-4142-85b3-9f958558d91f" />

OpenShift Operator Upgrades — What happens behind the scenes?

An Operator upgrade isn’t just “install the new version.”

The upgrade typically flows through:

New Version → CatalogSource → Subscription → InstallPlan → Approval → New CSV → New Operator

🔄 Automatic approval: OLM can proceed automatically.

🛡️ Manual approval: Admin reviews the InstallPlan before the upgrade — a safer approach for production.

🎯 Interview Tip

Before approving an Operator upgrade, check:

* Upgrade path
* InstallPlan
* CSV Events
* Dependencies & CRDs
* Operator logs

💡 And an important point:

Existing Custom Resources are not simply deleted during an Operator upgrade. The new Operator version continues managing them, provided the upgrade is compatible.

Don’t blindly approve an upgrade. Understand the upgrade path first.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/b0e12fb4-710b-47f4-acab-6811a1ec7153" />


OperatorHub & CatalogSource — Where do OpenShift Operators come from?

When you install an Operator from OpenShift, have you ever wondered where that Operator actually comes from?

Think of it like a marketplace:

🏪 OperatorHub → Marketplace where you discover Operators
📚 CatalogSource → Catalog containing Operator metadata
📦 Package/Channel → Available Operator versions
🎫 Subscription → “I want this Operator”
📋 InstallPlan → “Here’s how to install it”
🪪 CSV → “This exact Operator version”
🤖 Operator → Running in the cluster

🔄 The flow

OperatorHub → CatalogSource → Package/Channel → Subscription → InstallPlan → CSV → Operator

🎯 Production tip:

If an Operator suddenly disappears from OperatorHub, check the CatalogSource first — its status, events, pod logs, image accessibility, network/proxy and package availability.

OperatorHub is the marketplace. CatalogSource is the catalog. OLM handles the installation.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/f8477973-bb85-4381-8be7-8bf9ff652583" />




OpenShift Operator Lifecycle & CSV States

Understanding CSV states is extremely useful when troubleshooting Operators in production.

A CSV can move through states such as:

🕐 Pending → Installation hasn’t started
⚙️ Installing → OLM is installing the Operator
✅ Succeeded → Operator installed successfully
❌ Failed → Something went wrong
🔄 Replacing/Upgrading → New CSV is replacing the old version

🔥 Production troubleshooting

If a CSV is stuck, don’t guess.

Check:

CSV → Events → InstallPlan → Operator Pod → CRDs/RBAC → Dependencies

Useful commands:

oc get csv
oc describe csv <name>
oc get installplan
oc get events

🎯 Interview takeaway:

CSV status tells you where the Operator lifecycle is, while Events often tell you why it’s stuck.

Understanding the state + events can save a lot of troubleshooting time.




<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/56aa2a95-f2c7-46fc-8eec-87ebe9c90e71" />

OpenShift Operator Pod Troubleshooting — Production Scenarios

When an Operator isn’t working, don’t immediately blame OLM.

Follow the stack:

CSV → Deployment → Operator Pod → Logs/Events → RBAC → Dependencies

Common issues include:

❌ CrashLoopBackOff
⏳ Pod stuck in Pending
📦 ImagePullBackOff
🔐 Missing RBAC permissions
⚠️ CSV stuck in Installing
🔄 Operator repeatedly restarting

🎯 My troubleshooting approach

Observe → Analyze → Fix → Verify

Start with:

oc describe csv <name>
oc describe pod <pod>
oc logs <pod> --previous
oc get events

💡 Interview tip:

If the CSV is Succeeded but the Operator Pod is failing, the problem is likely at the runtime level, not the OLM installation level.

Find the first broken layer — don’t guess.

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/0ed1bb34-ffb9-4e33-b00a-46c12f223eee" />

OpenShift Learning Series: Operators vs Deployments 🤔

“Why do I need an Operator if I can simply use a Deployment?”

This is a common OpenShift interview question—and an important production decision.

🏗️ Deployment

* Best for simple, stateless applications
* Manages Pods, replicas and rolling updates
* Self-heals failed Pods
* Application-specific operations are handled by you

🤖 Operator

* Understands the application’s domain and lifecycle
* Uses Custom Resources (CRs)
* Can automate configuration, upgrades, scaling, backup/recovery and other application-specific tasks
* Ideal for complex or stateful workloads

🔥 Production scenario

Running PostgreSQL in OpenShift?

A Deployment can run the Pods, but you would need to handle many database-specific operations yourself.

An Operator can automate much of that application lifecycle.

👉 Simple stateless app → Deployment
👉 Complex/stateful application → Operator

🎯 Interview takeaway

Deployments run applications.
Operators manage applications.

The key is not “Operator is better than Deployment.”

It’s choosing the right tool based on the application’s lifecycle and operational complexity.

<img width="1225" height="1284" alt="image" src="https://github.com/user-attachments/assets/12d39826-55a7-4338-b718-1a8a492d17cf" />


Operator Scope — Namespace vs Cluster 🎯

One Operator doesn’t always mean cluster-wide access.

A common OpenShift interview question is:

Can an Operator manage only one namespace, or can it manage the entire cluster?

The answer depends on the OperatorGroup configuration.

🔹 Namespace-scoped Operator

* Watches/manages specific namespace(s)
* Provides better isolation
* Useful for team or multi-tenant environments
* Limits the Operator’s operational reach

🌐 Cluster-scoped Operator

* Can watch/manage resources across the cluster
* Useful for cluster-wide capabilities like monitoring, logging, or infrastructure services
* Requires broader permissions and careful governance

🔑 Remember

OperatorGroup → defines the Operator’s target namespaces → determines its scope.

🔥 Interview tip

If asked:

“How do you install an Operator for specific namespaces?”

Say:

“I use an OperatorGroup with spec.targetNamespaces configured for the required namespaces.”

For cluster-wide scope, an OperatorGroup with an empty targetNamespaces can represent all namespaces, depending on the Operator/OLM setup.

Same Operator. Different reach. Choose the scope based on security and operational requirements.

<img width="1225" height="1284" alt="image" src="https://github.com/user-attachments/assets/a8b10eb3-b3b0-4d25-9914-b3a717d9aea8" />



OpenShift Learning Series 🔗 | Operator Dependencies

Your Operator may not be the only Operator working behind the scenes.

In OpenShift, some Operators depend on other Operators or APIs to provide the functionality they need.

Think about it like this:

🏗️ Dependency → Foundation
🤖 Operator → Builder
🚀 Application → Final Product

If the foundation is missing, the build cannot move forward.

That’s why an Operator stuck in Pending should immediately make you ask:

“Is a dependency missing?”

A quick troubleshooting flow:

CSV → InstallPlan → Dependencies → CRDs → Events

🎯 Interview tip:
Don’t troubleshoot an Operator in isolation. Always check its dependencies.

<img width="1312" height="1199" alt="image" src="https://github.com/user-attachments/assets/2b9850ea-edef-4d0c-8610-66054ddbbe9f" />



Troubleshooting an Operator Installation

Operator stuck in Pending? Don’t troubleshoot randomly. Follow the flow.

A simple troubleshooting approach:

Subscription → InstallPlan → CSV → Operator Pod → CR → Managed Resources

🔹 Subscription — Is the channel and CatalogSource correct?
🔹 InstallPlan — Any failed steps or missing dependencies?
🔹 CSV — Is it Succeeded or stuck in Pending?
🔹 Operator Pod — Is it Running? Check logs and events.
🔹 CR — Is the configuration valid?
🔹 Managed Resources — Are the expected resources created and healthy?

Useful commands:

oc get subscription -n 
oc get installplan -n 
oc get csv -n 
oc get pods -n 
oc get events -n 
oc logs  -n 

🎯 Interview tip:

“I troubleshoot an Operator layer by layer, starting with the Subscription and following the flow until the managed resources are healthy.”

Know the flow → Find the layer → Fix the issue.

<img width="1225" height="1284" alt="image" src="https://github.com/user-attachments/assets/4cc7e186-ae7d-42b3-985f-235d04a78c3e" />

Operator Installed, But Application Not Working?

Operator installed successfully ≠ Application is healthy.

This is a common production troubleshooting scenario.

Follow the flow:

Operator → CR → Reconciliation → Managed Resources → Application

🔹 Check Operator
Is the Pod running? Is the CSV in Succeeded state?

🔹 Check the CR
Is the Custom Resource created? Check its status, conditions and events.

🔹 Check Reconciliation
Is the Operator successfully processing the CR? Check Operator logs.

🔹 Check Managed Resources
Are Pods, PVCs, Services and Routes being created correctly?

🔹 Check Application
Are the Pods healthy? Is the Service/Route accessible? Check application logs.

🎯 Interview tip:

“I would troubleshoot layer by layer—starting with the CR status and Operator logs, then checking managed resources and finally the application.”

Operator healthy → CR healthy → Resources healthy → Application healthy.

<img width="1225" height="1284" alt="image" src="https://github.com/user-attachments/assets/811d7b0c-bfec-4748-84fc-ff3384c4f881" />

Operator Not Visible in OperatorHub?

Ever searched for an Operator in OperatorHub and couldn’t find it?

Don’t assume the Operator is unavailable. Troubleshoot the catalog path first.

Follow the flow:

CatalogSource → Catalog Pod → Package/Channel → OperatorHub → Subscription

🔹 CatalogSource — Is it READY?
🔹 Catalog Pod — Is it Running without errors?
🔹 Package & Channel — Does the Operator and required channel exist?
🔹 OperatorHub — Are you checking the correct scope/project?
🔹 Subscription — Is the correct package, channel and source selected?

Common causes include:

• CatalogSource not ready
• Catalog pod crashing
• Wrong/missing package or channel
• Registry/network issues
• Incorrect OperatorHub scope
• Disconnected environment problems

🎯 Interview tip:

“I would troubleshoot the catalog layer first—CatalogSource, catalog pod, package/channel, and scope—before creating the Subscription.”

If you can’t see the Operator, check the catalog path first.

<img width="1225" height="1284" alt="image" src="https://github.com/user-attachments/assets/c2343a09-aee1-4eac-b412-826f3064082d" />

Operators in a Disconnected Cluster

No internet access ≠ No Operators.

In enterprise environments, OpenShift clusters are often air-gapped. So how do we install Operators without direct internet access?

The basic flow is:

Red Hat Registry → Mirror → Private Registry → CatalogSource → OperatorHub → Subscription → Operator

🔹 Mirror the required Operator catalogs and images using oc-mirror
🔹 Store them in a private registry
🔹 Configure a CatalogSource pointing to the mirrored catalog
🔹 Verify the Operator appears in OperatorHub
🔹 Create a Subscription and install the Operator

⚠️ Common issues:

* CatalogSource not Ready
* Missing images/dependencies
* Registry authentication or certificate issues
* Wrong package/channel
* Image pull failures

🎯 Interview takeaway:

“In a disconnected environment, I would mirror the required catalogs and images, store them in a private registry, configure a CatalogSource, and then install the Operator through OLM.”

No internet. No problem. The catalog just needs a different path.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/a7fed56f-975e-4771-bdc4-fb5c290823d8" />



Operator Upgrade Failure

Operator upgrade stuck in Pending or Failed? Don’t panic—follow the OLM flow.

The upgrade path is:

Subscription → InstallPlan → New CSV → Operator Pod → CR → Application

When something goes wrong, check each layer:

🔹 Subscription — Correct channel and upgrade path?
🔹 InstallPlan — Created and approved? Any dependency errors?
🔹 CSV — Succeeded, Replacing, or Failed?
🔹 Operator Pod — Running? Check logs and image-pull errors.
🔹 Events — Any dependency, permission or configuration errors?
🔹 CR — Still healthy after the upgrade?

Common causes:
• Missing dependencies
• Image pull/registry issues
• Incompatible CRD/API changes
• Wrong channel or version
• Operator Pod failures

🎯 Interview tip:

“I would troubleshoot the Subscription, InstallPlan and CSV first, then check Operator Pods, events and CR health before considering a rollback.”

Never troubleshoot an upgrade blindly. Follow the OLM chain.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7200ecc5-bd7c-4296-9b98-b15473c614b0" />




















