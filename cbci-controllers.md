# CloudBees CI Managed Controllers

## Overview

In this tutorial you will learn how to provision managed controllers in controller specific, non-CJOC Kubernetes, `Namespaces` and how to use Kubernetes network policies to isolate different CloudBees CI `pods` for a mult-tenant environment. Kubernetes `Namespaces` allow you to partition a Kubernetes cluster based on non-Kubernetes specific groups - such as team, business unit, product, location, cost center, etc. 

The official Kubernetes documentation description of `Namespaces`:

"In Kubernetes, namespaces are the fundamental unit of organization of most objects. They also form the fundamental unit of isolation and security in Kubernetes. Most policies and isolation objects operate at the namespace level, such as RBAC roles, secrets, service accounts, resource quotas, and network policies."

However, before we start provisioning controllers in `Namespaces` we will create a GKE cluster and install CloudBees CI with Helm and **Kustomize** as a Helm post renderer.

Then we will review the roles and service accounts that are part of the standard CloudBees CI Helm install.

<walkthrough-tutorial-duration duration="100"></walkthrough-tutorial-duration>

## Create Cluster and Install CloudBees CI

If you already have a GKE cluster with CloudBees CI (and supporting tools) installed, then proceed to the next lesson.

If you are starting with this module then you will need to create a GKE cluster and install CloudBees CI (with supporting tools) with the following command:

```bsh
chmod +x install-cbci.sh
./install-cbci.sh
```

## Installing CloudBees CI with Kustomize

We saw in Module 1 how you can override certain parameters of a Helm chart by specifying them in a `values.yaml` file, as we have done in the <walkthrough-editor-open-file filePath="helm/cbci-values.yml">`k8s/helm/cbci-values.yml`</walkthrough-editor-open-file> file. However, there will almost always be certain configuration values you want to override, configuration you want to add and/or additional Kubernetes resources you will want to create as part of a Helm install. Luckily, Helm supports a concept of **post rendering**, allowing you "to use tools like **Kustomize** to apply configuration changes without the need to fork a public chart or requiring chart maintainers to specify every last configuration option for a piece of software." 

In addition to the ability to extend an existing Helm chart without forking or modifying the chart itself, the Helm `--post-renderer` also managed all additional Kubernetes resources as part of the same Helm `install`; so a `helm delete` will delete everything from the chart and everything added by Kustomize.

If you take a look at the <walkthrough-editor-open-file filePath="install-cbci.sh">`install-cbci.sh`</walkthrough-editor-open-file> script (line 60) we used to create your GKE cluster and install the CloudBees CI Helm chart (among other charts), you will see that we are using the <walkthrough-editor-open-file filePath="kustomize-wrapper.sh">`kustomize-wrapper.sh`</walkthrough-editor-open-file> script as a `helm` `--post-renderer`.

The configuration for the `kustomize-wrapper.sh` script is found in the <walkthrough-editor-open-file filePath="kustomization.yaml">`kustomization.yaml`</walkthrough-editor-open-file> file and includes:

- `configMapGenerator`: will generate the `cbci-oc-init-groovy` `configmap` containing the `init_groovy/09-license-activate.groovy` file and the `oc-casc-bundle` `configmap` that will contain all of the CJOC CasC bundle files.
- `resources`: will create additional Kubernetes resources to include:
  - the `letsencrypt-prod` `ClusterIssuer` for cert manager to provide a tls certificate for our CloudBees CI install.
  - the `regional-pd-ssd` `StorageClass` which is the `StorageClass` used by CJOC and managed controllers.
- `transformers`: will transform resources create by the `helm install` to add some `labels` to the CJOC and hibernation `pods` with the <walkthrough-editor-open-file filePath="transformers/pod-labels.yaml">`transformers/pod-labels.yaml`</walkthrough-editor-open-file> file. We will use this `label` with a Kubernetes network policy we will create in another lesson.
- `patches`: will patch resources create by the `helm install`. It allows us to add the GKE workload identity annotation to the `jenkins` `serviceaccount`.

Let's take a look at the <walkthrough-editor-open-file filePath="patches/jenkins-sa-patch.yaml">**patched**</walkthrough-editor-open-file> `jenkins` `serviceaccount`:

```bsh
kubectl get sa -n cbci jenkins -o yaml
```

Note the `annotation` for workload identity is included on the `jenkins` `serviceaccount` as part of the Hem `install`: 

```yaml
  annotations:
    iam.gke.io/gcp-service-account: core-cloud-run@core-workshop.iam.gserviceaccount.com
```

Next, navigation to your CloudBees CI Operations Center:

```bsh
echo https://REPLACE_GITHUB_USER.workshop.cb-sa.io/cjoc/
```

## Kubernetes Network Policies

