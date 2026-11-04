# Mirror the ACM GitOps add-on for disconnected clusters

RHACM supports managing CNCF-compliant Kubernetes Clusters. In the following we focus on Gitops and especially Air-Gapped Clusters.

Use this feature when a **non-OpenShift Kubernetes cluster** cannot reach the registries that provide the ACM GitOps add-on images. ACM can rewrite the add-on's image locations to a mirror registry for the clusters selected by a `Placement`.

> **OpenShift is different.** For the GitOps operator, Argo CD, Redis, and Dex, use the normal OpenShift disconnected path: `CatalogSource` plus `ImageDigestMirrorSet` or `ImageContentSourcePolicy`. Use the ACM mirroring feature on OpenShift only when the **Argo CD agent image** needs a placement-specific mirror.

## At a glance

| If you run… | Mirror these images with… | Why |
| --- | --- | --- |
| Other Kubernetes distributions | `ManagedClusterImageRegistry` with the GitOps annotation | The add-on deploys the operator, Argo CD, Redis, Dex, and (when enabled) the Argo CD agent from its own configuration. |
| OpenShift | `CatalogSource` and `ImageDigestMirrorSet` / `ImageContentSourcePolicy` | OLM installs the operator, Argo CD, Redis, and Dex from the catalog. |
| OpenShift, agent image only | `ManagedClusterImageRegistry` with the GitOps annotation | ACM deploys the Argo CD agent, whose image follows the hub principal. |

## What this feature changes

The feature is opt-in. Add the following annotation to a `ManagedClusterImageRegistry`:

```yaml
apps.open-cluster-management.io/gitops-addon-image-mirroring: "true"
```

For selected clusters, ACM rewrites the GitOps add-on's configured image locations to the mirror. It does **not**:

- change clusters outside the referenced `Placement`;
- alter klusterlet mirroring or other uses of the same registry object; or
- populate the mirror or make the cluster's Git server reachable.

The add-on defaults remain the source of truth. When a hub upgrade changes an image default, the mirror rewrite follows it automatically. Removing the annotation or deleting the object restores the recorded source image values during the next reconcile.

## Prerequisites

Before configuring a mirror, make sure that:

1. A `GitOpsCluster` has already deployed the add-on. There is nothing to rewrite until its configuration exists.
2. You have a `Placement` that selects the intended clusters.
3. The mirror contains every image the add-on will request.
4. The mirror credentials are available through the pull-secret chain distributed by the add-on.
5. For agent-based GitOps, the managed cluster can reach the hub principal and the application Git server. Image mirroring only solves image pulls.

## Configure a mirror for a fleet

The examples use the `openshift-gitops` namespace. Adjust it if your GitOps installation uses a different namespace.

### 1. Create the mirror pull secret

```bash
oc create secret docker-registry mirror-registry-pull-secret \
  -n openshift-gitops \
  --docker-server=my-mirror.example.com:5000 \
  --docker-username=<user> \
  --docker-password=<token>
```

### 2. Select the disconnected clusters

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

### 3. Create the registry mapping

`spec.registries` contains source-to-mirror mappings. Entries are processed in order; an entry matches when `source` is empty or matches the image's registry host, and the **last matching entry wins**. Put the catch-all first, then more-specific registries.

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

For example, `registry.redhat.io/openshift-gitops-1/gitops-rhel8` becomes `my-mirror.example.com:5000/redhat/openshift-gitops-1/gitops-rhel8`. Any unlisted host uses the empty-source catch-all.

If every image uses one mirror, use the simpler `spec.registry` instead. It is ignored if `spec.registries` is set.

```yaml
spec:
  registry: my-mirror.example.com:5000
```

## Verify the configuration

### On the hub

A finalizer shows that ACM has started managing the registry object. The add-on configuration in each selected cluster's namespace should contain mirrored values and annotations that preserve the originals.

```bash
oc get managedclusterimageregistry gitops-addon-mirror -n openshift-gitops \
  -o jsonpath='{.metadata.finalizers}{"\\n"}'

oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> -o yaml

oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> \
  -o jsonpath='{range .spec.customizedVariables[*]}{.name}={.value}{"\\n"}{end}'
```

### On a non-OpenShift managed cluster

Confirm the operator and the running GitOps pods use the mirror location.

```bash
oc get deployment openshift-gitops-operator-controller-manager \
  -n openshift-gitops-operator \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="manager")].image}{"\\n"}'

oc get pods -n openshift-gitops \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\\t"}{range .spec.containers[*]}{.image}{" "}{end}{"\\n"}{end}'
```

Existing pods keep their current image until they are recreated. Wait for a rollout or recreate the relevant pod.

### On an OpenShift managed cluster

Do not use the operator image to validate this feature; it is normally installed and mirrored by OLM. Check the agent image instead:

```bash
oc get addondeploymentconfig gitops-addon-config -n <managed-cluster-name> \
  -o jsonpath='{.spec.customizedVariables[?(@.name=="ARGOCD_AGENT_IMAGE")].value}{"\\n"}'
```

Start with a single cluster, confirm that the source values return after removing the annotation, then widen the `Placement`.

## How ACM applies the mirror

On the hub, the `GitOpsCluster` controller writes the default image values into each managed cluster's add-on configuration, including the image of the hub principal. When the annotation is present, the mirroring controller rewrites matching values and records their originals. The add-on framework delivers that configuration; the managed cluster then installs from the delivered values.

The `ARGOCD_AGENT_IMAGE` needs no separate configuration. It follows the image used by the hub principal and is rewritten with the other images on opted-in clusters after an upgrade.

## Remove the mirror

If the same `ManagedClusterImageRegistry` also mirrors the klusterlet, keep the object and remove only the GitOps annotation:

```bash
oc annotate managedclusterimageregistry gitops-addon-mirror -n openshift-gitops \
  apps.open-cluster-management.io/gitops-addon-image-mirroring-
```

Otherwise, delete the object:

```bash
oc delete managedclusterimageregistry gitops-addon-mirror -n openshift-gitops
```

ACM restores the recorded image values and removes its bookkeeping annotations on the next reconcile. If values remain mirrored after a few minutes, verify that the referenced `Placement` still resolves. Running pods are not changed until recreated.

## Troubleshoot an agent that cannot reach the hub

If the images are correct but the Argo CD agent cannot connect, use the symptom that matches the logs.

### `name resolver error: produced zero addresses`

The hub chooses the principal address in this order: **Route, LoadBalancer, then NodePort**. A Route is selected when the object exists; ACM does not verify that the managed cluster can resolve it.

Use a reachable LoadBalancer when possible. Otherwise, expose the principal through NodePort and configure `GitOpsCluster` with a reachable hub node IP and port:

```bash
oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p \
  '{"spec":{"argoCDAgent":{"principal":{"server":{"service":{"type":"NodePort"}}}}}}'

oc get svc openshift-gitops-agent-principal -n openshift-gitops \
  -o jsonpath='{.spec.ports[0].nodePort}{"\\n"}'

oc patch gitopscluster <gitopscluster-name> -n openshift-gitops --type=merge -p \
  '{"spec":{"gitopsAddon":{"argoCDAgent":{"serverAddress":"<hub-node-ip>","serverPort":"<nodePort>"}}}}'
```

On the managed cluster, `acm-openshift-gitops` should show the configured `principalServerAddress` and `principalServerPort`.

### The agent reports an expired certificate

Check the certificate held in the principal TLS Secret before restarting anything:

```bash
oc get secret argocd-agent-principal-tls -n openshift-gitops \
  -o jsonpath='{.data.tls\\.crt}' | base64 -d | openssl x509 -noout -dates -subject
```

If `notAfter` is in the past, fix the issuer. If the Secret is current but the error remains, restart the principal so it reloads the certificate:

```bash
oc rollout restart deployment -n openshift-gitops -l app.kubernetes.io/component=principal
oc rollout status deployment -n openshift-gitops -l app.kubernetes.io/component=principal
```

### The principal is using a generated certificate

The hub log line below means the principal did not load `argocd-agent-principal-tls`:

```text
Using one-time generated TLS certificate for gRPC
```

This is a short-lived development certificate (`argocd-agent-principal--generated`), served when `ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE=true`. The default is `false`. Find and remove that variable from the GitOps operator Subscription or Deployment, then restart the principal.

```bash
oc get subscription -n openshift-gitops-operator -o yaml | grep -A3 ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE
oc get deployment -n openshift-gitops-operator -o yaml | grep -A3 ARGOCD_PRINCIPAL_TLS_SERVER_ALLOW_GENERATE
```

The expected principal log is:

```text
Loading gRPC TLS certificate from secret openshift-gitops/argocd-agent-principal-tls
```

If the TLS handshake still fails, delete `argocd-agent-principal-tls` and allow the `GitOpsCluster` controller to recreate it.

## Useful logs

On the hub:

```bash
oc logs -n open-cluster-management multicluster-operators-application-<pod> \
  -c multicluster-operators-gitopscluster
oc logs -n openshift-gitops -l app.kubernetes.io/component=principal
```

On a managed cluster:

```bash
oc logs -n open-cluster-management-agent-addon gitops-addon-<pod>
oc logs -n openshift-gitops acm-openshift-gitops-agent-agent-<pod>
oc get argocd -n openshift-gitops acm-openshift-gitops -o yaml
```

## Further reading

- [GitOps with Advanced Cluster Management 5.0](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/gitops/index)
- [Applications](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/5.0/html/applications/index)
- [GitOps add-on image mirroring reference](https://github.com/stolostron/multicloud-integrations/blob/main/docs/gitops-addon-disconnected-image-mirroring.md)
