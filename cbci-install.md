# Install CloudBees CI on GKE

## Overview

In this tutorial you will create a GKE cluster and then install CloudBees CI on that cluster.

<walkthrough-tutorial-duration duration="100"></walkthrough-tutorial-duration>

## Create GKE Cluster

### GKE Cluster Availability Types

Having already decided to create a cluster using the Standard mode (instead of Autopilot), we must next decide what type of availability we want for our cluster. GKE supports zonal and regional availability for clusters. 

- A zonal cluster allows running nodes in single or multiple zones, but only provides a single replica of the control plane running in a single zone.
- A regional cluster allows running nodes in multiple zones, but also provides multiple replicas of the control plane running in multiple zones.

We will be creating a regional cluster to better simulate a production environment while learning some of the GCP/GKE specific features we need to leverage for a more stable and secure CloudBees CI environment.

### Use the glcoud CLI to create a regional cluster

There are over 100 different flags available for the `gcloud` CLI `clusters create` command and we are only specifying 13. Click on the <walkthrough-cloud-shell-icon></walkthrough-cloud-shell-icon> button for the `gloud` CLI command listed below to copy the command into your cloudshell console and then run that command:
```bsh
gcloud config set project "REPLACE_GCP_PROJECT"
gcloud container clusters create "REPLACE_GITHUB_USER" \
    --region "us-east1" \
    --node-locations "us-east1-b","us-east1-c" \
    --num-nodes=1 \
    --cluster-version "1.21.5-gke.1302" --release-channel "regular" \
    --machine-type "n1-standard-4" \
    --disk-type "pd-ssd" --disk-size "50" \
    --service-account "gke-nodes-for-workshop-testing@core-workshop.iam.gserviceaccount.com" \
    --enable-autoscaling --min-nodes "0" --max-nodes "4" \
    --autoscaling-profile optimize-utilization \
    --enable-dataplane-v2 \
    --workload-pool "core-workshop.svc.id.goog"
```
>**NOTE:** GKE cluster creation may take as long as 5 minutes or more, so we will take that time to review all of the parameters that we set above.
The flags we are setting are:
- **`--region`** - This is required to create a regional GKE cluster. To create a zonal cluster you would use the `--zone` flag. The value is set to `us-east1` to comply with CloudBees Ops rules.
- **`--node-locations`** - Specifies the zone(s) where the worker nodes will run. If not specified then they are spread across 3 random zones within the cluster region (for regional clusters). We have specified two zones for use with GCP regional persistent disks, because GCP regional disks are only replicated across two zones.
- **`--num-nodes`** - The number of nodes we want initially in each of the cluster's zones. In this case we are defining two zones for the cluster nodes so there will be 2 total nodes across the entire cluster.
- **`--cluster-version` and `--release-channel`** - We have specified this flag as we would like to use a non-default GKE version from the **regular** release channel. By using a version later than `1.21.0-gke.1500`, we pick up a number of changes to the default values for flags to include using VPC-native as the default network mode.
- **`--machine-type`** - The default machine type is an **e2-medium** that has not been as stable for running CloudBees CI workloads as the **n1** machines have been.
- **`--disk-type` and `--disk-size`** - The defaults are too large (100gb) and slower that the one we are specifying.
- **`--service-account`** - Required to pull the pre-release container images we are using for CloudBees CI from an Ops managed GCR.
- **`--enable-autoscaling`** - Autoscaling is not enabled by default and is very easy to enable on GKE via this flag. For AWS EKS you must manually configure and install the Kubernetes Cluster Autoscaler. 
- **`--autoscaling-profile optimize-utilization`** - Autoscaling profiles allow you to specify the utilization of available resources for a GKE cluster. The default autoscaling profile is `balanced` and optimizes for minimizing provisioning time to include taking longer to deprovision nodes that no longer have pods that can be evicted. The `optimize-utilization` profile configures the cluster autoscaler to scale down the cluster more aggressively: it can remove more nodes, and remove nodes faster. It also configures GKE to prefer to schedule Pods in nodes that already have high utilization, helping the cluster autoscaler to identify and remove underutilized nodes. This profile will result in better performance with the CloudBees CI hibernation feature.
- **`--enable-dataplane-v2`:**  enables Dataplane V2 for your GKE cluster. Among other features, this results in automatic enablement of Kubernetes Network policies. 
- **`--workload-pool`:** enables Workload Identity for the initial node pool created for the cluster. We will learn more about Workload Identity in a later section.

