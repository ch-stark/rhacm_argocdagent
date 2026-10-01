
Managed mode blog · MD
# Scaling GitOps with the Argo CD Agent: Managed and Hybrid Mode in Red Hat Advanced Cluster Management
 
*How a pull-based, agent-driven architecture lets you run GitOps across thousands of clusters without giving up central control, and how Hybrid Mode lets the hub deploy to itself too.*
 
---
 
## TL;DR
 
- With **Red Hat Advanced Cluster Management (ACM)** and **OpenShift GitOps**, the **Argo CD Agent** is generally available.
- The agent moves reconciliation to the workload clusters while keeping a single control point and a single UI on the hub.
- In **Managed Mode**, you author `Application` and `ApplicationSet` resources on the hub (the *Principal*), and agents on the managed clusters (the *spokes*) pull and apply them.
- In **Hybrid Mode**, you run Managed Mode for your remote clusters *and* keep the hub's own Argo CD controller enabled, so `Application` and `ApplicationSet` resources can also deploy to the hub cluster itself.
- The agent works with **OpenShift and non-OpenShift Kubernetes clusters** and is **fully supported** on both, including **disconnected** environments (using `ManagedClusterImageRegistry` to point clusters at your mirror registry).
- Spokes only make **outbound** connections, all traffic is protected by **mTLS**, and the certificate lifecycle is automated.
- Setup is driven by a `Placement` and a `GitOpsCluster` resource. **The `GitOpsCluster` must never select the hub cluster (`local-cluster`).**
---
 
## 1. Why the push model hits a ceiling
 
For years the standard multi-cluster GitOps pattern was the **push model**: one central Argo CD instance reaches out to every remote cluster and manages it. That works well for dozens of clusters. At thousands, the cracks show:
 
| Challenge | What happens in the push model |
|---|---|
| **Connections** | The hub keeps persistent connections to every remote cluster. |
| **Resource load** | The hub watches and caches resources across the whole fleet, so CPU and memory grow with fleet size. |
| **Credentials** | The hub stores access credentials for every managed cluster. |
| **Blast radius** | Compromising the hub can mean access to every cluster. |
| **Networking** | The hub must be able to reach every spoke, which is painful with firewalls, NAT, or edge links. |
 
The Argo CD Agent takes a different approach. Reconciliation happens **on the workload cluster**, and a bi-directional streaming connection back to the hub preserves the central view. You get the scalability and security of a pull architecture *and* the single pane of glass that enterprises expect.
 
---
 
## 2. Choosing a mode: Managed, Autonomous or Hybrid
 
The deciding questions are: **where does the source of truth for your application definitions live, and where do your apps need to run?**
 
| Dimension | Managed Mode | Autonomous Mode | Hybrid Mode |
|---|---|---|---|
| **Where apps are authored** | Hub cluster (Principal) | Workload cluster (spoke) | Hub cluster (Principal) |
| **Who controls the Application spec** | The hub, fully | The spoke; the hub copy is read-only and any change is reverted | The hub, fully |
| **Where apps can deploy** | Managed clusters | The spoke itself | Managed clusters **and** the hub cluster |
| **Hub Argo CD controller** | Not used for fleet workloads | Not used for fleet workloads | **Enabled**, for hub-local apps |
| **Routing** | Destination-based mapping from hub to agent | Local-only reconciliation | Destination-based mapping to agents; hub-local apps handled by the hub controller |
| **Typical use case** | Centralized GitOps control and policy enforcement | Decentralized or air-gapped environments needing local autonomy | Central control for the fleet *plus* GitOps for hub-hosted components |
 
> **Architect's note:** Managed and Hybrid Mode are a strong fit for compliance-driven environments (for example SOC 2 or ISO 27001) where the hub should be the single, guarded authority for what gets deployed. In Autonomous Mode the opposite holds: if someone edits an `Application` through the hub UI or CLI, the change is automatically reverted to match the agent's local state, so the spoke stays the true master.
 
### How this differs from the "basic" pull model
 
ACM also offers a lighter-weight pull approach based purely on `ManifestWork`. The Argo CD Agent is the **advanced** option: it supports full resource trees in the Argo CD UI and live-state comparison, which a ManifestWork-only approach does not provide.
 
