# External Secrets Operator with AWS Secrets Manager

This repository demonstrates how to set up and use the External Secrets Operator (ESO) to sync secrets from AWS Secrets Manager to a Kubernetes cluster. The setup is tested on a local KIND (Kubernetes in Docker) cluster.

## Overview

The External Secrets Operator allows you to integrate external secret management systems like AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, and others with Kubernetes. This example shows how to:

- Install External Secrets Operator using Helm
- Configure AWS Secrets Manager as a secret provider
- Sync secrets from AWS to Kubernetes
- Use the synced secrets in your applications

## Prerequisites

- Local Kubernetes cluster (KIND recommended)
- Helm 3.x installed
- AWS CLI configured with appropriate credentials
- kubectl configured to access your cluster
- AWS account with Secrets Manager access

## Installation

### 1. Install External Secrets Operator

Add the External Secrets Operator Helm repository and install:

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace
```

### 2. Verify Installation

Check that all Custom Resource Definitions (CRDs) are created:

```bash
kubectl get crds | grep external-secrets.io
```

You should see output similar to:
```
acraccesstokens.generators.external-secrets.io          2025-08-30T10:31:30Z
clusterexternalsecrets.external-secrets.io              2025-08-30T10:31:30Z
clustergenerators.generators.external-secrets.io        2025-08-30T10:31:30Z
clusterpushsecrets.external-secrets.io                  2025-08-30T10:31:30Z
clustersecretstores.external-secrets.io                 2025-08-30T10:31:30Z
externalsecrets.external-secrets.io                     2025-08-30T10:31:30Z
secretstores.external-secrets.io                        2025-08-30T10:31:30Z
# ... and more
```

## AWS Configuration

### 1. Create IAM Policy

Create an IAM policy with least privilege access to AWS Secrets Manager:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "GetSecretValue",
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "arn:aws:secretsmanager:*:442042543385:secret:*"
        },
        {
            "Sid": "ListSecrets",
            "Effect": "Allow",
            "Action": "secretsmanager:ListSecrets",
            "Resource": "*"
        }
    ]
}
```

### 2. Create IAM User and Attach Policy

1. Create an IAM user in AWS Console
2. Attach the above policy to the user
3. Generate access keys for the user

## Kubernetes Configuration

### 1. Create AWS Credentials Secret

Store your AWS credentials in a Kubernetes secret:

```bash
kubectl create secret generic aws-creds \
  --from-literal=access-key-id=YOUR_ACCESS_KEY_ID \
  --from-literal=secret-access-key=YOUR_SECRET_ACCESS_KEY
```

### 2. Create SecretStore

Create a `secret-store.yaml` file to configure the AWS provider:

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: secretstore-aws
spec:
  refreshInterval: 600 # refresh interval to sync secrets & use pull based method
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1 
      auth:
        secretRef: # local k8s authentication to aws secrets manager ( use only for local dev & there is no IRSA )
          accessKeyIDSecretRef:
            name: aws-creds # reference to secret containing aws creds & must create this secret first
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-creds # reference to secret containing aws creds & must create this secret first
            key: secret-access-key
```

Apply the configuration:

```bash
kubectl apply -f secret-store.yaml
```

Verify the SecretStore is ready:

```bash
kubectl get secretstore
```

Expected output:
```
NAME              AGE   STATUS   CAPABILITIES   READY
secretstore-aws   8s    Valid    ReadWrite      True
```

### 3. Create ExternalSecret

Create an `external-secret.yaml` file to define which secrets to sync:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: aws-external-secret
spec:
  refreshInterval: 600s
  secretStoreRef:
    name: secretstore-aws
    kind: SecretStore
  target:
    name: mongodb-secret # secret name to be created in k8s
    creationPolicy: Owner
  data:
  - secretKey: user # key in the k8s secret
    remoteRef:
      key: mongodb-credentials # secret name used in the external secret store
      property: username # key name used in the external secret store
  - secretKey: password # key in the k8s secret
    remoteRef:
      key: mongodb-credentials 
      property: password # key name used in the external secret store
  - secretKey: database # key in the k8s secret
    remoteRef:
      key: mongodb-credentials 
      property: database # key name used in the external secret store
```

Apply the configuration:

```bash
kubectl apply -f external-secret.yaml
```

Verify the ExternalSecret is synced:

```bash
kubectl get externalsecret
```

Expected output:
```
NAME                  STORETYPE     STORE             REFRESH INTERVAL   STATUS         READY
aws-external-secret   SecretStore   secretstore-aws   600s               SecretSynced   True
```

### 4. Verify Secret Creation

Check that the Kubernetes secret was created:

```bash
kubectl get secret mongodb-secret
```

## Example Application

Create a `mongo.yaml` file to deploy MongoDB using the synced secret:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongo-db
  labels:
    app: mongodb
spec:
  containers:
  - name: mongodb
    image: mongo:latest
    ports:
    - containerPort: 27017
    env:
    - name: MONGO_INITDB_ROOT_USERNAME
      valueFrom:
        secretKeyRef:
          name: mongodb-secret
          key: user
    - name: MONGO_INITDB_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mongodb-secret
          key: password
    - name: MONGO_INITDB_DATABASE
      valueFrom:
        secretKeyRef:
          name: mongodb-secret
          key: database
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-database
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP
```

Deploy the application:

```bash
kubectl apply -f mongo.yaml
```

Verify the deployment:

```bash
kubectl get all
```

Expected output:
```
NAME           READY   STATUS    RESTARTS   AGE
pod/mongo-db   1/1     Running   0          12s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
service/kubernetes       ClusterIP   10.132.0.1      <none>        443/TCP     10d
service/mongo-database   ClusterIP   10.132.13.184   <none>        27017/TCP   12s
```

## File Structure

```
.
├── README.md
├── secret-store.yaml
├── external-secret.yaml
├── mongo.yaml
└── aws-policy.json
```

## Troubleshooting

### Common Issues

1. **SecretStore not ready**: Check AWS credentials and IAM permissions
2. **ExternalSecret sync failed**: Verify the secret exists in AWS Secrets Manager
3. **Application can't access secret**: Ensure the secret name matches in both ExternalSecret and application manifests

### Debugging Commands

```bash
# Check SecretStore status
kubectl describe secretstore secretstore-aws

# Check ExternalSecret status
kubectl describe externalsecret aws-external-secret

# View secret contents (base64 encoded)
kubectl get secret mongodb-secret -o yaml

# Check External Secrets Operator logs
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets
```

## Security Considerations

- Use least privilege IAM policies
- Rotate AWS access keys regularly
- Consider using IAM roles with service accounts (IRSA) for EKS clusters
- Monitor secret access in AWS CloudTrail
- Use different AWS accounts for different environments

## References

- [External Secrets Operator Documentation](https://external-secrets.io/)
- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/)
- [KIND Documentation](https://kind.sigs.k8s.io/)
- [Helm Documentation](https://helm.sh/docs/)

## Contributing

Feel free to submit issues and enhancement requests!

## License

This project is licensed under the MIT License - see the LICENSE file for details.