>NOTE: CloudBees CI managed controllers are configured with meta-data that does not allow them to be evicted from a node that may otherwise be able to be removed by the Cluster Autoscaler if moved to another node with capacity.

[View the clusters in the GCP web console.](https://console.cloud.google.com/kubernetes/list/overview?project=core-workshop)

Once the cluster creation completes, run the following command to see where your nodes are running:
```bsh
kubectl get nodes --label-columns failure-domain.beta.kubernetes.io/region,failure-domain.beta.kubernetes.io/zone
```

It should be no surprise that all the nodes are running in either the `us-east1-b` zone or the `us-east1-c` zone, as specified in the `clusters create` command.

Also, you may wonder how the `kubectl` CLI was able to interact with your cluster. When you run the `gcloud container cluster create` command from Cloud Shell, it automatically configure `kubeconfig` to connect to the cluster that was just created. Run the following command to explore your `kubeconfig` file:
```bsh
more ~/.kube/config
```

## Install Supporting Kubernetes Services

There are two supporting Kubernetes services that we will install before installing CloudBees CI. One will be used to expose HTTP and HTTPS routes (Ingresses) from outside the cluster to CloudBees CI services (controllers) within the cluster and the other to provide dynamic TLS (HTTPS certificate) for those Ingresses.

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
- **`upgrade --install`:** if a release by this name doesn't already exist, runs an install. By using `upgrade --install`, we are able to use the same command to install as we would for updating the application. This is especially useful in the context of software automation.
- **`--wait`:** will wait until all Pods, PVCs, Services, and minimum number of Pods of a Deployment, StatefulSet, or ReplicaSet are in a ready state before marking the release as successful.
- **`--create-namespace`:** will create the Kubernetes namespace if it doesn't already exist.

Once the install is complete there should be a new Kubernetes `Service` of `LoadBalancer TYPE` in the newly created `ingress-nginx namespace`. Run the following command to check:
```bsh
kubectl get services -n ingress-nginx ingress-nginx-controller
```

Note that the `ingress-nginx-controller` has an `EXTERNAL-IP`.

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
- **`--set`:** The various `--set` parameters are used to override the default values of different variables in the helm chart. You may also pass those values as a yaml file with the `--values` or `-f` parameters as we will see with CloudBees CI. The **`installCRDs=true`** value installs the cert-manager **CustomResourceDefinitions** (Kubernetes objects that allow you to extend Kubernetes with custom features) that we will interact with next.

Run the following command to check the custom CRDs were installed and the cert-manager pods are running:
```bsh
kubectl get pods --namespace cert-manager
kubectl api-resources | grep cert-manager.io/v1 
```

#### Create a cert-manager `Issuer` for CloudBees CI

A cert-manager `Issuer` is a custom Kubernetes resource that represents a certificate authority (CA) that is able to generate signed certificates by honoring certificate signing requests (CSRs). cert-manager automatically creates CSRs when you install an application that is configured to use a cert-manager provided certificate. cert-manager supports `ClusterIssuers` that work across all namespaces in a cluster and `Issuers` that can only issue certificates in the namespace they are created. We will be creating a `ClusterIssuer` to give us the flexibility of deploying CJOC, managed controller, and other supporting applications, that require TLS, to different namespaces.

Before we create our `ClusterIssuer` let's explore the contents by clicking <walkthrough-editor-open-file filePath="k8s/cluster-issuers.yml">`k8s/cluster-issuers.yml`</walkthrough-editor-open-file>.

Now, use the `kubectl` CLI to install the `letsencrypt-staging` and `letsencrypt-prod` `ClusterIssuers`:
```bsh
kubectl apply -f ./k8s/cluster-issuers.yml
```

And the run the following command to check that they were created:
```bsh
kubectl get ClusterIssuers
```

## Create Regional Persistent Disk Storage Class

When creating a GKE cluster, several `StorageClasses` (cluster wide resources) are automatically created for you. We can get a list of those by running the following command with `sc` being shorthand for `StorageClasses`:
```bsh
kubectl get sc
```
Next, take deeper look at the provided `premium-rwo StorageClass` by running the following command:
```bsh
kubectl describe sc premium-rwo
```
Note, that the `Parameters` include `type=pd-ssd` which is the faster type of disk we want to use, but it is not regional.

We have created a regional cluster, across two zones. If we used one of the provided `StorageClasses` (`SC`), then a managed controller would not be able to failover to the other zone because a regular persistent disk is zone specific. So we will need to create a custom `StorageClass` that uses GCP regional persistent disks.

First, lets take a look at the manifest for our custom `StorageClass`. Click <walkthrough-editor-open-file filePath="k8s/regional-pd-ssd-sc.yml">`k8s/regional-pd-ssd-sc.yml`</walkthrough-editor-open-file>.

Some things to note are:
- **`provisioner`:** the `SC` is using the `pd.csi.storage.gke.io` provisioner. The GKE *containers storage interface* driver is required for using regional persistent disk.
- **`parameters/type`:** the type is `pd-ssd` which is backed by fast SSD persistent disks, and faster disk results in better performance for CloudBees CI.
- **`parameters/replication-type`:** `regional-pd` must be specified here to use regional persistent disk.
- **`volumeBindingMode: Immediate`:** the `Immediate` mode indicates that volume binding and dynamic provisioning occurs once the `PersistentVolumeClaim` is created. Whereas the `WaitForFirstConsumer` mode delays the binding and provisioning of a `PersistentVolume` until a `Pod` using the `PersistentVolumeClaim` is created. If we weren't using GCP regional persistent disk, we would have to use `WaitForFirstConsumer` mode to ensure the persistent disk is created in the same zone as the controller `Pod`.
- **`allowedTopologies`:** the `values` for the `topology.gke.io/zone` `key` must be set to match the zones where we deployed the GKE cluster nodes.

Use `kubectl` to install the `regional-pd-ssd` `SC` into your cluster:
```bsh
kubectl apply -f k8s/regional-pd-ssd-sc.yml
```

Finally, run the following command to make sure the new `StorageClass` was created:
```bsh
kubectl describe sc regional-pd-ssd
```
Note that the `Parameters` include `replication-type=regional-pd` and the `AllowedTopologies:` zones includes `[us-east1-b, us-east1-c]`; the same zones we specified for the worker nodes or our clusters.

## Configure DNS for CloudBees CI

We will use GCP DNS to create a new A record against the `workshop.cb-sa.io` domain, but first we must get the IP address of the load balancer created for the installed ingress-nginx controller. Run the following command to assign that IP to an environment variable and print it out:

```bsh
INGRESS_IP=$(kubectl get services -n ingress-nginx | grep LoadBalancer  | awk '{print $4}')
echo $INGRESS_IP
```
Next, we will set a few more environment variables and then use the `gcloud` CLI to add a new DNS record:
```bsh
PROJECT_ID=core-workshop
DNS_ZONE=workshop-cb-sa
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io

gcloud dns --project=$PROJECT_ID record-sets transaction start --zone=$DNS_ZONE
gcloud dns --project=$PROJECT_ID record-sets transaction add $INGRESS_IP --name=$CBCI_HOSTNAME. --ttl=300 --type=A --zone=$DNS_ZONE
gcloud dns --project=$PROJECT_ID record-sets transaction execute --zone=$DNS_ZONE
```

Finally, you can use the `ping` command to verify that your `CBCI_HOSTNAME` is being directed to the external IP address of your `ingress-nginx` controller:
```bsh
ping REPLACE_GITHUB_USER.workshop.cb-sa.io
```

## Install CloudBees CI

As mentioned earlier, we will be using a file to specify the chart values to override for our installation of CloudBees CI (and the `--set` parameter, but more about that ahead). 

You can generate a values yaml file for the CloudBees CI Helm chart by running the following command:
```bsh
helm show values cloudbees/cloudbees-core
```

But a let's take a look at the <walkthrough-editor-open-file filePath="helm/cbci-values.yml">values file</walkthrough-editor-open-file> that has already been created for you for this lab. Some things to note include:
- **`dockerImage`:** on line 35 notice how the `dockerImage` is coming from a GCP Container Registry. And more specifically, from a CloudBees Ops registry that is not public and requires a Google IAM service account that has been provided access - in this case, the `gke-nodes-for-workshop-testing@core-workshop.iam.gserviceaccount.com` that we specified in the `gcloud` command to create our GKE clusters has the necessary permissions.
- **`CasC`:** line 38; it is `enabled` and the `ConfigMapName` is set to `oc-casc-bundle`. We will create that `ConfigMap` Kubernetes resource below before we install CloudBees CI.
- **`Protocol`:** line 61, it is set to `https` - thank you cert-manager.
- **`JavaOpts`:** line 89 
    - controller provisioning has been configure to delete persistent storage when a managed controller is deleted
    - the Jenkins setup wizard has been disabled because we don't need it since we are using CasC
    - the `ManagePermission` and `SystemReadPermission` permissions have been enabled (without a plugin)
- **`Ingress`:** line 159
    - the `Class` is set to `nginx` to use the ingress-nginx controller we installed earlier
    - the `cert-manager.io/cluster-issuer` annotation is set to `letsencrypt-prod`; this will trigger the cert-manager add-on we install to create a TLS certificate for our CloudBees CI `Ingress`
    - under `tls`, `Enable` is set to `true` and note the `SecretName` of `cbci-tls` - this will be the name of the Kubernetes `Secret` that the cert-manager creates with the TLS certificate it retrieves from Let's Encrypt
- **`ExtraVolumes`:** line 201, specifies the Kubernetes `ConfigMap` `cbci-oc-init-groovy` to be mounted to the Operations Center `Pod` that we will create next before we install CloudBees CI.
- **`ExtraVolumeMounts`:** line 212, specified where to mount the `ExtraVolumes`
- **`StorageClass`:** line 259, the name specified here, `regional-pd-ssd`, matches the `StorageClass` we created earlier with regional ssd persistent disk

Before we use `helm` to install CloudBees CI, we need to create the `cbci-oc-init-groovy` `ConfigMap` resource with the contents of <walkthrough-editor-open-file filePath="init_groovy/09-license-activate.groovy">init_groovy/09-license-activate.groovy</walkthrough-editor-open-file> that will automatically create a trial license for our CloudBees CI installation:
```bsh
kubectl create ns cbci
kubectl -n cbci create configmap cbci-oc-init-groovy --from-file=init_groovy/ --dry-run=client -o yaml | kubectl apply -f -
```
The command above may look a bit more complicated than it needs to, so let's take a look at it:
- **`create ns`:** we need to create the `ConfigMap` in the same name space as CloudBees CI
- **`--from-file=init_groovy/`:** this parameter allows you to create a `ConfigMap` from one file or a directory of files. By pointing at a directory, we can add additional init groovy scripts and they will be added to the same `ConfigMap`
- **`--dry-run==client`:** We are using this so we can pipe the output to a `kubectl apply` command. `kubectl create` is imperative and will fail if an object with the same name and in the same namespace already exist, but the `apply` command is declarative and just tells Kubernetes the state we want it to be in - so if the `ConfigMap` already exists it will just modify it to match the new one being applied. This is especially useful with automation.

We also need to create a `ConfigMap` resource for the Operations Center CasC bundle named `oc-casc-bundle`. In this case, it will be made up of multiple files from your `casc/oc/` directory:
```bsh
kubectl -n cbci create configmap oc-casc-bundle --from-file=casc/oc --dry-run=client -o yaml | kubectl apply -f -
```

Now that we have created the `ConfigMap` we are now ready to use `helm` to install CloudBees CI:
```bsh
helm repo add cloudbees https://charts.cloudbees.com/public/cloudbees
helm repo update
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io
helm upgrade --install --wait cbci cloudbees/cloudbees-core \
  --set OperationsCenter.HostName=$CBCI_HOSTNAME \
  --namespace='cbci'  --create-namespace \
  --set OperationsCenter.Ingress.tls.Host=$CBCI_HOSTNAME \
  --values ./helm/cbci-values.yml
``` 
- First, we add the `cloudbees` helm charts and then update them. 
- We set the `CBCI_HOSTNAME` environment variable to use in the `helm` command to install CloudBees CI.
- Finally, we use the `helm upgrade` command with the `--install` flag. Also note, that we are use a combination of `--set` parameters along with the `--values` parameter to override default chart values for our install.

Run the following command to check that the `cbci-oc-init-groovy` `ConfigMap` was created in the `/var/jenkins_config/init.groovy.d/` directory of the `cjoc` container:
```bsh
kubectl -n cbci exec cjoc-0 -- more /var/jenkins_config/init.groovy.d/09-license-activate.groovy
```

The Operations Center for your CloudBees CI cluster should be running or starting up, and available at: [http://REPLACE_GITHUB_USER.workshop.cb-sa.io]/cjoc](http://REPLACE_GITHUB_USER.workshop.cb-sa.io]/cjoc])

Don't **Create First Admin User**. In the next section we will update the OC CasC bundle to create a user for us and retrieve the [password from the GCP Secrets Manager](https://console.cloud.google.com/security/secret-manager/secret/cbci-oc-admin-password/versions?project=core-workshop).

## Setting Up Workload Identity

Workload Identity for GKE allows us to bind GCP IAM Service Accounts (GSA) to Kubernetes Service Accounts (KSA) in a specific Kubernetes Namespace. This allows us to integrate other GCP services with Kubernetes services, such as CloudBees CI. 

### Configure Workload Identity
We will now configure the `cjoc serviceAccount` (the `serviceAccount` that the `cjoc pod` is running under) in the `cbci namespace` to be bound to the `core-cloud-run@REPLACE_GCP_PROJECT.iam.gserviceaccount.com` GCP IAM service account - which has been given the necessary permissions to retrieve the secrets we need from the GCP secrets manager. First we will create an IAM policy binding between the Kubernetes `cjoc ServiceAccount` and the GCP IAM service account by running the following command:
```bsh
gcloud iam service-accounts add-iam-policy-binding core-cloud-run@REPLACE_GCP_PROJECT.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:REPLACE_GCP_PROJECT.svc.id.goog[cbci/cjoc]"
```
Next, we will add an annotation to the `cjoc ServiceAccount` by running the following command:
```bsh
kubectl annotate serviceaccount \
    --namespace cbci cjoc \
    iam.gke.io/gcp-service-account=core-cloud-run@REPLACE_GCP_PROJECT.iam.gserviceaccount.com
```

## Integrating GCP Secrets Manager with CloudBees CI CasC

Sometimes you need to integrate other GCP services with CloudBees CI controllers and/or agents. We need to integrate with the GCP Secrets Manager so we can inject those secrets into our CloudBees CI CasC bundles. Workload Identity for GKE allows us to bind GCP IAM Service Accounts (GSA) to Kubernetes Service Accounts (KSA) in a specific Kubernetes Namespace. In addition to configuring Workload Identity for our GKE clusters, we will also need to install the Secrets Store CSI driver for Kubernetes secrets and the GCP provider that integrates the GCP Secrets Manager.

### Install Secrets Store CSI driver

We will use `helm` to install the Secrets Store CSI driver for Kubernetes secrets:
```bsh
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
```
Next, will install the Google Secret Manager provider for the Secret Store CSI Driver:
```bsh
git clone https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp.git
kubectl apply -f secrets-store-csi-driver-provider-gcp/deploy/provider-gcp-plugin.yaml
```

Now that we have Secrets Store CSI driver and the Google Secret Manager provider installed we can <walkthrough-editor-open-file filePath="k8s/cbci-cjoc-secret-provider.yml">create a `SecretProviderClass`</walkthrough-editor-open-file> (another CRD, provided by the Secret Store CSI Driver) to use with the `cjoc Pod`:
```bsh
kubectl apply -f ./k8s/cbci-cjoc-secret-provider.yml
```

In order to inject the `cbciCjocAdminPassword` secret into the `cjoc Pod` and use it in the `jenkinsyaml` of the OC CasC bundle we must:
- update the JCasC `jenkins.yaml` to use the secret and update the `oc-casc-bundle ConfigMap`
- add a JCasC environment variable that tells JCasC where to find secret variables to replace
- mount the secret to the `cjoc Pod`
- use `helm` to update the CloudBees CI application to get the new environment variable, the mount and the updated `ConfigMap`

First, open the <walkthrough-editor-open-file filePath="casc/oc/jenkins.yaml">casc/oc/jenkins.yaml</walkthrough-editor-open-file> file. Replace the entire `jenkins` section with the following:
```yml
jenkins:
  authorizationStrategy: "cloudBeesRoleBasedAccessControl"
  securityRealm:
    local:
      allowsSignup: false
      enableCaptcha: false
      users:
       - id: admin
         password: "${cbciCjocAdminPassword}"
```
Notice that the `password` value `cbciCjocAdminPassword` must match the mapping we created in the `SecretProviderClass` we created above. Next, run the following command to update the `oc-casc-bundle ConfigMap`:
```bsh
kubectl -n cbci create configmap oc-casc-bundle --from-file=casc/oc --dry-run=client -o yaml | kubectl apply -f -
```

Now we will update the `helm` values for the CloudBees CI application. Open the <walkthrough-editor-open-file filePath="helm/cbci-values.yml">helm/cbci-values.yml</walkthrough-editor-open-file> file. 
- At around line 83 insert the following `ContainerEnv` section to add the necessary JCasC environment variable:
```yml

  ContainerEnv:
  - name: "SECRETS"
    value: "/var/jenkins_home/jcasc_secrets"
```
- At around line 208 add the `cbci-cjoc-secret-provider SecretProviderClass` as an additional `ExtraVolumes`:
```yml

  - name: "jcasc-secrets"
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "cbci-cjoc-secret-provider"
```
- At around line 224 add the `ExtraVolumeMounts` for that volume and note that the `mountPath` matches the ` SECRETS` environment variable `value`:
```yml

  - name: "jcasc-secrets"
    mountPath: "/var/jenkins_home/jcasc_secrets"
    readOnly: true
```
- Finally, we will use `helm` to update the `cbci` application:
```bsh
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io
helm upgrade --install --wait cbci cloudbees/cloudbees-core \
  --set OperationsCenter.HostName=$CBCI_HOSTNAME \
  --namespace='cbci'  --create-namespace \
  --set OperationsCenter.Ingress.tls.Host=$CBCI_HOSTNAME \
  --values ./helm/cbci-values.yml
```
