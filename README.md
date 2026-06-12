# External Secrets Operator (ESO) - AWS Secrets Manager

Kubernetes manifests for syncing database credentials from AWS Secrets Manager into Kubernetes Secrets using the External Secrets Operator (ESO) with IRSA authentication.

## Prerequisites

- Install kubectl
  Refer https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

- Install eksctl
  ```
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin
  ```

- Install helm
  ```
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
  chmod 700 get_helm.sh
  ./get_helm.sh
  ```

- Create an EKS Cluster
  ```
  eksctl create cluster --name my-cluster --region ap-south-1 --node-type t2.medium --version 1.35
  ```

- [External Secrets Operator](https://external-secrets.io/) installed in the cluster
  
  ```bash
  helm repo add external-secrets https://charts.external-secrets.io
  helm repo update
  helm install external-secrets external-secrets/external-secrets --namespace external-secrets --create-namespace --set installCRDs=true
  ```

- Install EBS CSI Controller
  ```
  aws iam attach-role-policy \
  --role-name <NodeInstanceRoleName> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
  ```

  ```
  kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.44"
  ```

- If using a KMS customer-managed key, add `kms:Decrypt` permission on the KMS key ARN

## Files

| File | Description |
|------|-------------|
| `service-account.yaml` | ServiceAccount annotated with the IAM role ARN for IRSA |
| `cluster-secret-store.yaml` | ClusterSecretStore connecting ESO to AWS Secrets Manager in `ap-south-1` |
| `external-secret.yaml` | ExternalSecret that syncs `prod/db-credentials` into a Kubernetes Secret |
| `mongo-service.yaml` | Headless Service for MongoDB StatefulSet DNS discovery |
| `mongo-statefulset.yaml` | MongoDB 3-node replica set StatefulSet using ESO-synced credentials |

## Setup

### 1. Create the secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
  --name prod/db-credentials \
  --region ap-south-1 \
  --secret-string '{"username":"admin","password":"s3cur3pass"}'
```

### 2. Enable OIDC provider for the EKS cluster

```bash
# Check if OIDC is already associated
aws eks describe-cluster --name my-cluster --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text

# If not, create the OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --region ap-south-1 \
  --approve
```

### 3. Create the IAM policy

```bash
aws iam create-policy \
  --policy-name ESO-SecretsManager-Policy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ],
        "Resource": "arn:aws:secretsmanager:ap-south-1:<ACCOUNT_ID>:secret:prod/db-credentials-*"
      }
    ]
  }'
```

### 4. Create the IAM role with trust policy

```bash
# Get your OIDC provider ID
OIDC_ID=$(aws eks describe-cluster --name my-cluster --region ap-south-1 \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)

# Create the role
aws iam create-role \
  --role-name ESO-SecretsManager-Role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/'$OIDC_ID'"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
          "StringEquals": {
            "oidc.eks.ap-south-1.amazonaws.com/id/'$OIDC_ID':sub": "system:serviceaccount:external-secrets:external-secrets-sa",
            "oidc.eks.ap-south-1.amazonaws.com/id/'$OIDC_ID':aud": "sts.amazonaws.com"
          }
        }
      }
    ]
  }'

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name ESO-SecretsManager-Role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ESO-SecretsManager-Policy
```

### 5. Update placeholders

In `service-account.yaml`, replace:
- `<ACCOUNT_ID>` — your AWS account ID
- `<ESO_IAM_ROLE_NAME>` — the IAM role name created for ESO

In `external-secret.yaml`, update:
- `namespace` — the namespace where your application runs
- `remoteRef.key` — if your secret name differs from `prod/db-credentials`

### 6. Apply the manifests

```bash
kubectl apply -f service-account.yaml
kubectl apply -f cluster-secret-store.yaml
kubectl apply -f external-secret.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-statefulset.yaml
```

### 7. Verify ESO

```bash
# Check the ClusterSecretStore status
kubectl get clustersecretstore aws-secrets-manager

# Check the ExternalSecret sync status
kubectl get externalsecret -n app db-credentials

# Verify the Kubernetes Secret was created
kubectl get secret -n app db-credentials -o jsonpath='{.data.username}' | base64 -d
```

### 8. Verify MongoDB

```bash
# Check StatefulSet rollout
kubectl get statefulset -n app mongo

# Check all pods are running
kubectl get pods -n app -l app=mongo

# Initialize the replica set (run once after all pods are ready)
kubectl exec -n app mongo-0 -- mongosh -u admin -p <password>

```

## Updating Secrets

Update the secret value in AWS Secrets Manager:

```bash
aws secretsmanager put-secret-value \
  --secret-id prod/db-credentials \
  --region ap-south-1 \
  --secret-string '{"username":"admin","password":"newpassword"}'
```

ESO will sync the updated value on the next refresh cycle (configured to `1h` in `external-secret.yaml`).
