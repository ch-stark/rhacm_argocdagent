# GitOps image mirroring for clusters that are not OpenShift

Red Hat Advanced Cluster Management for Kubernetes 5.0 installs the GitOps add-on on managed clusters that are not OpenShift. In a disconnected network those clusters still have to pull the GitOps operator, Argo CD, Redis, Dex, and the Argo CD agent. `ManagedClusterImageRegistry` is how the hub rewrites those image references to a mirror, for the clusters you select.

## What is the value?

A disconnected cluster that is not OpenShift cannot pull from `registry.redhat.io`. The GitOps add-on installs from image strings in its own configuration, and those strings name the public registry by default.

Opting a `ManagedClusterImageRegistry` into add-on mirroring rewrites those strings to the mirror before the add-on is delivered. The hub stays the owner of the image list.

- One registry object and one `Placement` cover every cluster that shares a mirror. You stop keeping a separate image override on each cluster.
- A GitOps upgrade on the hub changes the source image. The next reconcile rewrites the new source to the mirror, including the Argo CD agent image copied from the principal on the hub.
- Removing the annotation restores the original references. A trial on one cluster ends by taking the annotation off.
- The add-on's defaults remain the source of truth. The mirror is a rewrite of those defaults, so the two stay aligned.

Platform engineers meet this on the first imported EKS or upstream Kubernetes cluster. Architects meet it when a fleet standard that already works on OpenShift stops there.

## Can an ACM Policy do this?

A Policy can patch image fields. It is the wrong writer for these fields.

The `GitOpsCluster` controller writes the image values into the add-on configuration. The add-on agent on the managed cluster installs from that configuration. A Policy that then patches the operator Deployment or the Argo CD custom resource on the spoke is a second writer on objects the add-on reconciles. The add-on puts the public registry back, or the Policy stays noncompliant and the objects flap.

A Policy that hardcodes the mirror strings has to be edited when the hub GitOps version changes a digest or adds a component. This feature rewrites whatever the add-on just wrote, so the upgrade flows through.

Deleting a Policy leaves the last patched images in place unless a second remediation puts the defaults back. Removing the mirroring annotation restores the defaults on the next reconcile.

Use a Policy for configuration ACM does not own. These image strings are owned by the add-on. Rewriting them before they are delivered keeps a single owner, uses the `Placement` already attached to `ManagedClusterImageRegistry`, and reverts with one annotation.

## Why OpenShift does not need it

On an OpenShift managed cluster the add-on subscribes the GitOps operator through Operator Lifecycle Manager. OLM resolves the operator, Argo CD, Redis, and Dex from its catalog. A disconnected OpenShift cluster already serves that catalog from a mirror `CatalogSource` and rewrites `registry.redhat.io` with an `ImageDigestMirrorSet` or `ImageContentSourcePolicy`. Those image strings never come from the add-on configuration, so this feature has nothing to rewrite for them. An operator pod that still shows `registry.redhat.io` is on that OLM path.

The Argo CD agent is the exception. The add-on deploys the agent directly, on OpenShift and on other Kubernetes, from the image the hub principal is running. Opt an OpenShift cluster in only when that agent image must follow a `Placement`-specific mirror. Everything OLM installs stays on the cluster's own mirror policy.

| Cluster | Operator, Argo CD, Redis, Dex | Argo CD agent |
| --- | --- | --- |
| OpenShift | OLM catalog and the cluster mirror policy | Add-on configuration. Mirror it here only for a Placement-specific mirror |
| Other Kubernetes | Add-on configuration, mirrored when you opt in | Add-on configuration, mirrored when you opt in |

Check which path a cluster is on before you debug a pod. On other Kubernetes, the add-on fills the `openshift-gitops-operator` namespace directly. On OpenShift, look for a GitOps `Subscription`, `InstallPlan`, or `ClusterServiceVersion`. On OpenShift, judge this feature by the agent image only.

## How to opt in

The add-on reads a `ManagedClusterImageRegistry` only when this annotation is set:

```yaml
apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
```

Without the annotation, this feature ignores the object. Other uses of the same API, including klusterlet mirroring, stay in place. If the same object also mirrors the klusterlet, remove the annotation and keep the object.

The object needs three companions, all in its namespace:

- A `Placement` that selects the clusters to mirror.
- A pull secret for the mirror registry.
- Either one catch-all mirror in `spec.registry`, or source-to-mirror pairs in `spec.registries`. When `spec.registries` is set, `spec.registry` is ignored.

Deploy the add-on with a `GitOpsCluster` first. Mirroring rewrites an add-on that is already running.

```yaml
apiVersion: imageregistry.open-cluster-management.io/v1alpha1
kind: ManagedClusterImageRegistry
metadata:
  name: gitops-addon-mirror
  namespace: openshift-gitops
  annotations:
    apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
spec:
  pullSecret:
    name: mirror-registry-pull-secret
  placementRef:
    group: cluster.open-cluster-management.io
    resource: placements
    name: disconnected-clusters-placement
  registries:
    - source: ""
      mirror: my-mirror.example.com:5000/default
    - source: registry.redhat.io
      mirror: my-mirror.example.com:5000/redhat
```

Matching is last-match-wins. An empty `source` matches every host. Put that catch-all first and the specific registries after it, so `registry.redhat.io` keeps its own mirror.

The three objects share a namespace. `openshift-gitops` is the usual place to put them.

## What the hub writes

Two controllers share the add-on configuration.

The `GitOpsCluster` controller writes the default image values. With the Argo CD agent enabled, it copies the agent image from the principal pod running on the hub. After a GitOps upgrade on the hub, that live image is the one every agent-enabled managed cluster is brought in line with.

If an opted-in `ManagedClusterImageRegistry` selects the cluster, the image mirroring controller rewrites those values to the mirror. Once a value is mirrored, it stays mirrored until the underlying default changes. After a hub change, the next reconcile rewrites the new default to the mirror.

The add-on framework delivers that configuration to the GitOps add-on agent on the managed cluster. The agent installs the operator from the references it received. The operator starts the Argo CD instance. In agent mode, the Argo CD agent uses the mirrored agent image and connects back to the hub principal.

On the hub, confirm the rewrite in the managed cluster namespace:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> -o yaml
```

On a non-OpenShift cluster, the operator deployment and the Argo CD pods should name the mirror host.

## What stays your job

**Credentials.** This feature changes where an image is requested. Pull access stays with the secret the add-on already distributes: it sends that secret to the managed cluster, mounts it on the component service accounts, and recreates a pod that fails to pull. Put the mirror's credentials in that same pull-secret chain.

**The registry and the Git server.** Building the mirror, and hosting the Git repository the clusters sync, are outside this feature. A local registry that is missing an image, or a local Git server the cluster cannot reach, still fails after the references are rewritten.

**Timing.** Mirroring applies after `GitOpsCluster` has reconciled and the add-on is installed. Removing the annotation restores the original values on the next reconcile.

**The path back to the hub.** Image pull and the agent connection are different problems. The hub picks the principal address in this order: Route, then LoadBalancer, then NodePort. A Route counts as found as soon as the object exists, even when the disconnected cluster cannot resolve the hostname. When the managed cluster cannot resolve the Route hostname, point `GitOpsCluster` at a hub node IP and a NodePort.

## Start with one cluster

Register the non-OpenShift cluster and let `GitOpsCluster` deploy the add-on before you opt in. The ACM 5.0 GitOps documentation covers that install:

<https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/gitops/index>

Then add the annotation on a `ManagedClusterImageRegistry` whose `Placement` contains one disconnected cluster. Confirm the add-on configuration on the hub names your mirror, and confirm the pods on that cluster are pulling it. Widen the `Placement` after that check passes.

Field-level steps, the revert options, and the OpenShift versus non-OpenShift checks are in the GitOps add-on image mirroring guide:

<https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md>
