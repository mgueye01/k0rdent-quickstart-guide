# k0rdent Production-Grade Cluster Deployment Quickstart Guide

This guide provides step-by-step instructions to deploy a production-grade k0rdent cluster. You'll set up a local kind cluster, deploy k0rdent on it, and then create two managed clusters - one on AWS and one on Azure.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1: Set Up Management Node and Cluster](#part-1-set-up-management-node-and-cluster)
- [Part 2: Configure AWS Provider](#part-2-configure-aws-provider)
- [Part 3: Configure Azure Provider](#part-3-configure-azure-provider)
- [Part 4: Deploy AWS Cluster](#part-4-deploy-aws-cluster)
- [Part 5: Deploy Azure Cluster](#part-5-deploy-azure-cluster)
- [Part 6: Accessing and Managing Your Clusters](#part-6-accessing-and-managing-your-clusters)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

## Prerequisites

Before starting, ensure you have:

- A Linux-based machine with at least 32GB RAM, 8 vCPUs, and 100GB SSD
- Administrative access to AWS and Azure accounts
- SSH access to your management node with passwordless sudo configured
- Inbound traffic: SSH (port 22) and ping from your laptop's IP address
- Outbound traffic: All to any IP address

## Part 1: Set Up Management Node and Cluster

### 1.1 Install Required Packages

```bash
# Update system packages
sudo apt-get update
sudo apt-get upgrade -y

# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg unzip
```

### 1.2 Install k0s Kubernetes

```bash
# Install k0s
curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start

# Verify k0s is running
sudo k0s kubectl get nodes
```

### 1.3 Install kubectl

```bash
# Add Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

### 1.4 Configure kubectl for k0s

```bash
# Get the kubeconfig from k0s
sudo cp /var/lib/k0s/pki/admin.conf KUBECONFIG
sudo chmod +r KUBECONFIG
export KUBECONFIG=./KUBECONFIG

# Verify kubectl works
kubectl get nodes
```

### 1.5 Install Helm

```bash
# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### 1.6 Install k0rdent

```bash
# Install k0rdent using Helm
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 0.1.0 -n kcm-system --create-namespace
```

### 1.7 Verify k0rdent Installation

Wait for all pods to be in the Running state:

```bash
# Check k0rdent pods
kubectl get pods -n kcm-system

# Check projectsveltos pods
kubectl get pods -n projectsveltos

# Verify k0rdent is ready
kubectl get Management -n kcm-system

# Verify provider templates are available
kubectl get providertemplate -n kcm-system

# Verify cluster templates are available
kubectl get clustertemplate -n kcm-system

# Verify service templates are available
kubectl get servicetemplate -n kcm-system
```

## Part 2: Configure AWS Provider

### 2.1 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 2.2 Install clusterawsadm

```bash
curl -LO https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/download/v2.7.1/clusterawsadm-linux-amd64
sudo install -o root -g root -m 0755 clusterawsadm-linux-amd64 /usr/local/bin/clusterawsadm
```

### 2.3 Configure AWS Credentials

```bash
# Export your AWS admin credentials
export AWS_REGION=<your-aws-region>
export AWS_ACCESS_KEY_ID=<your-access-key-id>
export AWS_SECRET_ACCESS_KEY=<your-secret-access-key>
# Optional if using MFA
export AWS_SESSION_TOKEN=<your-session-token>
```

### 2.4 Create k0rdent AWS User

```bash
# Create a dedicated user for k0rdent
aws iam create-user --user-name k0rdentQuickstart
```

Save the output, especially the ARN.

### 2.5 Configure AWS IAM for k0rdent

```bash
# Create CloudFormation stack with required IAM policies
clusterawsadm bootstrap iam create-cloudformation-stack
```

### 2.6 Attach IAM Policies to k0rdent User

Replace `<YOUR-ACCOUNT-ID>` with your AWS account ID:

```bash
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::<YOUR-ACCOUNT-ID>:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::<YOUR-ACCOUNT-ID>:policy/controllers-eks.cluster-api-provider-aws.sigs.k8s.io
aws iam attach-user-policy --user-name k0rdentQuickstart --policy-arn arn:aws:iam::<YOUR-ACCOUNT-ID>:policy/controllers.cluster-api-provider-aws.sigs.k8s.io
```

Verify policies were assigned:

```bash
aws iam list-policies --scope Local
```

### 2.7 Create AWS Credentials for k0rdent User

```bash
aws iam create-access-key --user-name k0rdentQuickstart
```

Save the `AccessKeyId` and `SecretAccessKey` securely.

### 2.8 Create AWS Credentials Secret in k0rdent

Create a file named `aws-cluster-identity-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-cluster-identity-secret
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
type: Opaque
stringData:
  AccessKeyID: "<YOUR-ACCESS-KEY-ID>"
  SecretAccessKey: "<YOUR-SECRET-ACCESS-KEY>"
```

Apply the secret:

```bash
kubectl apply -f aws-cluster-identity-secret.yaml -n kcm-system
```

### 2.9 Create AWSClusterStaticIdentity Object

Create a file named `aws-cluster-identity.yaml`:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSClusterStaticIdentity
metadata:
  name: aws-cluster-identity
  labels:
    k0rdent.mirantis.com/component: "kcm"
spec:
  secretRef: aws-cluster-identity-secret
  allowedNamespaces:
    selector:
      matchLabels: {}
```

Apply the configuration:

```bash
kubectl apply -f aws-cluster-identity.yaml -n kcm-system
```

### 2.10 Create k0rdent Credential Object for AWS

Create a file named `aws-cluster-identity-cred.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: aws-cluster-identity-cred
  namespace: kcm-system
spec:
  description: "AWS Credential for k0rdent"
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSClusterStaticIdentity
    name: aws-cluster-identity
```

Apply the credential:

```bash
kubectl apply -f aws-cluster-identity-cred.yaml -n kcm-system
```

### 2.11 Create Resource Template ConfigMap for AWS

Create a file named `aws-cluster-identity-resource-template.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-cluster-identity-resource-template
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
  annotations:
    projectsveltos.io/template: "true"
```

Apply the ConfigMap:

```bash
kubectl apply -f aws-cluster-identity-resource-template.yaml -n kcm-system
```

## Part 3: Configure Azure Provider

### 3.1 Install Azure CLI

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### 3.2 Log in to Azure

```bash
az login
```

### 3.3 Register Required Azure Resource Providers

```bash
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.ManagedIdentity
az provider register --namespace Microsoft.Authorization
```

### 3.4 Get Azure Subscription ID

```bash
az account list -o table
```

Note your Subscription ID from the output.

### 3.5 Create Azure Service Principal for k0rdent

Replace `<YOUR-SUBSCRIPTION-ID>` with your Azure Subscription ID:

```bash
az ad sp create-for-rbac --role contributor --scopes="/subscriptions/<YOUR-SUBSCRIPTION-ID>"
```

Save the output, especially `appId`, `password`, and `tenant`.

### 3.6 Create Azure Credentials Secret

Create a file named `azure-cluster-identity-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-cluster-identity-secret
  namespace: kcm-system
  labels:
    k0rdent.mirantis.com/component: "kcm"
stringData:
  clientSecret: "<YOUR-SP-PASSWORD>"
type: Opaque
```

Apply the secret:

```bash
kubectl apply -f azure-cluster-identity-secret.yaml
```

### 3.7 Create AzureClusterIdentity Object

Create a file named `azure-cluster-identity.yaml`:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AzureClusterIdentity
metadata:
  name: azure-cluster-identity
  namespace: kcm-system
  labels:
    clusterctl.cluster.x-k8s.io/move-hierarchy: "true"
    k0rdent.mirantis.com/component: "kcm"
spec:
  allowedNamespaces: {}
  clientID: "<YOUR-SP-APP-ID>"
  clientSecret:
    name: azure-cluster-identity-secret
    namespace: kcm-system
  tenantID: "<YOUR-SP-TENANT>"
  type: ServicePrincipal
```

Apply the configuration:

```bash
kubectl apply -f azure-cluster-identity.yaml
```

### 3.8 Create k0rdent Credential Object for Azure

Create a file named `azure-cluster-identity-cred.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Credential
metadata:
  name: azure-cluster-identity-cred
  namespace: kcm-system
spec:
  identityRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AzureClusterIdentity
    name: azure-cluster-identity
    namespace: kcm-system
```

Apply the credential:

```bash
kubectl apply -f azure-cluster-identity-cred.yaml
```

## Part 4: Deploy AWS Cluster

### 4.1 Find Available AWS Regions

If you need to find available AWS regions:

```bash
aws ec2 describe-regions --output table
```

### 4.2 Create AWS ClusterDeployment

Create a file named `aws-cluster-deployment.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-cluster
  namespace: kcm-system
spec:
  template: aws-standalone-cp-0-1-0
  credential: aws-cluster-identity-cred
  config:
    clusterLabels: {}
    region: us-east-2  # Replace with your preferred region
    controlPlane:
      instanceType: t3.small
    worker:
      instanceType: t3.small
    controlPlaneNumber: 1  # For production, use 3
    workersNumber: 2
    publicIP: true
```

Apply the ClusterDeployment:

```bash
kubectl apply -f aws-cluster-deployment.yaml
```

### 4.3 Monitor AWS Cluster Deployment

```bash
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com aws-cluster --watch
```

Wait until the status shows `ClusterDeployment is ready`.

### 4.4 Get AWS Cluster Kubeconfig

```bash
kubectl -n kcm-system get secret aws-cluster-kubeconfig -o jsonpath='{.data.value}' | base64 -d > aws-cluster-kubeconfig.kubeconfig
```

### 4.5 Verify AWS Cluster

```bash
KUBECONFIG="aws-cluster-kubeconfig.kubeconfig" kubectl get nodes
```

## Part 5: Deploy Azure Cluster

### 5.1 Find Available Azure Locations

If you need to find available Azure locations:

```bash
az account list-locations -o table
```

### 5.2 Create Azure ClusterDeployment

Create a file named `azure-cluster-deployment.yaml`:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-cluster
  namespace: kcm-system
spec:
  template: azure-standalone-cp-0-1-0
  credential: azure-cluster-identity-cred
  config:
    clusterLabels: {}
    location: "eastus"  # Replace with your preferred location
    subscriptionID: "<YOUR-SUBSCRIPTION-ID>"
    controlPlane:
      vmSize: Standard_A4_v2
    worker:
      vmSize: Standard_A4_v2
    controlPlaneNumber: 1  # For production, use 3
    workersNumber: 2
```

Apply the ClusterDeployment:

```bash
kubectl apply -f azure-cluster-deployment.yaml
```

### 5.3 Monitor Azure Cluster Deployment

```bash
kubectl -n kcm-system get clusterdeployment.k0rdent.mirantis.com azure-cluster --watch
```

Wait until the status shows `ClusterDeployment is ready`.

### 5.4 Get Azure Cluster Kubeconfig

```bash
kubectl -n kcm-system get secret azure-cluster-kubeconfig -o jsonpath='{.data.value}' | base64 -d > azure-cluster-kubeconfig.kubeconfig
```

### 5.5 Verify Azure Cluster

```bash
KUBECONFIG="azure-cluster-kubeconfig.kubeconfig" kubectl get nodes
```

## Part 6: Accessing and Managing Your Clusters

### 6.1 List All Managed Clusters

```bash
kubectl get ClusterDeployments -A
```

### 6.2 Switch Between Clusters

To use the AWS cluster:

```bash
export KUBECONFIG=aws-cluster-kubeconfig.kubeconfig
```

To use the Azure cluster:

```bash
export KUBECONFIG=azure-cluster-kubeconfig.kubeconfig
```

To return to the management cluster:

```bash
export KUBECONFIG=./KUBECONFIG
```

### 6.3 Deploy Services to Managed Clusters

You can deploy services to your managed clusters using k0rdent's ServiceDeployment feature. For example, to deploy cert-manager:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ServiceDeployment
metadata:
  name: cert-manager-deployment
  namespace: kcm-system
spec:
  template: cert-manager-1-16-2
  clusterSelector:
    matchLabels:
      k0rdent.mirantis.com/clusterdeployment: "aws-cluster"
```

## Troubleshooting

### Common Issues

1. **Pods stuck in Pending state**
   - Check node resources: `kubectl describe node <node-name>`
   - Check pod events: `kubectl describe pod <pod-name> -n <namespace>`

2. **Cluster deployment fails**
   - Check ClusterDeployment status: `kubectl describe clusterdeployment <name> -n kcm-system`
   - Check CAPI cluster status: `kubectl describe cluster <name> -n kcm-system`
   - Check provider-specific resources (e.g., AWSCluster, AzureCluster)

3. **Authentication issues with cloud providers**
   - Verify credentials are correct in the secrets
   - Check IAM permissions for AWS or RBAC for Azure

### Logs

To check k0rdent controller logs:

```bash
kubectl logs -n kcm-system -l app=kcm-controller-manager
```

## Cleanup

### Delete AWS Cluster

```bash
kubectl delete ClusterDeployment aws-cluster -n kcm-system
```

### Delete Azure Cluster

```bash
kubectl delete ClusterDeployment azure-cluster -n kcm-system
```

### Uninstall k0rdent

```bash
kubectl delete management.k0rdent kcm
helm uninstall kcm -n kcm-system
kubectl delete ns kcm-system
kubectl delete ns projectsveltos
```

### Stop k0s

```bash
sudo k0s stop
sudo k0s reset
```

---

This quickstart guide provides a comprehensive approach to deploying a production-grade k0rdent environment. For more detailed information, refer to the [official k0rdent documentation](https://docs.k0rdent.io/v0.1.0/).
