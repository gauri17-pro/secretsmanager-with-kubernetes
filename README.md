# External Secrets Operator (ESO) - AWS Secrets Manager

Kubernetes manifests for syncing database credentials from AWS Secrets Manager into Kubernetes Secrets using the External Secrets Operator (ESO) with IRSA authentication.

## Prerequisites

- EKS cluster with OIDC provider configured
- [External Secrets Operator](https://external-secrets.io/) installed in the cluster
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

## Updating Secrets

Update the secret value in AWS Secrets Manager:

```bash
aws secretsmanager put-secret-value \
  --secret-id prod/db-credentials \
  --region ap-south-1 \
  --secret-string '{"username":"admin","password":"newpassword"}'
```

ESO will sync the updated value on the next refresh cycle (configured to `1h` in `external-secret.yaml`).
