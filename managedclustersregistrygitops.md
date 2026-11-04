# Run the GitOps add-on from your mirror registry

Red Hat Advanced Cluster Management for Kubernetes (ACM) 5.0 can install the GitOps add-on on managed clusters that are not OpenShift. Those clusters pull the GitOps operator, Argo CD, Redis, Dex, and the Argo CD agent. On a disconnected network that pull has to come from a registry the cluster can already reach.

This post is for that case. It covers what the mirroring feature is worth, why an ACM policy is a poor substitute, and why an OpenShift managed cluster uses a different path.

## The problem

The add-on installs several images on each managed cluster:

- the GitOps operator
- the Argo CD instance it creates (application controller, repository server, Redis, and Dex)
- in agent mode, the Argo CD agent that connects back to the hub

By default those images are pulled from a public registry, usually `registry.redhat.io`. A disconnected cluster cannot open that registry. It can open the private mirror you already run.

Before this capability, ACM had no supported per-cluster way to send the add-on to that mirror. Administrators set the image references on each cluster, then edited them again whenever the hub changed.

## What you get

You opt in on a `ManagedClusterImageRegistry`. ACM already uses that API for image overrides, including the klusterlet (the agent ACM installs so a cluster can register). The GitOps add-on reads the object only when you set this annotation:

```yaml
apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
```

Without the annotation, this feature ignores the object. Klusterlet mirroring, and any other use of the same object, stays as it is. Adding the annotation does not change clusters that are not using the GitOps add-on.

For the clusters a `Placement` selects:

- One registry object covers every cluster that shares a mirror. You do not repeat the mirror host on each cluster.
- The add-on's own defaults stay the source of truth. The mirror is a rewrite of those defaults, so the two do not drift apart.
- Mirrored values stay on the mirror until the underlying default changes. After a GitOps upgrade on the hub, the next reconcile rewrites the new default to the mirror. You do not re-apply the mirror by hand.
- Turning the annotation off, or deleting the object, restores the original image references. The restore waits for the next reconcile. A running pod is not restarted only because the reference changed.
- The Argo CD agent image is included. The hub copies the image from the principal running on the hub, which the OpenShift GitOps operator on the hub owns. The mirror rewrite follows that image.

## Why an ACM policy is a poor substitute

A governance policy can patch image fields. It fights the component that owns them.

The `GitOpsCluster` controller writes the image values into the add-on configuration. The add-on agent on the managed cluster installs from that configuration. A policy that then patches the operator Deployment or the Argo CD custom resource on the managed cluster is a second writer. The add-on puts the public registry back, or the policy stays noncompliant and the objects flap.

A policy that hardcodes mirror digests has to be edited when the hub GitOps version changes a digest or adds a component. This feature rewrites whatever the add-on just wrote, so the upgrade flows through.

Removing a policy leaves the last patched images in place unless you add a second remediation to restore the defaults. Removing the mirroring annotation restores the defaults on the next reconcile.

Use a policy for configuration ACM does not own. These image strings belong to the add-on.

## Why OpenShift managed clusters do not use this path

On an OpenShift managed cluster the add-on subscribes the GitOps operator through Operator Lifecycle Manager (OLM). OLM resolves the operator, Argo CD, Redis, and Dex from its catalog. A disconnected OpenShift cluster already serves that catalog from a mirror `CatalogSource` and rewrites `registry.redhat.io` with an `ImageDigestMirrorSet` or `ImageContentSourcePolicy`. Those image strings never come from the add-on configuration. Rewriting them with this feature has no effect on the operator install. An operator pod that still shows `registry.redhat.io` is on the OLM path.

The Argo CD agent is the exception on OpenShift as well. The add-on deploys the agent directly, from the image the hub principal is running. It is not part of the OLM-resolved operator. Opt an OpenShift cluster in only when that agent image must follow a placement-specific mirror. Everything OLM installs stays on the cluster's own mirror policy.

| | Other Kubernetes | OpenShift |
| --- | --- | --- |
| Who installs the operator and Argo CD | The add-on, from the image references in its configuration | OLM, from the catalog |
| Disconnected path | This annotation plus `ManagedClusterImageRegistry` | Mirror `CatalogSource` plus `ImageDigestMirrorSet` or `ImageContentSourcePolicy` |
| Argo CD agent image | Rewritten with the rest when you opt in | Opt in only if that one image must follow a placement-specific mirror |

How to tell which path a cluster is on:

- Other Kubernetes: the add-on creates the operator pods in `openshift-gitops-operator`.
- OpenShift: look for a GitOps `Subscription`, `InstallPlan`, or `ClusterServiceVersion`.

## Before you start

Mirroring extends a GitOps deployment. It does not create one.

- A `GitOpsCluster` has already deployed the add-on. If the add-on is not installed, the mirror configuration sits unused until it is.
- A `Placement` in the same namespace as the `ManagedClusterImageRegistry`.
- A pull secret for the mirror, in that same namespace.
- The mirror's credentials are already in the pull-secret chain the add-on distributes. The add-on sends that secret to the managed cluster, mounts it on the component service accounts, and recreates a pod that fails to pull. This feature changes the image location, not how the cluster authenticates.