---
 
## 3. How Managed Mode works
 
Managed Mode combines centralized authoring with distributed execution. Four building blocks make it work:
 
### 3.1 Centralized authoring
`Application` and `ApplicationSet` resources are created on the **Principal** (the hub). The hub is the single entry point for configuration and the single place where you review and audit changes.
 
### 3.2 Agent-initiated gRPC streaming
The agent on each spoke opens a **bi-directional gRPC connection** to the Principal. Because the connection is outbound-only from the spoke:
 
- Workload clusters need **no inbound ports** or ingress rules.
- The hub does **not need credentials to reach the managed cluster's API server**.
- The hub does not need to know the network topology of the spokes.
### 3.3 Local reconciliation
The Argo CD application controller **on the managed cluster** performs the actual deployment. This also makes the design resilient: if the connection to the hub drops, the spoke keeps running and continues reconciling against its last known state. This matters for edge and unreliable networks.
 
### 3.4 Live feedback through the Redis proxy
To keep the central UI responsive without caching every resource in the fleet, the Principal uses a **Redis proxy**:
 
- Non-agent keys, such as cluster metadata, are served from the hub.
- Agent-specific keys, such as `app|resources-tree|*` and `app|managed-resources|*`, are routed directly to the relevant agent.
The result is a hub UI that shows live state and resource trees while the hub carries only a fraction of the data it would hold in a push model.
 
```mermaid
flowchart LR
    subgraph Hub["Hub cluster (Principal)"]
        UI["Argo CD UI / API"]
        APPS["Applications &<br/>ApplicationSets"]
        RP["Redis proxy"]
    end
 
    subgraph Spoke["Managed cluster (Spoke)"]
        AG["Argo CD Agent"]
        CTRL["Application<br/>controller"]
        WL["Workloads"]
    end
 
    APPS --> UI
    AG -- "outbound gRPC + mTLS" --> Hub
    AG --> CTRL --> WL
    UI <--> RP <--> AG
```
 
---
 
## 4. Hybrid Mode: one control plane for the fleet and the hub
 
Managed Mode pushes work out to remote clusters. But real hubs are rarely empty. They host their own platform components, such as operators, policies, configuration and shared services, and you probably want to manage those with GitOps as well.
 
**Hybrid Mode** covers both cases on the same hub:
 
- **Remote clusters** run the agent in `managed` mode and are targeted by name through destination-based mapping.
- **The hub cluster** keeps its regular Argo CD application controller enabled, so `Application` and `ApplicationSet` resources can also target the hub itself.
In other words, `ApplicationSet`s can also be deployed on the hub cluster, not only on the fleet.
 
```mermaid
flowchart LR
    subgraph Hub["Hub cluster"]
        APPS["Applications &<br/>ApplicationSets"]
        HC["Hub Argo CD<br/>controller"]
        PR["Principal"]
        HW["Hub-local workloads"]
    end
 
    subgraph S1["Managed cluster A"]
        A1["Agent"] --> W1["Workloads"]
    end
 
    subgraph S2["Managed cluster B"]
        A2["Agent"] --> W2["Workloads"]
    end
 
    APPS -- "destination: hub" --> HC --> HW
    APPS -- "destination.name: cluster-a/b" --> PR
    A1 -- "outbound gRPC + mTLS" --> PR
    A2 -- "outbound gRPC + mTLS" --> PR
```
 
### The one rule: never select the hub in `GitOpsCluster`
 
The `GitOpsCluster` (through its `Placement`) must **only** select remote managed clusters. It must **not** point to the hub cluster (`local-cluster`).
 
The agent is installed only on the clusters the `GitOpsCluster` selects. The hub already has its own Argo CD controller for hub-local apps, so installing an agent there would create a second, competing path to the same cluster. Keep the two paths separate:
 
| Target | Handled by | Selected via |
|---|---|---|
| Remote managed clusters | Agent (managed mode) | `Placement` referenced by `GitOpsCluster` |
| Hub cluster | Hub Argo CD controller | Regular `Application` destination for the hub; **not** in the `GitOpsCluster` placement |
 
### Prerequisites
 
