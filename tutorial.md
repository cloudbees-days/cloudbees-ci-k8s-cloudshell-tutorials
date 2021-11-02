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

### GKE Autoscaling Profiles

Autoscaling profiles allow you to specify the utilization of available resources for a GKE cluster.

```bsh
gcloud container clusters "REPLACE_GITHUB_USER" \
    --autoscaling-profile optimize-utilization
```

### Configure Cluster Maintenance Window


```bsh
gcloud container clusters update "REPLACE_GITHUB_USER" \
    --maintenance-window-start "2020-08-10T04:00:00Z" \
    --maintenance-window-end "2020-08-11T04:00:00Z" \
    --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SA,SU"
```

## Create Regional Persistent Disk Storage Class

If you click on the link below you will see that `regional-pd-ssd-csi-storageclass` has been specified as the storage class to use for your CloudBees CI cluster.

<walkthrough-editor-open-file filePath="helm/cbci-values.yml">CBCI helm values</walkthrough-editor-open-file>