`openshift-gitops` is the usual namespace for these objects.

You still build the mirror and the Git server yourself. A missing image in the mirror, or a Git server the managed cluster cannot reach, is outside this feature.

## Turn mirroring on

Annotate the registry object and point it at the clusters and the secret:

```yaml
apiVersion: imageregistry.open-cluster-management.io/v1alpha1
kind: ManagedClusterImageRegistry
metadata:
  name: gitops-disconnected
  namespace: openshift-gitops
  annotations:
    apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
spec:
  placementRef:
    group: cluster.open-cluster-management.io
    name: disconnected-clusters
    resource: placements
  pullSecret:
    name: mirror-pull-secret
  registry: private-registry.example.com/rhacm
```

`registry` is one mirror for every source. `registries` is a list of source-and-mirror pairs. If you set `registries`, `registry` is ignored.

The last matching pair wins. Put an empty `source` first when you want a catch-all, and put specific hosts after it:

```yaml
spec:
  registries:
    - source: ""
      mirror: private-registry.example.com/mirror
    - source: registry.redhat.io
      mirror: private-registry.example.com/rh
    - source: quay.io
      mirror: private-registry.example.com/quay
```

`registry.redhat.io/openshift-gitops-1/gitops-rhel8` then resolves to `private-registry.example.com/rh/openshift-gitops-1/gitops-rhel8`. A host that is not listed falls through to the empty source.

Two controllers share the work. The `GitOpsCluster` controller writes the default image values, and copies the agent image from the hub principal, into the add-on configuration for each managed cluster. The mirroring controller rewrites those values to the mirror for clusters that opted in. When the default changes, it rewrites again. You do not edit the add-on configuration by hand.

## Check the rewrite

On the hub, read the add-on configuration for one managed cluster. The image references should name the mirror host:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> -o yaml
```

On a managed cluster that is not OpenShift, the operator and Argo CD pods should name the same host. On OpenShift, the operator, Redis, and Dex still come from OLM. Only the agent image is a meaningful check for this feature on OpenShift.

Hub logs for the controllers that write and rewrite the values:

```bash
oc logs -n open-cluster-management multicluster-operators-application-<pod> -c multicluster-operators-gitopscluster
oc logs -n openshift-gitops openshift-gitops-agent-principal-<pod>
```

On the managed cluster:

```bash
oc logs -n open-cluster-management-agent-addon gitops-addon-<pod>
oc logs -n openshift-gitops acm-openshift-gitops-agent-agent-<pod>
oc get argocd -n openshift-gitops acm-openshift-gitops -o yaml
```

Substitute the pod names from `oc get pods`.

## Turn mirroring off

Both of these restore the source image references. Neither is instant. Wait for the next reconcile.

- Remove `apps.open-cluster-management.io/gitops-addon-image-mirroring` and keep the object, when the same object also mirrors the klusterlet.
- Delete the `ManagedClusterImageRegistry`.

## The agent image follows the hub

There is no separate setting for the agent image. On every managed cluster where the agent is enabled, the hub keeps that image equal to the image the hub principal is running. After an OpenShift GitOps upgrade on the hub, the new principal image is what the managed clusters receive. If the cluster is opted into mirroring, that image is rewritten to the mirror with the rest.

## If the pods pull and the agent still cannot connect

These are not image-mirror failures. They show up in the same disconnected setup, after the images are already on the cluster.

**The agent cannot resolve the principal hostname.** The hub chooses the principal address in this order: Route, then LoadBalancer, then NodePort. A Route counts as found as soon as the object exists. The hub does not check that the managed cluster can resolve the hostname. The agent log then reports `name resolver error: produced zero addresses`. Point `GitOpsCluster` at a hub node IP and a NodePort. The agent then needs IP and port reachability, not DNS for the Route.

**The agent reports an expired principal certificate while the Secret is still valid.** The principal renews the certificate in the Secret and keeps serving the previous certificate from memory until the process restarts. Check the Secret with `openssl x509 -dates`. If the dates are current, restart the principal Deployment. The next reconnect uses the certificate loaded at start.

## Prove it on one cluster

Select one disconnected managed cluster in the `Placement`. Confirm the add-on configuration and the pods name the mirror host. Remove the annotation, confirm the source references return, then add the annotation again. Widen the `Placement` only after that cluster stays on the mirror across a reconcile.

Then deploy an application through the agent. The mirror work is done when the pods are running from the private registry. Application sync is a separate check: the managed cluster still has to reach the Git server.

## Further reading

- [GitOps with Advanced Cluster Management 5.0](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/gitops/index)
- [Applications](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/applications/index)
- [GitOps add-on image mirroring](https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md) — field reference for the annotation, the two controllers, and the registry match order
