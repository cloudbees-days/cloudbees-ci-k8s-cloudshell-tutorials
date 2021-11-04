# Install CloudBees CI on GKE

## Overview

In this tutorial you will create a GKE cluster and then install CloudBees CI on that cluster.

<walkthrough-tutorial-duration duration="100"></walkthrough-tutorial-duration>

## Create GKE Cluster

### GKE Cluster Availability Types

Having already decided to create a cluster using the Standard mode (instead of Autopilot), we must next decide what type of availability we want for our cluster. GKE supports zonal and regional availability for clusters. 

- A zonal cluster allows running nodes in single or multiple zones, but only provides a single replica of the control plane running in a single zone.
- A regional cluster allows running nodes in multiple zone, but also provides multiple replicas of the control plane running in multiple zones.

We will be creating a regional cluster to better simulate a production environment and learn some of the GCP/GKE specific features.

### Use the glcoud CLI to create a regional cluster

There are over 100 different flags available for the `clusters create` command and we are only specifying 10. Click on the <walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon> button for the `gloud` CLI command listed below to copy the command into your cloudshell console and then run that command.
```bsh
gcloud container --project "REPLACE_GCP_PROJECT" clusters create "REPLACE_GITHUB_USER" \
    --region "us-east1" \
    --node-locations "us-east1-b","us-east1-c" \
    --cluster-version "1.21.3-gke.2001" --release-channel "regular" \
    --machine-type "n1-standard-4" \
    --disk-type "pd-ssd" --disk-size "50" \
    --service-account "gke-nodes-for-workshop-testing@core-workshop.iam.gserviceaccount.com" \
    --enable-autoscaling --min-nodes "1" --max-nodes "4"
```

The flags we are setting are:
- `--region` - This is required to create a regional GKE cluster. To create a zonal cluster you would use the `--zone` flag. The value is set to `us-east1` to comply with CloudBees Ops rules.
- `--node-locations` - Specifies the zone(s) where the worker nodes will run. If not specified then they are spread across 3 random zones withing the cluster region (for regional clusters). We have specified two zones for use with GCP regional persistent disks that only support two zones.
- `--cluster-version` and `--release-channel` - We have specified this flag as we would like to use a non-default GKE version from the **regular** release channel. By using a version later than 1.21.0-gke.1500 we pick up a number of changes to the default values for flags to include using VPC-native as the default network mode.
- `--machine-type` - The default machine type is an **e2-medium** that has not been as stable for running CloudBees CI workloads as the **n1** machines have been.
- `--disk-type` and `--disk-size` - The defaults are too large (100gb) and too slow.
- `--service-account` - Required to pull the pre-release container images we are using for CloudBees CI from an Ops managed GCR.
- `--enable-autoscaling` - Autoscaling is not enabled by default and is very easy to enable on GKE via this flag. For AWS EKS you must manually configure and install the Kubernetes Cluster Autoscaler. 

Once the cluster creation completes, run the following command to see where your nodes are running:
```bsh
kubectl get nodes --label-columns failure-domain.beta.kubernetes.io/region,failure-domain.beta.kubernetes.io/zone
```

It should be no surprise that all the nodes are running in either the `us-east1-b` zone or the `us-east1-c` zone, as specified in the `clusters create` command.

Also, you may wonder how the `kubectl` CLI was able to interact with your cluster. When you run the `gcloud container cluster create` command from Cloud Shell, it automatically configure `kubeconfig` to connect to the cluster that was just created. Click <walkthrough-editor-open-file filePath="~/.kube/kubeconfig">kubeconfig</walkthrough-editor-open-file> to open your `kubeconfig` in the Cloud Shell editor.

### GKE Autoscaling Profiles

Autoscaling profiles allow you to specify the utilization of available resources for a GKE cluster. The default autoscaling profile is `balanced` and optimizes for minimizing provisioning time to include taking longer to deprovision nodes that no longer have pods that can be evicted. The `optimize-utilization` profile configures the cluster autoscaler to scale down the cluster more aggressively: it can remove more nodes, and remove nodes faster. It also configures GKE to prefer to schedule Pods in nodes that already have high utilization, helping the cluster autoscaler to identify and remove underutilized nodes. This profile will result in better results with the CloudBees CI hibernation feature.

Even though we already create the GKE cluster, we can still update it. Run the following command to update your GKE cluster to use the `optimize-utilization` profile:
```bsh
gcloud container clusters "REPLACE_GITHUB_USER" \
    --autoscaling-profile optimize-utilization
```

>NOTE: CloudBees CI managed controllers are configured with meta-data that does not allow them to be evicted from a node that may otherwise be able to be removed by the Cluster Autoscaler if moved to another node with capacity.


## Install Supporting Kubernetes Services

There are two supporting Kubernetes services that we will install before installing CloudBees CI to exposes HTTP and HTTPS routes (Ingresses) from outside the cluster to CloudBees CI services (controllers) within the cluster and to provide dynamic TLS for those Ingresses.

>NOTE: Openshift actually refers to these as a `Route` resource instead of an `Ingress` resource.

### Ingress

CloudBees requires that you use the Nginx Ingress controller to provide routes or Ingresses to the CloudBees CI controllers running as Kubernetes services. 

We will use `helm` to install the Nginx Ingress controller.

