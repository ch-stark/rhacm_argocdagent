# ManagedClusterRegistry: the value it delivers

## The big value

ManagedClusterRegistry makes GitOps workable in disconnected and air-gapped ACM environments without forcing platform teams to hand-maintain image references and SHA overrides for every managed cluster. It gives the ACM GitOps add-on a supported, consistent way to resolve the images it needs from the registries that are actually reachable in that environment.

In plain terms: customers can operate the same GitOps model across connected and disconnected fleets with far less bespoke registry configuration. That removes a common adoption barrier for regulated, edge, and isolated deployments, where direct access to public registries is unavailable or prohibited.

## Customer problem

Today, deploying and updating the GitOps add-on in a disconnected environment can require administrators to determine the correct image SHA references, mirror images, and coordinate several override mechanisms. This creates three costs:

- **Operational friction:** Each change can become a manual, environment-specific configuration exercise.
- **Update risk:** A wrong or incomplete image reference can prevent an add-on from deploying or updating reliably.
- **Uneven experience:** GitOps automation that works in connected environments becomes difficult to standardize across the customer fleet.

## What changes with ManagedClusterRegistry

The feature lets the GitOps add-on use ManagedClusterRegistry to resolve its images in the context of each managed cluster. It is designed to complement existing cluster image-mirroring mechanisms, including ImageContentSourcePolicy and ImageDigestMirrorSet, rather than require customers to abandon them.

The result is a more declarative and reusable configuration path: ACM can understand where a managed cluster can obtain required images and configure the add-on accordingly.

## Why it matters

### Accelerates GitOps adoption in constrained environments

Air-gapped and disconnected environments are common in public sector, financial services, industrial, and edge use cases. ManagedClusterRegistry reduces the extra implementation work these customers face before they can use ACM GitOps at scale.

### Makes fleet operations more repeatable

Instead of treating image configuration as a per-installation exception, teams can apply a consistent registry-resolution approach across managed clusters. That improves repeatability for deployment, upgrades, and support.

### Lowers the risk of configuration drift

Centralizing how the add-on identifies usable image sources reduces the chance that image SHA and mirror configuration diverge between clusters or releases. Fewer manual overrides means fewer hard-to-diagnose failures during rollout.

### Enables the strategic Argo CD agent model

ManagedClusterRegistry is part of ACM's work to advance the `argocd-agent` integration. By making that model viable in disconnected environments, it prevents a major deployment constraint from limiting a scalable, secure GitOps architecture to connected customers only.

## Concise positioning

**ManagedClusterRegistry brings reliable GitOps image resolution to disconnected ACM fleets, so customers can deploy and operate the GitOps add-on consistently without hand-managing registry and image-SHA configuration for every environment.**
