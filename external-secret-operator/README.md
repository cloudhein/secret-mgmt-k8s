# External Secrets Operator with AWS Secrets Manager

This repository demonstrates how to set up and use the External Secrets Operator (ESO) to sync secrets from AWS Secrets Manager to a Kubernetes cluster. The setup is tested on a local KIND (Kubernetes in Docker) cluster.

# External Secret Operator Architecture

![ESO Diagram](./images/external-secret-operator.png)

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
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: secretstore-aws
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2  # Change to your AWS region
      auth:
        secretRef:
          accessKeyID:
            name: aws-creds
            key: access-key-id
          secretAccessKey:
            name: aws-creds
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
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: aws-external-secret
spec:
  refreshInterval: 600s  # Refresh every 10 minutes
  secretStoreRef:
    name: secretstore-aws
    kind: SecretStore
  target:
    name: mongodb-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: mongodb-credentials  # Name of secret in AWS Secrets Manager
      property: username
  - secretKey: password
    remoteRef:
      key: mongodb-credentials
      property: password
  - secretKey: database
    remoteRef:
      key: mongodb-credentials
      property: database
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

## Automatic Secret Rotation with Reloader

### The Problem with Secret Rotation

By default, Kubernetes Pods **do not automatically restart** when secrets are updated. Even though External Secrets Operator successfully syncs new credentials from AWS Secrets Manager to Kubernetes secrets, your applications will continue using the old credentials until the Pod is manually restarted.

### Solution: Install Reloader

Reloader is a Kubernetes controller that watches for changes in ConfigMaps and Secrets and automatically restarts associated Deployments, DaemonSets, and StatefulSets.

#### Install Reloader using Helm

```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update
helm install reloader stakater/reloader
```

Verify Reloader installation:

```bash
kubectl get pods | grep reloader
```

Expected output:
```
reloader-reloader-5c46c485f7-6z897   1/1     Running   0          3m29s
```

## Example Application with Automatic Secret Rotation

Create a `mongo-deployment.yaml` file to deploy MongoDB using a Deployment (required for Reloader):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deployment
  labels:
    app: mongo
  annotations:
    reloader.stakater.com/auto: "true"  # Auto-restart when any secret/configmap changes
    # OR use specific secret monitoring (more efficient):
    # secret.reloader.stakater.com/reload: "mongodb-secret"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo-container
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
        volumeMounts:
        - name: mongo-data
          mountPath: /bitnami/mongo
      volumes:
      - name: mongo-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-database
spec:
  selector:
    app: mongo
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP
```

Deploy the application:

```bash
kubectl apply -f mongo-deployment.yaml
```

Verify the deployment:

```bash
kubectl get all
```

Expected output:
```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/mongo-deployment-f5944ffd5-hzbj8     1/1     Running   0          8s
pod/reloader-reloader-5c46c485f7-6z897   1/1     Running   0          3m29s

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/kubernetes       ClusterIP   10.132.0.1       <none>        443/TCP     10d
service/mongo-database   ClusterIP   10.132.243.120   <none>        27017/TCP   8s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/mongo-deployment    1/1     1            1           8s
deployment.apps/reloader-reloader   1/1     1            1           3m29s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/mongo-deployment-f5944ffd5     1         1         1       8s
replicaset.apps/reloader-reloader-5c46c485f7   1         1         1       3m29s
```

## How Automatic Secret Rotation Works

The complete flow works as follows:

1. **AWS Secrets Manager**: You rotate/update credentials in AWS Secrets Manager
2. **External Secrets Operator**: Detects the change and updates the Kubernetes secret (within the refresh interval - 600s in this example)
3. **Reloader**: Detects the secret change and automatically restarts the deployment
4. **Application**: Starts with the new credentials from the updated secret

### Reloader Configuration Options

```yaml
# Option 1: Auto-reload on ANY secret or configmap change
annotations:
  reloader.stakater.com/auto: "true"

# Option 2: Monitor specific secret only (recommended for production)
annotations:
  secret.reloader.stakater.com/reload: "mongodb-secret"

# Option 3: Monitor multiple secrets
annotations:
  secret.reloader.stakater.com/reload: "mongodb-secret,another-secret"

# Option 4: Monitor configmaps
annotations:
  configmap.reloader.stakater.com/reload: "my-configmap"
```

### Testing Secret Rotation

1. **Update secret in AWS Secrets Manager**:
   - Go to AWS Console → Secrets Manager
   - Find your `mongodb-credentials` secret
   - Update any value (e.g., change password)

2. **Monitor the automatic rotation**:
   ```bash
   # Watch for secret changes
   kubectl get secret mongodb-secret -w
   
   # Watch for pod restarts
   kubectl get pods -l app=mongo -w
   
   # Check Reloader logs
   kubectl logs -f deployment/reloader-reloader
   ```

3. **Verify new credentials are loaded**:
   ```bash
   # Check the updated secret
   kubectl get secret mongodb-secret -o jsonpath='{.data.password}' | base64 -d
   
   # Verify pod environment variables
   kubectl exec deployment/mongo-deployment -- env | grep MONGO
   ```

The entire process is now fully automated - no manual intervention required for secret rotation!

## File Structure

```
.
├── README.md
├── secret-store.yaml
├── external-secret.yaml
├── mongo-deployment.yaml  # Updated to use Deployment with Reloader
└── aws-policy.json
```

## Troubleshooting

### Common Issues

1. **SecretStore not ready**: Check AWS credentials and IAM permissions
2. **ExternalSecret sync failed**: Verify the secret exists in AWS Secrets Manager
3. **Application can't access secret**: Ensure the secret name matches in both ExternalSecret and application manifests
4. **Pods not restarting on secret change**: Ensure you're using Deployment (not Pod) with proper Reloader annotations
5. **Reloader not working**: Check that Reloader is installed and the annotations are correct

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

# Check Reloader logs for restart events
kubectl logs -f deployment/reloader-reloader

# Watch secret changes in real-time
kubectl get secret mongodb-secret -w

# Watch pod restarts
kubectl get pods -l app=mongo -w
```

## Security Considerations

- Use least privilege IAM policies
- Rotate AWS access keys regularly
- Consider using IAM roles with service accounts (IRSA) for EKS clusters
- Monitor secret access in AWS CloudTrail
- Use different AWS accounts for different environments

## References

- [External Secrets Operator Documentation](https://external-secrets.io/)
- [Reloader Documentation](https://github.com/stakater/Reloader)
- [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/)
- [KIND Documentation](https://kind.sigs.k8s.io/)
- [Helm Documentation](https://helm.sh/docs/)

## Contributing

Feel free to submit issues and enhancement requests!

## License

This project is licensed under the MIT License - see the LICENSE file for details.