- OpenShift GitOps installed on the hub, with an `ArgoCD` instance (for example `openshift-gitops`).
- Your remote clusters registered with ACM and in the **Available** state.
- A `ManagedClusterSetBinding` in the GitOps namespace for each `ManagedClusterSet` that contains your target clusters, otherwise the `Placement` cannot resolve them.
- Each target cluster labeled so your `Placement` can select it (the example below uses `argocd-agent.rhacm.io/setup=hybrid`).
### What to configure
 
**1. A Placement that excludes the hub.** Select the remote clusters by label and explicitly exclude `local-cluster`:
 
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: argocd-agent-hybrid
  namespace: openshift-gitops
spec:
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            argocd-agent.rhacm.io/setup: hybrid
          matchExpressions:
            - key: local-cluster
              operator: NotIn
              values:
                - "true"
```
 
**2. A GitOpsCluster in managed mode that references that Placement:**
 
```yaml
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-agent-hybrid
  namespace: openshift-gitops
spec:
  argoServer:
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: argocd-agent-hybrid
    namespace: openshift-gitops
  gitopsAddon:
    enabled: true
    overrideExistingConfigs: true
  argoCDAgent:
    enabled: true
    mode: managed
    propagateHubCA: true
```
 
**3. An ArgoCD instance on the hub with both the controller and the principal enabled:**
 
```yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  controller:
    enabled: true            # hub-local apps are reconciled by the hub controller
  sourceNamespaces:
    - '*'
  argoCDAgent:
    principal:
      enabled: true
      destinationBasedMapping: true
      auth: 'mtls:CN=system:open-cluster-management:cluster:([^:]+):addon:gitops-addon:agent:gitops-addon-agent'
      namespace:
        allowedNamespaces:
          - '*'
      server:
        route:
          enabled: true
```
 
Three details deserve attention:
 
- **`controller.enabled: true`** is what makes Hybrid Mode "hybrid". Without it, the hub has no controller to deploy hub-local applications.
- **`destinationBasedMapping: true`** means your `ApplicationSet`s address remote clusters through `destination.name`, which routes each application to the right agent.
- **The `auth` expression** maps the client certificate's common name to the agent identity, so each agent is recognized by the cluster it belongs to.
**4. Operator settings for the principal.** On the OpenShift GitOps operator subscription, the reference setup sets these environment variables:
 
| Variable | Value |
|---|---|
| `ARGOCD_CLUSTER_CONFIG_NAMESPACES` | the GitOps namespace (for example `openshift-gitops`) |
| `ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE` | `false` (certificates come from the ACM-managed CA) |
| `ARGOCD_PRINCIPAL_REDIS_SERVER_ADDRESS` | `<argocd-name>-redis:6379` |
 
**5. Permissions and project settings.** Because the hub controller deploys to the hub, its service account needs RBAC on the hub. With destination-based mapping, the `AppProject` your apps use must allow the relevant source namespaces and destination names.
 
> **Production note:** The reference setup is deliberately permissive: it grants the hub controller `cluster-admin` and opens the `default` AppProject with wildcards for source namespaces, repositories and destinations. That is convenient for a first run. For production, scope the RBAC down to what your hub-local apps need and use dedicated, restricted AppProjects.
 
### Example: deploying to the fleet and to the hub
 
A single `ApplicationSet` can fan out to remote clusters by name. The `destination.name` must match the name of the managed cluster:
 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook-fleet
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - cluster: cluster-a
          - cluster: cluster-b
  template:
    metadata:
      name: 'guestbook-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps
        targetRevision: HEAD
        path: guestbook
      destination:
        name: '{{cluster}}'     # destination-based mapping routes this to the agent
        namespace: guestbook
      syncPolicy:
        automated: {}
```
 
A hub-local application uses the standard Argo CD in-cluster destination and is reconciled by the hub controller:
 
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hub-platform-config
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/example/platform-config
    targetRevision: HEAD
    path: hub
  destination:
    name: in-cluster            # the hub itself, not a managed cluster
    namespace: platform
  syncPolicy:
    automated: {}
