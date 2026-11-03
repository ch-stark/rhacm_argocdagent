# GitOps for non-OpenShift clusters that cannot reach the public registry

Red Hat Advanced Cluster Management for Kubernetes 5.0 installs the GitOps add-on on clusters that are not OpenShift. In a disconnected network, those clusters still have to pull the GitOps operator, Argo CD, Redis, Dex, and the Argo CD agent from a registry they can actually reach. Opting a `ManagedClusterImageRegistry` into add-on mirroring is how the hub rewrites those image references to your mirror, for the clusters you select.

Platform engineers feel this first. Architects feel it when a fleet standard that works on OpenShift stops at the first imported Kubernetes cluster.

## OpenShift and everyone else install GitOps differently

On an OpenShift managed cluster, the add-on subscribes the GitOps operator through Operator Lifecycle Manager. OLM resolves the operator, Argo CD, Redis, and Dex from its catalog. You mirror that catalog the usual way: a `CatalogSource` served from your mirror, and an `ImageContentSourcePolicy` or `ImageDigestMirrorSet` on the cluster. OLM chooses those image strings from the catalog. An operator pod that still shows `registry.redhat.io` is on that path, which is what this feature leaves in place on OpenShift.

On a non-OpenShift cluster, upstream Kubernetes or a managed service such as Amazon EKS, there is no OLM catalog. The GitOps add-on installs the operator from its own configuration. Every image value in that configuration is what the cluster pulls: the GitOps operator, the Argo CD instance (application controller, repo server, Redis, and Dex), and, in agent mode, the Argo CD agent.

Those values name a public registry by default, usually `registry.redhat.io`. A disconnected cluster cannot open that registry. It can open the private mirror, often the registry on the bastion host. Until this capability, ACM had no supported per-cluster way to send the add-on to that mirror. Administrators set the image references on each cluster and edited them again whenever the hub changed.

## One registry object, for the clusters in a Placement

You opt in on a `ManagedClusterImageRegistry` you already know from Server Foundation. The add-on reads it only when this annotation is set:

```yaml
apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
```

Without the annotation, this feature ignores the object. Other uses of the same API, including klusterlet mirroring, stay in place. Removing the annotation, or deleting the object, restores the original image values. If the same object also mirrors the klusterlet, remove the annotation and keep the object.

The object needs three companions, all in its namespace:

- A `Placement` that selects the clusters to mirror.
- A pull secret for the mirror registry.
- Either one catch-all mirror in `spec.registry`, or source-to-mirror pairs in `spec.registries`. When `spec.registries` is set, `spec.registry` is ignored.

Deploy the add-on with a `GitOpsCluster` first. Mirroring only rewrites an add-on that is already running.

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

## What the hub writes, and what the cluster pulls

Two controllers share the add-on configuration, and they are built to cooperate.

The `GitOpsCluster` controller writes the default image values. With the Argo CD agent enabled, it copies the agent image from the principal pod running on the hub. After a GitOps upgrade on the hub, that live image is the one every agent-enabled managed cluster is brought in line with. The hub OpenShift GitOps operator owns that principal image.

If an opted-in `ManagedClusterImageRegistry` selects the cluster, the image mirroring controller rewrites those values to the mirror. Once a value is mirrored, it stays mirrored until the underlying default changes. After a hub change, the next reconcile rewrites the new default to the mirror.

The add-on framework delivers that configuration to the GitOps add-on agent on the managed cluster. The agent installs the operator from the references it received. The operator starts the Argo CD instance. In agent mode, the Argo CD agent uses the mirrored agent image and connects back to the hub principal.

That is the outcome for a disconnected non-OpenShift cluster:

- One `Placement` and one registry object cover the clusters that share a mirror, instead of an override maintained on each cluster.
- A hub upgrade updates the source image, and the mirror rewrite follows it.
- Turning the feature off restores the original references, so a trial on one cluster ends cleanly.
- The add-on's defaults remain the source of truth. The mirror is a rewrite of those defaults, so the two stay aligned.

## The agent image follows the same rule on OpenShift

The add-on deploys the Argo CD agent directly, separate from the operator OLM installs. It does this on OpenShift and on non-OpenShift, from the image the hub principal is running. Opt the cluster in, and that agent image is rewritten to the mirror with the rest.

| Cluster | Operator, Argo CD, Redis, Dex | Argo CD agent |
| --- | --- | --- |
| OpenShift | OLM catalog and the cluster's own mirror policy | Add-on configuration, mirrored when you opt in |
| Non-OpenShift | Add-on configuration, mirrored when you opt in | Add-on configuration, mirrored when you opt in |

Check which path a cluster is on before you debug a pod. On non-OpenShift, the add-on fills the `openshift-gitops-operator` namespace directly. On OpenShift, look for a GitOps `Subscription`, `InstallPlan`, or `ClusterServiceVersion`.

On the hub, confirm the rewrite in the managed cluster namespace:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> -o yaml
```

On the non-OpenShift cluster, the operator deployment and the Argo CD pods should name the mirror host. On OpenShift, judge this feature by the agent image only.

## What stays your job

**Credentials.** This feature changes where an image is requested. Pull access stays with the secret the add-on already distributes: it sends that secret to the managed cluster, mounts it on the component service accounts, and recreates a pod that fails to pull. Put the mirror's credentials in that same pull-secret chain.

**The registry and the Git server.** Building the mirror, and hosting the Git repository the clusters sync, are outside this feature. A local registry that is missing an image, or a local Git server the cluster cannot reach, will still fail after the references are rewritten.

**Timing.** Mirroring applies after `GitOpsCluster` has reconciled and the add-on is installed. Removing the annotation restores the original values on the next reconcile.

**The path back to the hub.** Image pull and the agent connection are different problems. The hub picks the principal address in this order: Route, then LoadBalancer, then NodePort. A Route counts as found as soon as the object exists, even when the disconnected cluster cannot resolve the hostname. When the managed cluster cannot resolve the Route hostname, point `GitOpsCluster` at a hub node IP and a NodePort. The agent connects to that IP and port.

If the agent logs a certificate error while the principal Secret on the hub is still inside its dates, the principal process is serving the previous certificate from memory. Confirm the Secret with `openssl x509 -dates`, then restart the principal deployment. The next reconnect uses the certificate that was loaded at start.

## Start with one disconnected cluster

Register the non-OpenShift cluster and let `GitOpsCluster` deploy the add-on before you opt in. The ACM 5.0 GitOps documentation covers that install, including registering clusters that are not OpenShift:

<https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/gitops/index>

Then add the annotation on a `ManagedClusterImageRegistry` whose `Placement` contains one disconnected cluster. Confirm the add-on configuration on the hub names your mirror, and confirm the pods on that cluster are pulling it. Widen the `Placement` after that check passes.

Field-level steps, the revert options, and the OpenShift versus non-OpenShift checks are in the GitOps add-on image mirroring guide:

<https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md>