"NetworkPolicies are an application-centric construct which allow you to specify how a pod is allowed to communicate with various network "entities" (we use the word "entity" here to avoid overloading the more common terms such as "endpoints" and "services", which have specific Kubernetes connotations) over the network." By default, pods are non-isolated and they will accept traffic from any source.

In a multi-tenant CloudBees CI environment on Kubernetes you may not want to allow:

- agent `pods` to be accessible from the internet.
- agent `pods` not able to access or be accessed by other agent `pods`, other managed controller `pods` and the CJOC `pod`.
- managed controller `pods` not able to access or be accessed by other managed controller `pods`.
- the CJOC `pod` must be able to access and be accessed by all managed controller `pods`.
- the CJOC, hibernation and managed controller `pods` must be accessible by the `ingress-nginx` `controller`.

But before we enable any specific network access we will disable all `Ingress` for the CloudBees CI `namespace` by applying the policy defined in the <walkthrough-editor-open-file filePath="k8s/network-policies/deny-all-ingress.yml">k8s/network-policies/deny-all-ingress.yml</walkthrough-editor-open-file> file:

```bsh
kubectl apply -f k8s/network-policies/deny-all-ingress.yml
echo https://REPLACE_GITHUB_USER.workshop.cb-sa.io/cjoc/
```

After running that command try to access your Operations Center in your browser and it should fail with a **504 Gateway Time-out** fron the `ingress-nginx` `controller`. That is because the `ingress-nginx` `controller` `pod` is denied **Ingress** to the CJOC `pod`.


So next, we will add a network policy that will allow the `ingress-nginx` `controller` to access (Ingress) any `pod` that is labelled with `networking/allow-internet-access: "true"` by applying the policy defined in the <walkthrough-editor-open-file filePath="k8s/network-policies/allow-ingress-nginx.yml">k8s/network-policies/allow-ingress-nginx.yml</walkthrough-editor-open-file> file (a `label` that we added to the CJOC and hibernation service `pods` with Kustomize):

```bsh
kubectl apply -f k8s/network-policies/allow-ingress-nginx.yml
echo https://REPLACE_GITHUB_USER.workshop.cb-sa.io/cjoc/
```

Now try to access your Operations Center in your browser and it should load as expected.

Next, navigate to the **Manage** screen for the **operations-ops** managed controller and we will see that CJOC thinks it is disconnected. We need to allow managed controllers in the CJOC `namespace` and any other managed controller `namespace` to be able to access the CJOC `pod` by applying the policy defined in the <walkthrough-editor-open-file filePath="k8s/network-policies/cjoc-controller-ingress.yml">k8s/network-policies/cjoc-controller-ingress.yml</walkthrough-editor-open-file> file:

```bsh
kubectl apply -f k8s/network-policies/cjoc-controller-ingress.yml
```

Finally, we need to allow CJOC to access the **operations-ops** managed controller `pod` by applying the policy defined in the <walkthrough-editor-open-file filePath="k8s/network-policies/controller-cjoc-ingress.yml">k8s/network-policies/controller-cjoc-ingress.yml</walkthrough-editor-open-file> file:

```bsh
kubectl apply -f k8s/network-policies/controller-cjoc-ingress.yml
```

## CloudBees CI RBAC

Before we provision a controller in its own non-CJOC `namespace` it will be useful to review the default Kubernetes RBAC configuration for a CloudBees CI install.

By default, with `rbac` and `hibernation` enabled, the CloudBees CI Helm chart creates four `ServiceAccounts` in your CloudBees CI `Namespace` (in addition to the `default` `ServiceAccount` that is automatically created for all `Namespaces`) that you can list with the following command:

```bsh
kubectl get --namespace cbci serviceaccounts
```
1. `cjoc`: the service account that the Operations Center `pod` runs as. It must have RBAC permissions to manage the lifecycle of managed controllers in Kubernetes.
2. `jenkins`: the service account that managed controller `pods` run as. It must have RBAC permissions to provision `pod` based agents.
3. `jenkins-agents`: 
4. `managed-master-hibernation-monitor`: the service account that the hibernation monitor service `pod` runs as. It must have permissions to scale the replicas of the managed controller `statefulsets` to 0 and scale them back up to 1.

The permissions for these service accounts are defined in 4 Kubernetes `roles`:

1. `cjoc-master-management`: bound to the `cjoc` service account with the `cjoc-role-binding` `RoleBinding`; it provides the most extensive set of permissions for any CloudBees CI component, to include: 
    - full CRUD for `statefulsets` so that CJOC is able to manage the entire life-cycle of managed controllers.
    - full CRUD for `persistentvolumeclaims` so that CJOC is able to manage the `persistentvolumeclaims` of managed controllers.
    - full CRUD for `ingresses` so that CJOC is able to manage `ingresses` for managed controllers.
