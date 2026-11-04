# Mirror GitOps add-on images on Kubernetes

This is how you mirror the GitOps add-on on Kubernetes distributions other than OpenShift. Those clusters cannot use Operator Lifecycle Manager. When they also cannot reach `registry.redhat.io`, Red Hat Advanced Cluster Management for Kubernetes (ACM) 5.0 sends the add-on pulls to a mirror you already run, for the clusters you choose.

On OpenShift, skip this feature for the operator, Argo CD, Redis, and Dex. Operator Lifecycle Manager installs those from the catalog, and a disconnected OpenShift cluster already mirrors them with a `CatalogSource` plus an `ImageDigestMirrorSet` or `ImageContentSourcePolicy`. The one OpenShift case for opting in is the Argo CD agent image, and only when that image must follow a placement-specific mirror. [OpenShift already has a mirror path](#openshift-already-has-a-mirror-path) is the detail.

This post is the practical version of that Kubernetes setup: what the feature is worth, how to turn it on, and what to check when the pods are up and the agent still cannot reach the hub.

The field reference, including the architecture diagram, is the [GitOps add-on image mirroring guide](https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md).

## What the add-on pulls

On each Kubernetes cluster that is not OpenShift, the add-on installs these images from its own configuration:

- the OpenShift GitOps operator
- the Argo CD instance that operator creates: application controller, repository server, Redis, and Dex
- when agent mode is on, the Argo CD agent that connects back to the principal on the hub

Those images come from a public registry unless you redirect them. Before this capability, the redirect was a per-cluster edit, and every hub change meant editing it again.

## What you turn on

On those Kubernetes clusters, the switch is an annotation on a `ManagedClusterImageRegistry`, the same API ACM already uses for image overrides such as the klusterlet:

```yaml
apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
```

The GitOps add-on ignores the object until that annotation is present. Klusterlet mirroring, and any other use of the same object, continues as before. Clusters outside the `Placement` are left alone.

Once a cluster is opted in:

- One registry object covers every cluster in its `Placement`.
- The add-on defaults stay the source of truth. A second controller rewrites those defaults to the mirror. When a default changes, after a GitOps upgrade on the hub for example, the rewrite follows. You do not re-apply the mirror by hand.
- Removing the annotation, or deleting the object, restores the recorded original image values on the next reconcile.
- The Argo CD agent image is part of the same rewrite.

## OpenShift already has a mirror path

On an OpenShift managed cluster the add-on subscribes the GitOps operator through Operator Lifecycle Manager. OLM resolves the operator, Argo CD, Redis, and Dex from its catalog. A disconnected OpenShift cluster already serves that catalog from a mirror `CatalogSource`, and rewrites `registry.redhat.io` with an `ImageDigestMirrorSet` or `ImageContentSourcePolicy`.

Rewriting those same images through `ManagedClusterImageRegistry` is harmless on OpenShift, and it does nothing to the operator install. An operator pod that still shows `registry.redhat.io` is the expected OLM result. Time spent debugging that pod is time spent on the wrong path.

The Argo CD agent is the exception. The add-on deploys the agent itself, from the image the hub principal is running. That image is under ACM control on OpenShift and on other Kubernetes distributions. Opt an OpenShift cluster in when that agent image must follow a placement-specific mirror. Leave the operator, Argo CD, Redis, and Dex on the OpenShift catalog mirror.

| | Other Kubernetes | OpenShift |
| --- | --- | --- |
| Operator, Argo CD, Redis, Dex | Installed by the add-on from its configuration. Mirroring applies. | Installed by OLM from the catalog. Mirror them with `CatalogSource` and `ImageDigestMirrorSet` or `ImageContentSourcePolicy`. |
| Argo CD agent | Mirrored with the rest when you opt in. | Same agent control as any other cluster. Opt in when this image must use the placement mirror. |
| How to tell | Pods exist in `openshift-gitops-operator`. | A GitOps `Subscription`, `InstallPlan`, or `ClusterServiceVersion` in `openshift-operators`. |

```bash
oc get pods -n openshift-gitops-operator
oc get subscription,installplan,csv -n openshift-operators | grep -i gitops
```

## A mirror for one fleet

Mirroring starts from a GitOps add-on that `GitOpsCluster` has already deployed. Until that configuration exists, there is nothing to rewrite.

In the same namespace, usually `openshift-gitops`, you need:

- a `Placement` for the clusters
- a pull secret for the mirror, referenced by the registry object
- that mirror's credentials already present in the pull-secret chain the add-on distributes today

The add-on propagates that chain to every managed cluster, mounts it on the component service accounts, and recreates a pod that cannot pull. Mirroring changes the image location that gets requested. Access to the mirror stays on the pull secret you already trust.

Create the secret:

```bash
oc create secret docker-registry mirror-registry-pull-secret \
  -n openshift-gitops \
  --docker-server=my-mirror.example.com:5000 \
  --docker-username=<user> \
  --docker-password=<token>
```

Select the clusters:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: disconnected-clusters-placement
  namespace: openshift-gitops
spec:
  clusterSets: [global]
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            environment: disconnected
```

Then create the registry object. `spec.registries` is a list of source-and-mirror pairs. Entries are read in order. An entry matches when its `source` is empty or equals the image's registry host, and the last match wins. Put a catch-all first and the specific hosts after it, so the specific hosts override the catch-all:

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
    - source: quay.io
      mirror: my-mirror.example.com:5000/quay
```

`registry.redhat.io/openshift-gitops-1/gitops-rhel8` resolves to `my-mirror.example.com:5000/redhat/openshift-gitops-1/gitops-rhel8`. A host you did not list falls through to the empty `source`.

When every image goes to one mirror, `spec.registry` is enough. It is ignored as soon as `spec.registries` is set:

```yaml
spec:
  registry: my-mirror.example.com:5000
```

Two controllers share the hub-side work. The `GitOpsCluster` controller writes the default image values into each managed cluster's add-on configuration, and copies in the image the hub principal is running. If you opted in, the mirroring controller rewrites the matching values to the mirror and records the originals so they can be restored. A mirrored value stays mirrored until the underlying default actually changes. The add-on framework then delivers that configuration. On the managed cluster, the add-on installs from the values it received.

You still build the mirror, and you still run the Git server the applications sync from. A missing image in the mirror, or a Git server the cluster cannot reach, sits outside this feature.

## See that it took

On the hub, a finalizer on the registry object means the controller has started managing it. The add-on configuration for a selected cluster should name the mirror, with bookkeeping annotations that remember the pre-mirror values:

```bash
oc get managedclusterimageregistry gitops-addon-mirror -n openshift-gitops \
  -o jsonpath='{.metadata.finalizers}{"\n"}'

oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> -o yaml
```

To see each rewritten variable on its own:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> \
  -o jsonpath='{range .spec.customizedVariables[*]}{.name}={.value}{"\n"}{end}'
```

On a managed cluster that is not OpenShift, check the operator and the live pods:

```bash
oc get deployment openshift-gitops-operator-controller-manager \
  -n openshift-gitops-operator \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="manager")].image}{"\n"}'