```
 
### Trade-offs to keep in mind
 
- **Larger hub footprint.** Running the controller on the hub alongside the principal means the hub does more work than in a pure Managed Mode setup. Hub-local apps are reconciled by the hub, not by an agent.
- **Higher privileges on the hub.** The hub controller needs permissions on the hub cluster. Keep them as narrow as your apps allow.
- **Two paths, one rule.** Keeping the hub out of the `GitOpsCluster` placement is what keeps these two paths from colliding.
---
 
## 5. Automating the fleet with `GitOpsCluster`
 
ACM automates the agent lifecycle through the **Open Cluster Management (OCM) Addon Framework**. Three resources work together:
 
| Resource | Role |
|---|---|
| **`Placement`** | Selects target clusters dynamically, for example by labels such as `environment: production`. In Hybrid Mode it must exclude the hub. |
| **`GitOpsCluster`** | The control resource. It references the `Placement`, sets the operating mode, and triggers auto-discovery of the Argo CD server address and port. |
| **GitOps add-on** | Uses OCM `ManifestWork` to deliver the agent and the local Argo CD components to each selected spoke. |
 
When a new cluster matches the `Placement`, the agent is rolled out to it automatically. When a cluster stops matching, it falls out of scope. No manual onboarding is required.
 
### Verifying the rollout
 
```bash
# GitOpsCluster status on the hub
oc get gitopscluster -n openshift-gitops
 
# The add-on should appear for each selected cluster (and not for local-cluster)
oc get managedclusteraddon -A | grep gitops-addon
 
# Confirm which clusters the Placement selected
oc get placementdecision -n openshift-gitops
```
 
Then open the Argo CD UI on the hub and check that your fleet applications show live resource trees.
 
### Common pitfalls
 
| Symptom | Likely cause |
|---|---|
| Agent gets installed on the hub | `local-cluster` was selected by the `Placement`. Exclude it explicitly. |
| `Placement` selects no clusters | Missing `ManagedClusterSetBinding` in the GitOps namespace, or the target label is missing. |
| Install fails on EKS, AKS or GKE | `olmSubscription.enabled: true` was forced. Leave it unset or set it to `false`. |
| Image pull errors in a disconnected cluster | Images are not mirrored, or the `ManagedClusterImageRegistry` is missing, or the cluster lacks the `open-cluster-management.io/image-registry` label. |
| Cluster registration issues for a spoke | The `ManagedCluster` has an empty API URL (`spec.managedClusterClientConfigs[0].url`). |
| Authentication failures after an upgrade | Principal and agent versions are out of sync (see section 7). |
| Apps for a remote cluster do not sync | `destination.name` does not match the managed cluster name, or the AppProject does not allow it. |
 
> **Note:** API versions and field names shown here follow the Hybrid Mode reference setup. Check them against the documentation for your ACM release before applying.
 
---
 
## 6. Non-OpenShift and disconnected environments
 
### Non-OpenShift clusters are fully supported
 
The Argo CD Agent is not limited to OpenShift. It works with non-OpenShift Kubernetes clusters (for example EKS, AKS or GKE) and is fully supported there, using the same `Placement` and `GitOpsCluster` workflow as for OpenShift. You can also mix OpenShift and non-OpenShift clusters in one fleet. A few settings differ:
 
- **OLM.** Do not force `olmSubscription.enabled: true`. That setting is only valid when every selected cluster is OpenShift. Leave it unset so the add-on auto-detects the cluster type, or set it to `false` for mixed or Kubernetes-only fleets.
- **Routes.** OpenShift `Route`s do not exist on plain Kubernetes. The reference setup disables the Argo CD server route on the spokes when non-OpenShift clusters are in the fleet.
- **Cluster API URL.** Make sure each `ManagedCluster` has its API URL set (`spec.managedClusterClientConfigs[0].url`), otherwise cluster registration on the hub can fail.
### Disconnected environments
 
In a disconnected (air-gapped or restricted) environment, the managed clusters cannot pull images from public registries. Because the agent and the local Argo CD components run **on the spokes**, those images have to come from your own mirror registry. This is where `ManagedClusterImageRegistry` comes in.
 
`ManagedClusterImageRegistry` is an ACM resource that overrides the image registry used for the agents ACM deploys to the selected managed clusters, together with a pull secret for that registry. It works in four steps:
 
1. **Mirror** the required images to your internal registry.
2. **Select the clusters** with a `Placement`.
3. **Create the `ManagedClusterImageRegistry`**, referencing the `Placement` and a pull secret that lives in the same namespace.
4. **Label each target `ManagedCluster`** so the override applies to it.
```yaml
apiVersion: imageregistry.open-cluster-management.io/v1alpha1
kind: ManagedClusterImageRegistry
metadata:
  name: disconnected-registry
  namespace: openshift-gitops
