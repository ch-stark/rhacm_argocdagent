# RHACM Search vs. Autonomous Mode (Argo CD Agent)

While RHACM Search provides a high-level inventory of managed Argo CD applications, Autonomous Mode federates spoke-owned applications directly into a central Argo CD control plane with real-time, interactive visibility.

> **Scope:** This document compares ACM Search/Discovery with Argo CD Agent **Autonomous Mode** only. **Managed Mode** — where the hub is the source of truth and pushes `Application` specs to spokes — is intentionally out of scope.

## Feature Comparison

| Feature | RHACM Search & Discovery | Autonomous Mode (Argo CD Agent) |
| --- | --- | --- |
| **Mechanism** | `search-collector` periodically indexes `Application` CRs | Spoke agent streams app data to hub Argo CD via gRPC |
| **Primary UI** | RHACM Console (Search, Topology, Lifecycle) | Centralized hub Argo CD UI / API |
| **Data Freshness** | Periodic scraping and indexing | Continuous, real-time status stream |
| **Detail Level** | CR metadata, basic health, high-level topology | Full resource trees, manifest diffs, detailed sync errors |
| **Hub Role** | Passive read-only observer | Active observability; **sync/terminate** from hub; **spec is read-only** |
| **Source of truth** | Whatever cluster already owns the `Application` CR | Managed (spoke) cluster |

## Core Architectural Difference

Autonomous Mode turns the hub into an active multi-cluster Argo CD platform without stripping spoke clusters of local authority. Rather than merely discovering `kind: Application` metadata in ACM Search, it mirrors spoke-managed applications into the central Argo CD instance.

- **Full Operational Visibility:** Resource trees, live vs. desired state, manifest diffs, and granular sync errors — not just basic status roll-ups.
- **Low-Latency Streaming:** Replaces scheduled index polling with gRPC status streams from spoke agents.
- **Hub-Initiated Operations:** Operators can trigger **Sync** and **Terminate** from the hub Argo CD UI/CLI/API; those actions are forwarded to the spoke agent.
- **Preserved Local Authority:** The managed cluster owns the `Application` spec. Local changes are mirrored to the hub, not overwritten by hub-side spec edits.

> **Note — Search and Autonomous Mode are complementary:** Autonomous mode does not replace ACM Search. Every `Application` reconciled on a managed cluster in autonomous mode is still indexed by `search-collector` and discoverable from the hub via ACM Search, Application Lifecycle, and topology views (e.g. `kind:application` and `apigroup:argoproj.io`). You can view and query the same fleet of applications from the ACM console while also using the hub Argo CD UI for deep GitOps operations. The two approaches overlap in visibility but differ in depth: Search gives you ACM-native fleet queries and roll-ups; autonomous mode adds full Argo CD UI integration with real-time status and sync/terminate from the hub.

## Expected Behavior — Hub (Mirrored, Spec-Read-Only Application)

In autonomous mode, the spoke cluster is the source of truth for `Application` definitions. The `argocd-agent` on the spoke reports application state to the hub principal, which mirrors it as an `Application` in a dedicated hub namespace named after the managed cluster (`<managed-cluster-name>`).

### Verify mirrored status on the hub

```bash
oc config use-context <hub-context>

oc get application test-autonomous -n <managed-cluster-name> \
  -o jsonpath='{.status.health.status}{"\n"}{.status.sync.status}{"\n"}'
# Healthy
# Synced
```

### What is read-only vs. what still works from the hub

| From the hub | Allowed? | Notes |
| --- | --- | --- |
| View health, sync status, resource tree | ✅ | Mirrored from spoke |
| Trigger **Sync** | ✅ | Forwarded to spoke agent |
| Trigger **Terminate** | ✅ | Forwarded to spoke agent |
| Edit **spec** (Git repo/path, destination, sync policy, parameters) | ❌ | Reverted to match spoke |
| Create `Application` on hub for autonomous spoke | ❌ | Create on spoke; hub mirrors it |
| Delete `Application` on hub | ❌ | Deletion must happen on spoke |

**Key point:** In autonomous mode, the spoke owns the `Application` spec. The hub mirrors it for visibility and can trigger sync/terminate, but cannot author or permanently change the application configuration. The managed cluster is the source of truth for application definitions.

### Prerequisites (autonomous mode)

- `GitOpsCluster` with `gitopsAddon.argoCDAgent.enabled: true` and `mode: autonomous`
- Hub `ArgoCD` CR `spec.sourceNamespaces` includes each managed cluster name (must match agent namespace mapping)
- `Application` and required `AppProject` resources are created on the **spoke** (typically via Git / app-of-apps)

## Decision Guide

- **Choose RHACM Search / Discovery** if you already run standalone Argo CD on managed clusters, want zero extra agent infrastructure, and only need fleet inventory and high-level status in the ACM console.
- **Choose Argo CD Agent Autonomous Mode** if you need a single, real-time Argo CD UI across the fleet, want hub-initiated sync/terminate while spokes keep spec ownership, and are standardizing on the RHACM GitOps add-on.
- **Use both** when you want ACM Search for fleet-wide queries and topology roll-ups **and** hub Argo CD for detailed GitOps operations on the same autonomous-mode applications.

## Summary

RHACM Search gives you a cluster manager's inventory view of `Application` CRs across the fleet. Autonomous Mode delivers true Argo CD multi-cluster federation: spoke-owned configuration, hub-side observability and operational control — without making the hub the source of truth for application definitions. Applications managed in autonomous mode appear in both places: indexed in ACM Search and mirrored in the hub Argo CD UI.