2. `cjoc-agents`: bound to the `jenkins` service account with the `cjoc-master-role-binding` `RoleBinding`; it provides the necessary permissions to provision and interact with `pod` based agents.
3. `managed-master-hibernation-monitor`: bound to the `managed-master-hibernation-monitor` service account

There is also 1 `clusterrole` that is optionally enabled:

1. `cjoc-master-management-{{ .Release.Namespace }}`: bound to the `cjoc` service account; it provides the ability to select a `storageclass` from the managed controller config when creating a managed controller via the UI.

Let's take a look at the `cjoc-role-binding` `RoleBinding`:

```bsh
kubectl get -n cbci rolebinding cjoc-role-binding -o yaml
```
Note that there is no `namespace` specified for the `cjoc` `subjects` entry. When no `namespace` is specified for a `ServiceAccount` `subjects` entry, it will default to the `namespace` of the `RoleBinding` - in this case `cbci`.

Now let's take a look at the `cjoc-master-management` `role` being bound to the `cjoc` `serviceaccount`:

```bsh
kubectl get -n cbci role cjoc-master-management -o yaml
```

Next we will take a look at the `cjoc-master-role-binding`:

```bsh
kubectl get -n cbci rolebinding cjoc-master-role-binding -o yaml
```

Note that it is binding the `cjoc-agents` `role` to the `jenkins` `ServiceAccount` in the `cbci` `namespace`. So let's take a look at the `cjoc-agents` `role` and compare it to the `cjoc-master-management` `role` we looked at above:

```bsh
kubectl get -n cbci role cjoc-agents -o yaml
```

There are a lot fewer Kubernetes RBAC permissions required for managed controllers than there is for CJOC.

## Create a Managed Controller Namespace and RBAC Configuration with Helm

The CloudBees CI Helm chart provides a values parameters configuration that will create all the necessary Kubernetes objects for running a managed controller in its own Kubernetes `namespace`. Before we run the `helm` command, let's take a look at the  <walkthrough-editor-open-file filePath="helm/cbci-values.yml">helm/controller-values.yml</walkthrough-editor-open-file> file. Some key differences between the `controller-values.yml` and the `cbci-values.yml` include:

- `OperationsCenter.Enabled` is set to `false` because we don't want another Operations Center installed in this controller specific `namespace`. However, note that the `OperationsCenter.Ingress` matches the configuration from the `cbci-values.yml`. This is necessary when also enabling `Hibernation` for the controller specific `namespace` becaus the CloudBees CI Helm chart uses the `OperationsCenter.Ingress` value parameters for its own `ingress` configuration.
- `Master.OperationsCenterNamespace` needs to be set to the `namespace` that Operations Center was created in so the `rolebinding` for the `cjoc-master-management` role in the controller specific `namespace` is bound to the `cjoc` `serviceaccount` in the Operations Center `namespace`.
- `RBAC.installCluster` is commented out and defaults to `false`. This role is only useful for Operations Center to list available `storageclasses` in the Managed Controller provisioning UI, so there is no reason to enable it for a controller specific `namespace`.

```bsh
cd controller-config
chmod +x kustomize-wrapper.sh
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io
helm upgrade --install --wait controller-a cloudbees/cloudbees-core \
  --set OperationsCenter.HostName=$CBCI_HOSTNAME \
  --namespace='controller-a'  --create-namespace \
  --set OperationsCenter.Ingress.tls.Host=$CBCI_HOSTNAME \
  --values ./helm/controller-values.yml --post-renderer ./kustomize-wrapper.sh
kubectl label ns controller-a com.cloudbees.ci.type=controller
```

Once that completes, run the following command to see all the Kubernetes resources created in the `controller-a` `namespace`:

```bsh
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n controller-a
```

Now, we will create a new managed controller in the `controller-a` `namespace` using the CloudBees CI CasC HTTP API. But first we must create an API token for your CloudBees CI `admin` user:

1. Navigate to Operations Center, ensure that you are logged in as `admin`, and click the **Admin** link in the top right corner and then click **Configure**.
2. Click on the **Add new Token** button, name it `cbci-k8s-workshop`, click the **Generate** button and copy the value.
2. Back in the Google Cloud Shell paste the value of the API token to the following command:

```bsh
CJOC_ADMIN_API_TOKEN=
```

3. Run the following CasC HTTP API command to create a new controller:

```bsh
curl --user "admin:$CJOC_ADMIN_API_TOKEN" -XPOST \
              https://REPLACE_GITHUB_USER.workshop.cb-sa.io/cjoc/casc-items/create-items \
              --data-binary @./controller-a-item.yaml -H 'Content-Type:text/yaml'
```

## Clean Up

Run the following commands to clean everything up:

```bsh
helm uninstall controller-a -n controller-a 
kubectl delete ns controller-a
helm uninstall cbci -n cbci
kubectl delete ns cbci
gcloud container clusters delete REPLACE_GITHUB_USER --region=us-east1 --async

```

