WIP

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
| **Hub Role** | Fleet discovery **and** hub-side edit of live resources on managed clusters (RBAC-dependent) | Argo CD observability; **sync/terminate** from hub; **mirrored spec is read-only** |
| **Edit from hub** | ✅ Edit the **live** resource on the managed cluster via ACM Search/console | ❌ Hub Argo CD mirror spec is read-only; ✅ edit the `Application` on the **spoke** (or via Search) |
| **Source of truth** | Whatever cluster already owns the `Application` CR | Managed (spoke) cluster |

## Core Architectural Difference

Autonomous Mode turns the hub into an active multi-cluster Argo CD platform without stripping spoke clusters of local authority. Rather than merely discovering `kind: Application` metadata in ACM Search, it mirrors spoke-managed applications into the central Argo CD instance.

- **Full Operational Visibility:** Resource trees, live vs. desired state, manifest diffs, and granular sync errors — not just basic status roll-ups.
- **Low-Latency Streaming:** Replaces scheduled index polling with gRPC status streams from spoke agents.
- **Hub-Initiated Operations:** Operators can trigger **Sync** and **Terminate** from the hub Argo CD UI/CLI/API; those actions are forwarded to the spoke agent.
- **Preserved Local Authority:** The managed cluster owns the `Application` spec. Local changes are mirrored to the hub Argo CD copy, not overwritten by hub-side Argo CD spec edits.

> **Note — Search and Autonomous Mode are complementary:** Autonomous mode does not replace ACM Search. Every `Application` reconciled on a managed cluster in autonomous mode is still indexed by `search-collector` and discoverable from the hub via ACM Search, Application Lifecycle, and topology views (e.g. `kind:application` and `apigroup:argoproj.io`). You can view and query the same fleet of applications from the ACM console while also using the hub Argo CD UI for deep GitOps operations.
>
> **Important:** ACM Search is not view-only. From the hub console you can also **edit resources on managed clusters directly** — including `Application` CRs — subject to your RBAC permissions. That is a different interface and a different constraint than the hub Argo CD mirror in autonomous mode (where the mirrored `Application` spec cannot be changed from the Argo CD UI).

## Editing from the Hub — Search vs. Autonomous Argo CD

This is easy to conflate because both are accessed from the hub:

| Action | Via ACM Search / console | Via hub Argo CD UI (autonomous mirror) |
| --- | --- | --- |
| Find `Application` across fleet | ✅ | ✅ (per-cluster namespace) |
| View health / topology | ✅ (ACM views) | ✅ (full Argo CD resource tree) |
| Edit `Application` **spec** on managed cluster | ✅ — edits the **live CR on the spoke** | ❌ — hub mirror spec is read-only and reverted |
| Trigger Sync / Terminate | ❌ (not an Argo CD action) | ✅ — forwarded to spoke agent |

So autonomous mode does **not** block all hub-side changes to applications. It blocks **Argo CD UI spec edits on the mirrored copy**. Operators with sufficient RBAC can still open the same `Application` in ACM Search and edit it on the managed cluster from the hub.

> **GitOps caveat:** Editing an `Application` CR via ACM Search changes the live object on the spoke, bypassing Git. In a proper GitOps workflow, configuration changes should still go through the spoke's Git repository — not ad-hoc console edits — regardless of which hub interface you use.

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

### What is read-only vs. what still works from the hub (Argo CD UI)

| From the hub Argo CD UI | Allowed? | Notes |
| --- | --- | --- |
| View health, sync status, resource tree | ✅ | Mirrored from spoke |
| Trigger **Sync** | ✅ | Forwarded to spoke agent |
| Trigger **Terminate** | ✅ | Forwarded to spoke agent |
| Edit **spec** on hub mirror (Git repo/path, destination, sync policy, parameters) | ❌ | Reverted to match spoke |
| Create `Application` on hub for autonomous spoke | ❌ | Create on spoke; hub mirrors it |
| Delete `Application` on hub mirror | ❌ | Deletion must happen on spoke |

| From ACM Search / console (same app, different path) | Allowed? | Notes |
| --- | --- | --- |
| View / query `Application` on managed cluster | ✅ | Indexed by `search-collector` |
| Edit `Application` **spec** on managed cluster | ✅ | Edits the live CR on the spoke (RBAC-dependent) |
| Trigger Sync / Terminate | ❌ | Use hub Argo CD UI for these operations |

**Key point:** In autonomous mode, the spoke owns the `Application` spec. The hub Argo CD UI mirrors it for visibility and can trigger sync/terminate, but cannot author or permanently change the mirrored application configuration. The managed cluster remains the source of truth — and you can still reach that live object from the hub via ACM Search if your role allows it.

### Prerequisites (autonomous mode)

- `GitOpsCluster` with `gitopsAddon.argoCDAgent.enabled: true` and `mode: autonomous`
- Hub `ArgoCD` CR `spec.sourceNamespaces` includes each managed cluster name (must match agent namespace mapping)
- `Application` and required `AppProject` resources are created on the **spoke** (typically via Git / app-of-apps)

## Decision Guide

- **Choose RHACM Search / Discovery** if you already run standalone Argo CD on managed clusters, want zero extra agent infrastructure, and need fleet inventory, topology roll-ups, **and the ability to edit live resources on managed clusters from the hub** (subject to RBAC).
- **Choose Argo CD Agent Autonomous Mode** if you need a single, real-time Argo CD UI across the fleet, want hub-initiated sync/terminate while spokes keep spec ownership, and are standardizing on the RHACM GitOps add-on.
- **Use both** when you want ACM Search for fleet-wide queries, topology, and hub-side CR editing **and** hub Argo CD for detailed GitOps operations (resource trees, diffs, sync/terminate) on the same autonomous-mode applications.

## Summary

RHACM Search gives you a cluster manager's view of `Application` CRs across the fleet — including the ability to **edit live resources on managed clusters from the hub**. Autonomous Mode delivers true Argo CD multi-cluster federation: spoke-owned configuration, hub Argo CD observability, and sync/terminate — with the **hub Argo CD mirror** spec read-only. Applications in autonomous mode appear in both places; choose the right interface for the job: ACM Search for fleet queries and direct CR edits, hub Argo CD for deep GitOps operations.
