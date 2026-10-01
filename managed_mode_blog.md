Scaling GitOps: Using Managed Mode of Argo CD Agent in Red Hat Advanced Cluster Management (ACM 5.0)
1. Introduction: The Evolution of Scalable GitOps
The General Availability (GA) of the Argo CD Agent in Red Hat Advanced Cluster Management (ACM) 5.0 and OpenShift GitOps 1.19 represents a fundamental architectural shift for the enterprise. For years, the traditional "push" model was the standard: a centralized Argo CD instance reached out to manage remote clusters. However, as cluster counts moved from dozens to thousands, this model hit a ceiling.
The challenges were not merely incremental; they were exponential. In a traditional push model, the central hub must maintain persistent connections, watch thousands of remote resources, and store the authentication credentials for the entire fleet. This creates a massive security liability—a single hub compromise grants access to every managed cluster—and leads to unsustainable resource consumption on the hub.
The Argo CD Agent introduces the "Best of Both Worlds": a distributed "pull" architecture that preserves centralized observability. By moving reconciliation to the workload clusters while maintaining a bi-directional streaming connection to the hub, we achieve massive scalability and hardened security without losing the single-pane-of-glass management that enterprises demand.
2. Choosing the Right Path: Autonomous vs. Managed Mode
Architects must choose between two operational modes based on where the "Source of Truth" for application lifecycle management resides. While both use the new agent-based transport, they differ significantly in their control-plane logic.
Dimension
Managed Mode
Autonomous Mode
Authoring Location
Hub Cluster (Principal)
Workload Cluster (Managed)
Control over Application Spec
Full Hub Control
Read-Only on Hub (Changes are automatically reverted)
Routing Logic
Destination-based mapping
Local-only reconciliation
Primary Use Case
Centralized GitOps control and policy
Decentralized/Air-gapped autonomy
Senior Architect’s Note: Managed Mode is the definitive choice for compliance-heavy environments (e.g., SOC2, ISO 27001) where the Hub must act as the guarded authority for policy enforcement. It is important to distinguish this from the "Basic" pull model (ManifestWork-only); the Argo CD Agent provides the "Advanced" model, supporting full UI resource trees and live state comparison that basic models lack. Crucially, in Autonomous Mode, any attempt to modify the Application spec via the Hub UI or CLI is automatically reverted to match the agent's local state, ensuring the spoke remains the true master.
3. The Mechanics of Managed Mode
Managed Mode utilizes a sophisticated agent-initiated architecture to outsource compute requirements while maintaining high-fidelity integration with the Principal (Hub).
Centralized Authoring: Applications and ApplicationSets are created on the Principal. The hub remains the single point of entry for configuration.
Agent-Initiated gRPC Pull Streaming: The Agent on the spoke initiates a bi-directional gRPC connection to the Principal. Because the connection is "outbound-only" from the spoke, the hub is topology-agnostic and does not require credentials for the managed cluster. This significantly simplifies network security by eliminating the need for ingress or open firewalls on the workload clusters.
Local Reconciliation: The local application controller on the managed cluster performs the actual resource deployment. A key design principle here is resiliency: if the gRPC connection drops, the workload cluster remains autonomous, continuing to reconcile against its last known state.
High-Fidelity Feedback via Redis Proxy: To keep the central UI performant, the Principal uses a Redis Proxy to route data. Non-agent keys (like cluster metadata) stay on the hub, while specific agent keys—such as app|resources-tree|* and app|managed-resources|*—are routed directly to the agent. This allows the hub UI to display live state and resource trees without the hub having to cache every resource in the fleet.
4. Automating the Fleet: Setup via GitOpsCluster CR
ACM 5.0 automates the lifecycle of the Argo CD Agent through the Open Cluster Management (OCM) Addon Framework. This is orchestrated via three primary resources:
Placement CR: Dynamically selects target managed clusters based on organizational labels (e.g., environment: production).
GitOpsCluster CR: The central control plane resource that references the Placement and defines the operational mode. It also triggers the auto-discovery of the Argo CD server address and port, simplifying initial setup.
OCM Addon Framework: The argocd-agent-addon uses the OCM ManifestWork mechanism to deliver the agent binary and local Argo CD components to the spokes automatically.
A sample GitOpsCluster configuration for Managed Mode:
apiVersion: apps.open-cluster-management.io/v1alpha1
kind: GitOpsCluster
metadata:
  name: prod-gitops-fleet
  namespace: open-cluster-management
spec:
  placementRef:
    kind: Placement
    name: regional-clusters-placement
  argoCDAgentAddon:
    mode: managed
5. Security Deep Dive: Automated Certificate Management
The security posture of the Argo CD Agent is significantly more robust than traditional models. All communication is encrypted via mutual TLS (mTLS), and the PKI lifecycle is entirely hands-off:
CA and Client Signing: The GitOpsCluster controller automatically manages a dedicated Certificate Authority (CA) on the Principal and generates unique, signed TLS client certificates for every agent in the fleet.
Secure Propagation via ManifestWork: OCM uses the ManifestWork mechanism to securely deliver these TLS credentials and connection parameters directly to the managed clusters.
Version Parity and Syncing: To prevent "Auth failure" errors caused by version mismatches, the controller monitors the argoCDAgent.principal.image on the Hub and ensures agents are updated in sync. If you need to pin an agent version for testing, you can use the annotation apps.open-cluster-management.io/disable-agent-image-sync=true.
Automated Rotation: The controller monitors for certificate changes and automatically restarts Principal and Agent pods to ensure continuous, secure connectivity.
6. Conclusion: The Future of Multi-Cluster GitOps
Red Hat ACM 5.0’s implementation of Managed Mode provides an elegant solution to the scalability-security paradox. By shifting to an agent-initiated, pull-based architecture, we reduce the hub's blast radius and resource footprint while providing the high-fidelity observability required for enterprise operations.
This architecture is uniquely suited for the "Edge"—environments where network resiliency is a luxury rather than a guarantee. Whether managing high-latency satellite links or massive retail footprints, the Argo CD Agent ensures that your GitOps pipeline remains secure, scalable, and resilient across any cloud or infrastructure.
