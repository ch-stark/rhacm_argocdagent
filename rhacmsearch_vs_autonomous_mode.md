# RHACM Search vs. Autonomous Mode (Argo CD Agent)

While RHACM Search provides a high-level inventory of managed Argo CD applications, Autonomous Mode federates spoke-owned applications directly into a central Argo CD control plane with real-time, interactive visibility.

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

## Expected Behavior — Hub (Mirrored, Spec-Read-Only Application)

In autonomous mode, the spoke cluster is the source of truth for `Application` definitions. The `argocd-agent` on the spoke reports application state to the hub principal, which mirrors it as an `Application` in a dedicated hub namespace named after the managed cluster (`<managed-cluster-name>`).

### Verify mirrored status on the hub

```bash
oc config use-context <hub-context>

oc get application test-autonomous -n <managed-cluster-name> \
  -o jsonpath='{.status.health.status}{"\n"}{.status.sync.status}{"\n"}'
# Healthy
# Synced