spec:
  registry: registry.example.internal:5000      # your mirror registry
  pullSecret:
    name: mirror-pull-secret                    # must be in the same namespace
  placementRef:
    group: cluster.open-cluster-management.io
    resource: placements
    name: disconnected-clusters
```
 
```bash
# Apply the override to a managed cluster: <namespace>.<name> of the ManagedClusterImageRegistry
oc label managedcluster <cluster-name> \
  open-cluster-management.io/image-registry=openshift-gitops.disconnected-registry
```
 
Two more things to plan for in a disconnected setup:
 
- **Git and Helm sources.** Reconciliation happens on the spoke, so each spoke must be able to reach your internal Git repositories (or other sources) for the applications it deploys.
- **Hub connectivity.** The spoke only needs an outbound connection to the Principal. No inbound access to the spoke is required.
> **Note:** Check the exact `ManagedClusterImageRegistry` spec for your ACM release, since the way registries are specified has evolved, and confirm that the GitOps add-on and agent images are covered by the override in your version.
 
---
 
## 7. Security: automated certificate management
 
All communication between the Principal and the agents is encrypted with **mutual TLS (mTLS)**, and the PKI lifecycle is handled for you:
 
1. **Dedicated CA and per-agent certificates.** The `GitOpsCluster` controller manages a dedicated Certificate Authority on the Principal and issues a unique, signed client certificate for each agent.
2. **Secure delivery.** OCM distributes the TLS credentials and connection parameters to the managed clusters via `ManifestWork`.
3. **Version parity.** To avoid authentication failures caused by version mismatches, the controller watches `argoCDAgent.principal.image` on the hub and keeps agents in sync. For testing, you can pin an agent version with the annotation:
```text
   apps.open-cluster-management.io/disable-agent-image-sync=true
```
 
4. **Automatic rotation.** When certificates change, the controller restarts the Principal and agent pods so connectivity continues without manual intervention.
Combined with outbound-only connections and no spoke API credentials held on the hub, this significantly shrinks the attack surface compared with the push model.
 
---
 
## 8. Which mode is right for you?
 
**Choose Managed Mode when:**
 
- You want a single place to define, review, and audit what is deployed across the fleet.
- Compliance requirements call for a guarded central authority.
- The hub is purely a control plane and does not need GitOps for its own components.
- You manage OpenShift and non-OpenShift clusters, including disconnected ones.
**Choose Hybrid Mode when:**
 
- You want everything from Managed Mode *and* GitOps for workloads on the hub itself.
- You want to use `ApplicationSet`s for both the hub and the fleet from one place.
- You can keep the hub out of the `GitOpsCluster` placement and accept running the hub Argo CD controller alongside the principal.
**Consider Autonomous Mode when:**
 
- Clusters must own their application definitions locally.
- You operate in decentralized or air-gapped settings.
---
 
## 9. Conclusion
 
Managed Mode resolves the classic trade-off between scalability and security in multi-cluster GitOps: reconciliation runs where the workloads live, while the hub keeps authoring and observability. Hybrid Mode extends that model so the hub can run its own GitOps workloads through the same set of `Application` and `ApplicationSet` resources, with one clear rule to keep it safe: the `GitOpsCluster` only ever selects remote clusters, never the hub.
 
The design is a particularly good match for edge scenarios, from high-latency satellite links to large retail footprints, where network resilience can't be taken for granted. If a spoke loses contact with the hub, it keeps working.
 
### Next steps
 
- Review the ACM and OpenShift GitOps documentation for the supported configuration of the Argo CD Agent.
- Try Managed or Hybrid Mode on a small, labeled set of clusters using a `Placement` that excludes `local-cluster`.
- Decide per environment whether Managed, Hybrid or Autonomous Mode fits your governance model.