oc get pods -n openshift-gitops \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

A pod that was already running keeps its old image until something recreates it. Delete that pod, or wait for a rollout that touches it.

On OpenShift, skip the operator pod. The agent image is the check that matters:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> \
  -o jsonpath='{.spec.customizedVariables[?(@.name=="ARGOCD_AGENT_IMAGE")].value}{"\n"}'
```

Try one cluster before you widen the `Placement`. Remove the annotation, confirm the source values return, then put the annotation back. After that, deploy an application through the agent. Image pulls and Git sync are separate: the pods can be running from the mirror while the cluster still cannot reach the Git server.

## Turn it off

Keep the object and drop the annotation when the same `ManagedClusterImageRegistry` is also used for the klusterlet:

```bash
oc annotate managedclusterimageregistry gitops-addon-mirror -n openshift-gitops \
  apps.open-cluster-management.io/gitops-addon-image-mirroring-
```

Otherwise delete it:

```bash
oc delete managedclusterimageregistry gitops-addon-mirror -n openshift-gitops
```

The controller writes the recorded original values back, and the bookkeeping annotations disappear. Give it a reconcile. If the mirror values are still there after a few minutes, check that the `Placement` named by the object can still be resolved. An already-running pod is left as it is until it is recreated.

## The agent image follows the hub

There is no separate agent-image field to maintain. On every managed cluster where the agent is enabled, the hub keeps `ARGOCD_AGENT_IMAGE` equal to the image the hub principal is running. The OpenShift GitOps operator on the hub owns that principal image. After an upgrade, the new image is what the managed clusters receive. Where you have opted into mirroring, that image is rewritten with the rest.

## When the images are right and the agent still cannot connect

These three failures show up while bringing up the Argo CD agent pull model on a disconnected managed cluster. The first two come from hub assumptions that break when the cluster has no path to the hub's public endpoints. The third is a hub TLS setting that produces the same authentication failure.

### The Route hostname does not resolve

The hub chooses the principal address in this order: Route, then LoadBalancer, then NodePort. A Route counts as found as soon as the object exists. The hub does not check that the managed cluster can resolve it. The agent then retries with `name resolver error: produced zero addresses`.

A LoadBalancer the managed cluster can reach is a valid address, and it saves you the node lookup below. Where you have no such load balancer, switch the principal Service to NodePort and tell `GitOpsCluster` a hub node IP and that port. The cluster was imported through that node, so the path depends on IP and port reachability. The hub adds its node IPs to the principal certificate when no LoadBalancer ingress is present, so this switch does not add a certificate name mismatch on top.

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p \
  '{"spec":{"argoCDAgent":{"principal":{"server":{"service":{"type":"NodePort"}}}}}}'

oc get svc openshift-gitops-agent-principal -n openshift-gitops \
  -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'

oc patch gitopscluster <gitopscluster-name> -n openshift-gitops --type=merge -p \
  '{"spec":{"gitopsAddon":{"argoCDAgent":{"serverAddress":"<hub-node-ip>","serverPort":"<nodePort>"}}}}'
```

