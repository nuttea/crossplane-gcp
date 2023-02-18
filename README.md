# Google Cloud Crossplane example

## Prerequisites 

- Kubernetes version v1.16.0 or later
- Helm version v3.2.0 or later
- Pre-defined Project, VPC, Subnets (Node, Pod, Service) for GKE

## Template of environement variables for Google Cloud

Example

```bash
# Set environement variables for Google Cloud
export GCP_PROJECT_ID="<YOUR_PROJECT_ID>"
export GCP_REGION="asia-southeast1"
export GCP_ZONE="asia-southeast1-b"
export GCP_PROJECT_NUMBER=$(gcloud projects describe $GCP_PROJECT_ID --format="value(projectNumber)")
export GKE_CLUSTER_NAME="crossplane-cluster-1"
export GKE_CLUSTER_ZONE="asia-southeast1-b"
export GKE_PRIVATE_CONTROLPLANE_CIDR="172.31.1.16/28"
export GKE_VPC="default"
export GKE_NODE_SUBNET="default"
export GKE_POD_SUBNET="pod-subnet"
export GKE_SERVICE_SUBNET="service-subnet"
export GCP_SERVICE_ACCOUNT_KEYFILE="crossplane-gcp-credentials.json"

# Set environment vars for Crossplane installation
export CROSSPLANE_VERSION="1.11.0"
export CROSSPLANE_NS="crossplane-system"
export PROVIDER_GCP="provider-gcp"
export PROVIDER_VERSION="0.26.0"
export CROSSPLANE_GCP_SA_NAME="crossplane-sa"
export CROSSPLANE_GCP_SERVICE_ACCOUNT="${CROSSPLANE_GCP_SA_NAME}@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
export CONTROLLER_CONFIG="gcp-config"
```

Get gke cluster credentials

```bash
gcloud container clusters get-credentials $GKE_CLUSTER_NAME --zone $GKE_CLUSTER_ZONE --project $GCP_PROJECT_ID
```

## Install Crossplane on GKE

Reference https://docs.crossplane.io/v1.11/software/install/

```bash
kubectl create namespace $CROSSPLANE_NS

helm repo add crossplane-stable https://charts.crossplane.io/stable && helm repo update

helm install crossplane \
--namespace $CROSSPLANE_NS \
--create-namespace crossplane-stable/crossplane \
--version $CROSSPLANE_VERSION
```

Check installed deployment

```bash
kubectl get deployments -n crossplane-system
```

## Install and configure Crossplane GCP Provider