First, we need to add the **ingress-nginx** helm repo (chart) so it is available to install:
```bsh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

```
Next we will update all of our local charts to ensure we have the latest version:
```bsh
helm repo update
```
Finally, we can install the ingress-nginx chart:
```bsh
helm upgrade --install --wait ingress-nginx ingress-nginx/ingress-nginx \
    -n ingress-nginx --create-namespace
```
- `upgrade --install`: if a release by this name doesn't already exist, runs an install. By using `upgrade --install`, we are able to use the same command to install as we would for updating the application. This is especially useful in the context of software automation.
- `--wait`: will wait until all Pods, PVCs, Services, and minimum number of Pods of a Deployment, StatefulSet, or ReplicaSet are in a ready state before marking the release as successful.
- `--create-namespace`: will create the Kubernetes namespace if it doesn't already exist.

### TLS for HTTPS

Of course we want web traffic to our CloudBees CI cluster to be secure. However, creating, managing and updating TLS certificates is a lot of work. That is why we will use the cert-manager Kubernetes add-on to automatically issue an X.509 certificate from Lets Encrypt (a non-profit, free, automated, and open certificate authority (CA)) and store it as a Kubernetes **Secret** in your GKE cluster.

We will use `helm` to install the cert-manager add-on:
```bsh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace \
  --version v1.5.4 \
  --set global.leaderElection.namespace=cert-manager  --set prometheus.enabled=false \
  --set installCRDs=true --wait
```
- `--set`: The various `--set` parameters are used to override the default values of different variables in the helm chart. You may also pass those values as a yaml file with the `-f` parameters as we will see with CloudBees CI. The `installCRDs=true` value installs the cert-manager **CustomResourceDefinitions** (Kubernetes objects that allow you to extend Kubernetes with custom features) that we will interact with next.

#### Create a cert-manager `Issuer` for CloudBees CI

A cert-manager `Issuer` is a Kubernetes resource that represents a certificate authority (CA) that is able to generate signed certificates by honoring certificate signing requests (CSRs). cert-manager automatically creates CSRs when you install an application that is configured to use a cert-manager provide certificate. cert-manager supports `ClusterIssuers` that work across all namespaces in a cluster and `Issuers` that can only issue certificates in the namespace they are created. We will be creating a `ClusterIssuer` to give us the flexibility of deploying managed controller, agents and other supporting applications, that require TLS, to different namespaces.

Before we create our `ClusterIssuer` let's explore the contents by clicking <walkthrough-editor-open-file filePath="k8s/cluster-issuers.yml">`k8s/cluster-issuers.yml`</walkthrough-editor-open-file>.

Now, use the `kubectl` CLI to install the `letsencrypt-staging` and `letsencrypt-prod` `ClusterIssuers`:
```bsh
kubectl apply -f ./k8s/cluster-issuers.yml
```

## Create Regional Persistent Disk Storage Class

When creating a GKE cluster, several `StorageClasses` (cluster wide resources) are automatically created for you. But we have create a regional cluster, across two zones. If we used one of the provided `StorageClasses` (`SC`) then a managed controller would not be able to failover to the other zone the persistent disk is zone specific. So we will need to create a custom `StorageClass` that uses GCP regional persistent disks.

First, lets take a look at the manifest for our custom `StorageClass`. Click <walkthrough-editor-open-file filePath="k8s/regional-pd-ssd.yml">`k8s/regional-pd-ssd.yml`</walkthrough-editor-open-file>.

Some things to note are:
- `provisioner`: the `SC` is using the `pd.csi.storage.gke.io` provisioner. The GKE containers storage interface driver is required for using regional persistent disk.
- `parameters/type`: the type is `pd-ssd` which is backed by fast SSD persistent disks, and faster disk results in better performance for CloudBees CI.
- `parameters/replication-type`: `regional-pd` must be specified here to use regional persistent disk.
- `allowedTopologies`: the `values` for the `topology.gke.io/zone` `key` must be set to match the zones where we deployed the GKE cluster nodes.

Use `kubectl` to install the `regional-pd-ssd` `SC` into your cluster:
```bsh
kubectl apply -f k8s/regional-pd-ssd.yml
```

## Configure DNS for CloudBees CI



```bsh
PROJECT_ID=core-workshop
DNS_ZONE=workshop-cb-sa
#get ingress-nginx lb ip
INGRESS_IP=$(kubectl get services -n ingress-nginx | grep LoadBalancer  | awk '{print $4}')
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io

gcloud dns --project=$PROJECT_ID record-sets transaction start --zone=$DNS_ZONE
gcloud dns --project=$PROJECT_ID record-sets transaction add $INGRESS_IP --name=$CBCI_HOSTNAME. --ttl=300 --type=A --zone=$DNS_ZONE
gcloud dns --project=$PROJECT_ID record-sets transaction execute --zone=$DNS_ZONE
```

## Install CloudBees CI

As mentioned earlier, we will be using a file to specify the chart values to override for our installation of CloudBees CI (and the `--set` parameter, but more about that ahead). Before we run the `helm` command to install CloudBees CI, let's take a look at the <walkthrough-editor-open-file filePath="helm/cbci-values.yml">values file</walkthrough-editor-open-file>. Some things to note include:
- `dockerImage`: on line 35 notice how the `dockerImage` is coming from a GCP Container Registry. And more specifically, from a CloudBees Ops registry that is not public and requires a Google IAM service account that has been provided access - in this case, the `gke-nodes-for-workshop-testing@core-workshop.iam.gserviceaccount.com` that we specified the `gcloud` command to create our GKE clusters.
- `CasC`: line 38; it is `enabled` and the `ConfigMapName` is set to `oc-casc-bundle`.
- `Protocol`: line 61, it is set to `https` - thank you cert-manager.
- `JavaOpts`: line 89, 
    - controller provisioning has been configure to delete persistent storage when a managed controller is deleted
    - the Jenkins setup wizard has been disable
    - the `ManagePermission` and `SystemReadPermission` permissions have been enabled (without a plugin).
- 