On the managed cluster, `acm-openshift-gitops` should then show `principalServerAddress` and `principalServerPort` from that patch, and the agent log should be free of the resolver error.

### The certificate in the Secret is valid, and the agent still says it expired

The agent checks the principal's server certificate, `argocd-agent-principal-tls`. A renewal writes the new certificate into the Secret. The principal process keeps serving the certificate it loaded at start.

Read the Secret first:

```bash
oc get secret argocd-agent-principal-tls -n openshift-gitops \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates -subject
```

If `notAfter` is still in the past, the issuer has not renewed it yet. Fix the issuer before restarting anything. If the dates are current and the agent error remains, restart the principal so it loads the Secret:

```bash
oc rollout restart deployment -n openshift-gitops -l app.kubernetes.io/component=principal
oc rollout status deployment -n openshift-gitops -l app.kubernetes.io/component=principal
```

### The principal is serving a generated certificate

A different log line means the principal never loaded that Secret. On the hub:

```text
Using one-time generated TLS certificate for gRPC
```

That certificate is named `argocd-agent-principal--generated`. It is a short-lived, self-signed certificate upstream Argo CD agent creates for development, and it is served only when `ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE` is `true` on the GitOps operator. The default is `false`. The agent first reports an expired certificate. After a principal restart it reports that the certificate is valid for `argocd-agent-principal--generated` rather than the hostname it expected. Another restart generates another short-lived certificate with the same name.

Find the variable on the operator Subscription or the operator Deployment, and remove it:

```bash
oc get subscription -n openshift-gitops-operator -o yaml | grep -A3 ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE
oc get deployment -n openshift-gitops-operator -o yaml | grep -A3 ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE
```

Restart the principal. The log you want is `Loading gRPC TLS certificate from secret openshift-gitops/argocd-agent-principal-tls`. If the handshake still fails after that, delete `argocd-agent-principal-tls` and let the `GitOpsCluster` controller recreate it.

## Where to look

On the hub:

```bash
oc logs -n open-cluster-management multicluster-operators-application-<pod> \
  -c multicluster-operators-gitopscluster
oc logs -n openshift-gitops -l app.kubernetes.io/component=principal
```

On the managed cluster:

```bash
oc logs -n open-cluster-management-agent-addon gitops-addon-<pod>
oc logs -n openshift-gitops acm-openshift-gitops-agent-agent-<pod>
oc get argocd -n openshift-gitops acm-openshift-gitops -o yaml
```

## Further reading

- [GitOps with Advanced Cluster Management 5.0](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/gitops/index)
- [Applications](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/applications/index)
- [GitOps add-on image mirroring](https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md)
