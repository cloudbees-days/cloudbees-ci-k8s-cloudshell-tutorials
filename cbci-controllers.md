# CloudBees CI Managed Controllers

## Overview

In this tutorial you will learn how to provision managed controllers in controller specific, non-CJOC Kubernetes, `Namespaces`. Kubernetes `Namespaces` allow you to partition a Kubernetes cluster based on non-Kubernetes specific groups - such as team, business unit, product, location, cost center, etc. 

The official Kubernetes documentation description of `Namespaces`:

"In Kubernetes, namespaces are the fundamental unit of organization of most objects. They also form the fundamental unit of isolation and security in Kubernetes. Most policies and isolation objects operate at the namespace level, such as RBAC roles, secrets, service accounts, resource quotas, and network policies."

However, before we start provisioning controllers in `Namespaces` we review the roles and service accounts that are part of the standard CloudBees CI Helm install.

<walkthrough-tutorial-duration duration="100"></walkthrough-tutorial-duration>

## CloudBees CI RBAC

By default, with `rbac` and `hibernation` enabled, the CloudBees CI Helm chart creates four `ServiceAccounts` in your CloudBees CI `Namespace` (in addition to the `default` `ServiceAccount` that is automatically created for all `Namespaces`) that you can list with the following command:

```bsh
kubectl get --namespace cbci serviceaccounts
```



## Create a Managed Controller Namespace and RBAC Configuration with Helm

The CloudBees CI Helm chart provides a values parameters configuration that will create all the necessary Kubernetes objects for running a managed controller in its own Kubernetes `namespace`. Before we run the `helm` command, let's take a look at the  <walkthrough-editor-open-file filePath="helm/cbci-values.yml">helm/controller-values.yml</walkthrough-editor-open-file> file. Some key differences between the `controller-values.yml` and the `cbci-values.yml` include:

- `OperationsCenter.Enabled` is set to `false` because we don't want another Operations Center installed in this controller specific `namespace`. However, note that the `OperationsCenter.Ingress` matches the configuration from the `cbci-values.yml`. This is necessary when also enabling `Hibernation` for the controller specific `namespace` becaus the CloudBees CI Helm chart uses the `OperationsCenter.Ingress` value parameters for its own `ingress` configuration.
- `Master.OperationsCenterNamespace` needs to be set to the `namespace` that Operations Center was created in so the `rolebinding` for the `cjoc-master-management` role in the controller specific `namespace` is bound to the `cjoc` `serviceaccount` in the Operations Center `namespace`.
- `RBAC.installCluster` is commented out and defaults to `false`. This role is only useful for Operations Center to list available `storageclasses` in the Managed Controller provisioning UI, so there is no reason to enable it for a controller specific `namespace`.

```bsh
CBCI_HOSTNAME=REPLACE_GITHUB_USER.workshop.cb-sa.io
helm upgrade --install --wait controller-a cloudbees/cloudbees-core \
  --set OperationsCenter.HostName=$CBCI_HOSTNAME \
  --namespace='controller-a'  \
  --set OperationsCenter.Ingress.tls.Host=$CBCI_HOSTNAME \
  --values ./helm/controllers-values.yml
```

## Hierarchical Namespaces

Before we install a managed controller into its own, unique `namespace` we are going to install Hierarchical Namespaces. Because at the end of the day, we want all of our CloudBees CI cluster's managed controllers in their own, unique `namespaces`. And while manually configuring a Kubernetes `namespace` for one managed controller, it quickly becomes a manual configuration burden when you have dozens or hundreds of managed controllers.



## Kubernetes Network Policies



