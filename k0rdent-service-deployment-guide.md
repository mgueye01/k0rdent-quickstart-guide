# k0rdent Service Deployment Guide

This guide provides step-by-step instructions for deploying service templates to your k0rdent-managed clusters. You'll learn how to deploy common services like Nginx Ingress, Cert-Manager, Velero, and Kyverno to your AWS and Azure clusters.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Understanding Service Templates](#understanding-service-templates)
- [Deploying Common Services](#deploying-common-services)
  - [Nginx Ingress Controller](#nginx-ingress-controller)
  - [Cert-Manager](#cert-manager)
  - [Velero (AWS Only)](#velero-aws-only)
  - [Kyverno (Azure Only)](#kyverno-azure-only)
- [Multi-Cluster Service Deployment](#multi-cluster-service-deployment)
- [Verifying Service Deployments](#verifying-service-deployments)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before starting, ensure you have:

- A working k0rdent management cluster
- At least one AWS and one Azure cluster deployed using k0rdent
- Access to the kubeconfig files for all clusters
- The kubectl command-line tool installed

## Understanding Service Templates

k0rdent uses ServiceTemplate objects to define services that can be deployed to managed clusters. These templates are pre-configured with best practices and can be customized to meet your specific requirements.

To list available service templates:

```bash
# Switch to the management cluster
export KUBECONFIG=./KUBECONFIG

# List available service templates
kubectl get servicetemplates -n kcm-system
```

You should see output similar to:

```
NAME                      VALID
cert-manager-1-16-2       true
dex-0-19-1                true
external-secrets-0-11-0   true
ingress-nginx-4-11-0      true
ingress-nginx-4-11-3      true
kyverno-3-2-6             true
velero-8-1-0              true
```

## Deploying Common Services

In k0rdent, services are deployed by adding them to the ClusterDeployment object's `serviceSpec` section. This approach allows you to manage both the cluster and its services in a unified way.

### Nginx Ingress Controller

The Nginx Ingress Controller provides an HTTP and HTTPS load balancer for Kubernetes services.

#### Deploy to AWS Cluster

To add Nginx Ingress to your AWS cluster, update your ClusterDeployment with the following:

```bash
# Get your current AWS ClusterDeployment
kubectl get clusterdeployment aws-cluster -n kcm-system -o yaml > aws-cluster.yaml
```

Edit the aws-cluster.yaml file to add the serviceSpec section:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/aws-load-balancer-type: nlb
                service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    priority: 100
```

Apply the updated ClusterDeployment:

```bash
kubectl apply -f aws-cluster.yaml
```

#### Deploy to Azure Cluster

Similarly, update your Azure ClusterDeployment:

```bash
# Get your current Azure ClusterDeployment
kubectl get clusterdeployment azure-cluster -n kcm-system -o yaml > azure-cluster.yaml
```

Edit the azure-cluster.yaml file to add the serviceSpec section:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /healthz
    priority: 100
```

Apply the updated ClusterDeployment:

```bash
kubectl apply -f azure-cluster.yaml
```

### Cert-Manager

Cert-Manager is a Kubernetes add-on to automate the management and issuance of TLS certificates.

#### Deploy to AWS Cluster

Update your AWS ClusterDeployment to add Cert-Manager:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/aws-load-balancer-type: nlb
                service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      - template: cert-manager-1-16-2
        name: cert-manager
        namespace: cert-manager
        values: |
          installCRDs: true
          prometheus:
            enabled: true
          webhook:
            timeoutSeconds: 30
    priority: 100
```

Apply the updated ClusterDeployment:

```bash
kubectl apply -f aws-cluster.yaml
```

#### Deploy to Azure Cluster

Update your Azure ClusterDeployment to add Cert-Manager:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /healthz
      - template: cert-manager-1-16-2
        name: cert-manager
        namespace: cert-manager
        values: |
          installCRDs: true
          prometheus:
            enabled: true
          webhook:
            timeoutSeconds: 30
    priority: 100
```

Apply the updated ClusterDeployment:

```bash
kubectl apply -f azure-cluster.yaml
```

### Velero (AWS Only)

Velero is a backup and disaster recovery solution for Kubernetes clusters. In this guide, we'll deploy it only to the AWS cluster.

#### Create AWS S3 Bucket for Backups

Before deploying Velero, you need to create an S3 bucket for storing backups:

```bash
# Set your AWS region and bucket name
export AWS_REGION=us-east-2
export BUCKET_NAME=k0rdent-velero-backups-$(date +%s)

# Create the S3 bucket
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION
```

#### Create IAM User for Velero

Create an IAM user with permissions to access the S3 bucket:

```bash
# Create IAM policy for Velero
cat > velero-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET_NAME}"
            ]
        }
    ]
}
EOF

# Create the policy
aws iam create-policy \
  --policy-name velero-policy \
  --policy-document file://velero-policy.json

# Create IAM user for Velero
aws iam create-user --user-name velero

# Attach the policy to the user
aws iam attach-user-policy \
  --user-name velero \
  --policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`velero-policy`].Arn' --output text)

# Create access key for the user
aws iam create-access-key --user-name velero
```

Save the `AccessKeyId` and `SecretAccessKey` from the output.

#### Create Velero Credentials Secret

Create a file named `velero-credentials.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: velero-aws-credentials
  namespace: kcm-system
type: Opaque
stringData:
  cloud: |
    [default]
    aws_access_key_id=<YOUR-VELERO-ACCESS-KEY-ID>
    aws_secret_access_key=<YOUR-VELERO-SECRET-ACCESS-KEY>
```

Replace `<YOUR-VELERO-ACCESS-KEY-ID>` and `<YOUR-VELERO-SECRET-ACCESS-KEY>` with the values from the previous step.

Apply the secret:

```bash
kubectl apply -f velero-credentials.yaml
```

#### Deploy Velero to AWS Cluster

Update your AWS ClusterDeployment to add Velero:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: aws-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/aws-load-balancer-type: nlb
                service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      - template: cert-manager-1-16-2
        name: cert-manager
        namespace: cert-manager
        values: |
          installCRDs: true
          prometheus:
            enabled: true
          webhook:
            timeoutSeconds: 30
      - template: velero-8-1-0
        name: velero
        namespace: velero
        values: |
          initContainers:
            - name: velero-plugin-for-aws
              image: velero/velero-plugin-for-aws:v1.9.0
              volumeMounts:
                - mountPath: /target
                  name: plugins
          credentials:
            secretRef:
              name: velero-aws-credentials
              namespace: kcm-system
          configuration:
            provider: aws
            backupStorageLocation:
              bucket: <YOUR-BUCKET-NAME>
              config:
                region: <YOUR-AWS-REGION>
            volumeSnapshotLocation:
              config:
                region: <YOUR-AWS-REGION>
          deployRestic: true
    priority: 100
```

Replace `<YOUR-BUCKET-NAME>` with the S3 bucket name and `<YOUR-AWS-REGION>` with your AWS region.

Apply the updated ClusterDeployment:

```bash
kubectl apply -f aws-cluster.yaml
```

### Kyverno (Azure Only)

Kyverno is a policy engine designed for Kubernetes. In this guide, we'll deploy it only to the Azure cluster.

Update your Azure ClusterDeployment to add Kyverno:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: ClusterDeployment
metadata:
  name: azure-cluster
  namespace: kcm-system
spec:
  # Existing configuration...
  serviceSpec:
    services:
      - template: ingress-nginx-4-11-3
        name: ingress-nginx
        namespace: ingress-nginx
        values: |
          controller:
            service:
              annotations:
                service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path: /healthz
      - template: cert-manager-1-16-2
        name: cert-manager
        namespace: cert-manager
        values: |
          installCRDs: true
          prometheus:
            enabled: true
          webhook:
            timeoutSeconds: 30
      - template: kyverno-3-2-6
        name: kyverno
        namespace: kyverno
        values: |
          admissionController:
            replicas: 3
            resources:
              limits:
                cpu: 1000m
                memory: 512Mi
              requests:
                cpu: 100m
                memory: 128Mi
          backgroundController:
            replicas: 2
          cleanupController:
            resources:
              limits:
                cpu: 1000m
                memory: 512Mi
              requests:
                cpu: 100m
                memory: 128Mi
          reportsController:
            resources:
              limits:
                cpu: 1000m
                memory: 512Mi
              requests:
                cpu: 100m
                memory: 128Mi
    priority: 100
```

Apply the updated ClusterDeployment:

```bash
kubectl apply -f azure-cluster.yaml
```

## Multi-Cluster Service Deployment

For deploying the same service to multiple clusters, you'll need to update each ClusterDeployment individually. However, you can use the same configuration values for consistency.

For example, to deploy Nginx Ingress with the same base configuration to both AWS and Azure clusters, you would add it to both ClusterDeployment objects with the appropriate cloud-specific annotations.

## Verifying Service Deployments

### Check ClusterDeployment Status

To check the status of your service deployments, examine the ClusterDeployment status:

```bash
kubectl describe clusterdeployment aws-cluster -n kcm-system
kubectl describe clusterdeployment azure-cluster -n kcm-system
```

Look for the `.status.services` section which shows the status of each deployed service.

### Verify Services on AWS Cluster

```bash
# Switch to the AWS cluster
export KUBECONFIG=aws-cluster-kubeconfig.kubeconfig

# Check Nginx Ingress pods
kubectl get pods -n ingress-nginx

# Check Cert-Manager pods
kubectl get pods -n cert-manager

# Check Velero pods
kubectl get pods -n velero
```

### Verify Services on Azure Cluster

```bash
# Switch to the Azure cluster
export KUBECONFIG=azure-cluster-kubeconfig.kubeconfig

# Check Nginx Ingress pods
kubectl get pods -n ingress-nginx

# Check Cert-Manager pods
kubectl get pods -n cert-manager

# Check Kyverno pods
kubectl get pods -n kyverno
```

## Troubleshooting

### Common Issues

1. **Services not deploying**
   - Check the ClusterDeployment status: `kubectl describe clusterdeployment <name> -n kcm-system`
   - Look for errors in the `.status.services` section
   - Verify that the ServiceTemplate exists and is valid: `kubectl get servicetemplate <template-name> -n kcm-system`

2. **Service pods not starting**
   - Check the pods in the target namespace: `kubectl get pods -n <namespace> -o wide`
   - Check pod events: `kubectl describe pod <pod-name> -n <namespace>`
   - Check logs: `kubectl logs <pod-name> -n <namespace>`

3. **Configuration issues**
   - Verify that the values provided in the ClusterDeployment's serviceSpec are correct
   - Check if the Helm chart for the service accepts the configuration values you're providing

### Logs

To check k0rdent controller logs related to service deployments:

```bash
kubectl logs -n kcm-system -l app=kcm-controller-manager
```

---

This guide provides a comprehensive approach to deploying service templates to your k0rdent-managed clusters. For more detailed information about specific services, refer to their official documentation.
