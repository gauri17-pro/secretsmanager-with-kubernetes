# External Secrets Operator (ESO) - AWS Secrets Manager

Kubernetes manifests for syncing database credentials from AWS Secrets Manager into Kubernetes Secrets using the External Secrets Operator (ESO) with IRSA authentication.

## Prerequisites

- EKS cluster with OIDC provider configured
  
  ```
  eksctl create cluster --name my-cluster --region ap-south-1 --node-type t2.medium --version 1.35
  ```

- [External Secrets Operator](https://external-secrets.io/) installed in the cluster
  
  ```bash
  helm repo add external-secrets https://charts.external-secrets.io
  helm repo update
  helm install external-secrets external-secrets/external-secrets --namespace external-secrets --create-namespace --set installCRDs=true
  ```

- IAM role with the following permissions:
  
  ```json
  {
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
  }
  ```

- IAM role trust policy allowing the EKS OIDC provider

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

### 2. Update placeholders

In `service-account.yaml`, replace:
- `<ACCOUNT_ID>` — your AWS account ID
- `<ESO_IAM_ROLE_NAME>` — the IAM role name created for ESO

In `external-secret.yaml`, update:
- `namespace` — the namespace where your application runs
- `remoteRef.key` — if your secret name differs from `prod/db-credentials`

### 3. Apply the manifests

```bash
kubectl apply -f service-account.yaml
kubectl apply -f cluster-secret-store.yaml
kubectl apply -f external-secret.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f mongo-statefulset.yaml
```

### 4. Verify

```bash
# Check the ClusterSecretStore status
kubectl get clustersecretstore aws-secrets-manager

# Check the ExternalSecret sync status
kubectl get externalsecret -n app db-credentials

# Verify the Kubernetes Secret was created
kubectl get secret -n app db-credentials -o jsonpath='{.data.username}' | base64 -d
```

### 5. Verify MongoDB

```bash
# Check StatefulSet rollout
kubectl get statefulset -n app mongo

# Check all pods are running
kubectl get pods -n app -l app=mongo

# Initialize the replica set (run once after all pods are ready)
kubectl exec -n app mongo-0 -- mongosh -u admin -p <password> --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0.mongo.app.svc.cluster.local:27017" },
    { _id: 1, host: "mongo-1.mongo.app.svc.cluster.local:27017" },
    { _id: 2, host: "mongo-2.mongo.app.svc.cluster.local:27017" }
  ]
})'

# Check replica set status
kubectl exec -n app mongo-0 -- mongosh -u admin -p <password> --eval "rs.status()"
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