[Crossplane GCP Provider github repo](https://github.com/crossplane-contrib/provider-gcp)

Authentication option - [Authenticating with Workload Identity](https://github.com/crossplane-contrib/provider-gcp)

Install provider-gcp

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: ${PROVIDER_GCP}
spec:
  package: xpkg.upbound.io/upbound/provider-gcp:v${PROVIDER_VERSION}
EOF
```

Configure service accounts to use as Crossplane Service Account

```bash
gcloud iam service-accounts create ${CROSSPLANE_GCP_SA_NAME} --project ${GCP_PROJECT_ID}
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member "serviceAccount:${CROSSPLANE_GCP_SERVICE_ACCOUNT}" \
  --role roles/container.admin \
  --project ${GCP_PROJECT_ID}
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member "serviceAccount:${CROSSPLANE_GCP_SERVICE_ACCOUNT}" \
  --role roles/compute.networkAdmin \
  --project ${GCP_PROJECT_ID}

gcloud iam service-accounts add-iam-policy-binding \
    ${GCP_PROJECT_NUMBER}-compute@developer.gserviceaccount.com \
    --member="serviceAccount:${CROSSPLANE_GCP_SERVICE_ACCOUNT}" \
    --role="roles/iam.serviceAccountUser"
```

Create a Service Account key

```bash
gcloud iam service-accounts keys create ${GCP_SERVICE_ACCOUNT_KEYFILE} \
  --iam-account=${CROSSPLANE_GCP_SERVICE_ACCOUNT}

kubectl create secret \
  generic gcp-secret \
  -n crossplane-system \
  --from-file=creds=./${GCP_SERVICE_ACCOUNT_KEYFILE}
```

Create a Provider Config

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $GCP_PROJECT_ID
  credentials:
    source: Secret
    secretRef:
      namespace: $CROSSPLANE_NS
      name: gcp-secret
      key: creds
EOF
```

## Provision a GKE Cluster

Example, deploy a Private GKE Cluster with a Spot Node Pool

```bash
cat <<EOF | kubectl apply -f -
---
# API Reference: https://marketplace.upbound.io/providers/upbound/provider-gcp/v0.27.0/resources/container.gcp.upbound.io/Cluster/v1beta1
apiVersion: container.gcp.upbound.io/v1beta1
kind: Cluster
metadata:
  name: gke-crossplane-cluster
spec:
  forProvider:
    releaseChannel:
      - channel: "STABLE"
    project: "${GCP_PROJECT_ID}"
    location: "${GCP_REGION}"
    network: "projects/${GCP_PROJECT_ID}/global/networks/${GKE_VPC}"
    subnetwork: "projects/${GCP_PROJECT_ID}/regions/${GCP_REGION}/subnetworks/${GKE_NODE_SUBNET}"
    removeDefaultNodePool: true
    initialNodeCount: 1
    workloadIdentityConfig:
      - workloadPool: "${GCP_PROJECT_ID}.svc.id.goog"
    ipAllocationPolicy:
      - clusterSecondaryRangeName: ${GKE_POD_SUBNET}
        servicesSecondaryRangeName: ${GKE_SERVICE_SUBNET}
    defaultMaxPodsPerNode: 64
    addonsConfig:
      - gcePersistentDiskCsiDriverConfig:
          - enabled: true
    networkPolicy:
      - enabled: true
    loggingConfig:
      - enableComponents:
        - SYSTEM_COMPONENTS
        - WORKLOADS
    monitoringConfig:
      - enableComponents:
        - SYSTEM_COMPONENTS
    privateClusterConfig:
      - enablePrivateNodes: true
        masterIpv4CidrBlock: "${GKE_PRIVATE_CONTROLPLANE_CIDR}"
    masterAuthorizedNetworksConfig:
      - cidrBlocks:
        - cidrBlock: "35.240.0.0/13"
          displayName: "Google IP 1"
        - cidrBlock: "34.128.0.0/10"
          displayName: "Google IP 2"
        
---
# API Reference: https://marketplace.upbound.io/providers/upbound/provider-gcp/v0.27.0/resources/container.gcp.upbound.io/NodePool/v1beta1
apiVersion: container.gcp.upbound.io/v1beta1
kind: NodePool
metadata:
  name: gke-crossplane-np-1
spec:
  forProvider:
    project: ${GCP_PROJECT_ID}
    nodeCount: 1
    nodeLocations:
      - "${GKE_CLUSTER_ZONE}"
    autoscaling:
      - maxNodeCount: 3
        minNodeCount: 1  
    clusterRef:
      name: gke-crossplane-cluster
    nodeConfig:
      - diskSizeGb: 100
        diskType: pd-balanced
        imageType: cos_containerd
        labels:
          test-label: crossplane-created
        machineType: e2-standard-4
        spot: true
        oauthScopes:
          - "https://www.googleapis.com/auth/devstorage.read_only"
          - "https://www.googleapis.com/auth/logging.write"
          - "https://www.googleapis.com/auth/monitoring"
          - "https://www.googleapis.com/auth/servicecontrol"
          - "https://www.googleapis.com/auth/service.management.readonly"
          - "https://www.googleapis.com/auth/trace.append"
    management:
      - autoRepair: true
        autoUpgrade: true
EOF
```

Example, add a second Node Pool to existing GKE Cluster

```bash
cat <<EOF | kubectl apply -f -
---
# API Reference: https://marketplace.upbound.io/providers/upbound/provider-gcp/v0.27.0/resources/container.gcp.upbound.io/NodePool/v1beta1
apiVersion: container.gcp.upbound.io/v1beta1
kind: NodePool
metadata:
  name: gke-crossplane-np-0
spec:
  forProvider:
    project: ${GCP_PROJECT_ID}
    nodeCount: 1
    nodeLocations:
      - "${GKE_CLUSTER_ZONE}"
    autoscaling:
      - maxNodeCount: 3
        minNodeCount: 1  
    clusterRef:
      name: gke-crossplane-cluster
    nodeConfig:
      - diskSizeGb: 100
        diskType: pd-balanced
        imageType: cos_containerd
        labels:
          test-label: crossplane-created
        machineType: e2-standard-4
        spot: true
        oauthScopes:
          - "https://www.googleapis.com/auth/devstorage.read_only"
          - "https://www.googleapis.com/auth/logging.write"
          - "https://www.googleapis.com/auth/monitoring"
          - "https://www.googleapis.com/auth/servicecontrol"
          - "https://www.googleapis.com/auth/service.management.readonly"
          - "https://www.googleapis.com/auth/trace.append"
    management:
      - autoRepair: true
        autoUpgrade: true
EOF
```
