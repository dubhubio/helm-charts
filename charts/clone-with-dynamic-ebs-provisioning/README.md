# DubHub Clone Helm Chart

Deploy a DubHub clone on Kubernetes to start a database clone from a DubHub snapshot image.

This chart:
- Pulls a Docker clone from ECR
- Runs the clone container and exposes the database port (default: 5432)
- Optionally runs post-startup SQL if required (e.g., create DBs/schemas/users)

## Prerequisites

- Kubernetes 1.12+
- Helm 3.0+
- Access to the ECR that hosts the Docker clone image created with Baseshift (see [Configuring Image Pull Secret for AWS ECR](#configuring-image-pull-secret-for-aws-ecr))

#### Configuring Image Pull Secret for AWS ECR

The `imagePullSecret` required by this Helm chart is not a regular AWS Secret but a Kubernetes secret that allows Kubernetes to pull images from ECR. This secret is generated using credentials obtained from AWS and is used by Kubernetes to authenticate with ECR. 

To create an `imagePullSecret` for your Kubernetes cluster that allows pulling images from AWS ECR, follow these steps:

1. Ensure you have the AWS CLI installed and configured with the necessary permissions to access ECR.

2. Run the following command to obtain an authentication token and generate a Kubernetes secret:

    ```bash
    ECR_PASSWORD=$(aws ecr get-login-password --region eu-west-1)
    kubectl create secret docker-registry <image-pull-secret-name> --docker-server=<aws_account_id>.dkr.ecr.<region>.amazonaws.com --docker-username=AWS --docker-password="$ECR_PASSWORD"
    ```

    Replace `<region>` with your AWS region, `<image-pull-secret-name>` with a name for your Kubernetes secret, and `<aws_account_id>` with your AWS account ID.

3. Once the secret is created, you can reference it in your `values.yaml` under the `imagePullSecrets` section:

Note - ECR auth tokens expire after ~12 hours. 

## Customizing the Chart Before Installing

To configure the chart with custom values, you can:

1. Edit the `values.yaml` file directly.
2. Provide a custom `values.yaml` file during installation:

    ```
    helm install my-release ./clone -f my-values.yaml
    ```


The following table lists the configurable parameters of the `dubhub-clone` chart and their default values.


| Parameter                  | Description                                                                                      | Default               |
|----------------------------|--------------------------------------------------------------------------------------------------|-----------------------|
| `image.repository`         | Image repository                                                                                 |                       |
| `image.pullPolicy`         | Image pull policy                                                                                | `IfNotPresent`        |
| `image.tag`                | Image tag                                                                                        | `latest`              |
| `service.type`             | Kubernetes Service type                                                                          | `ClusterIP`           |
| `service.port`             | Kubernetes Service port                                                                          | `5432`                |
| `env[0].name`              | Environment variable name for the encryption password defined when the Dub was created           | `PASSWORD`            |
| `env[0].value`             | Environment variable value for the encryption password stored as a secret                        |                       |
| `env[1].name`              | Environment variable name for the U value                                                        | `U`                   |
| `env[1].value`             | Environment variable value for the U value                                                       |                       |
| `env[2].name`              | Environment variable name for the clone UUID                                                     | `UUID`                |
| `env[2].value`             | Environment variable value for the clone UUID, if set the clone will not lose data after restart (optional) |                       |
| `env[3].name`              | Environment variable name for the clone backup schedule                                          | `BACKUP_SCHEDULE`     |
| `env[3].value`             | Environment variable value for the backup schedule in cron format (optional)                    |                       |
| `env[4].name`              | Environment variable name for the clone backup directory                                         | `BACKUP_DIR`          |
| `env[4].value`             | Environment variable value for the clone backup directory (optional)                            | `/dubhub/backups`     |
| `env[5].name`              | Environment variable name for the number of backups that are saved                              | `MAX_BACKUPS`         |
| `env[5].value`             | Environment variable value for the number of backups that are saved (optional)                  |                       |
| `env[6].name`              | Environment variable name for monitoring available space                                         | `SPACE_USAGE_MIN_PERCENT` |
| `env[6].value`             | Environment variable value for the min amount of available space allowed (optional)             |                       |
| `imagePullSecrets[0].name` | Name of the secret for image pulling                                                             |                       |
| `storage.size`             | Size of the volume                                                                               |                       |
| `storage.storageClassName` | Name of the storage class to provision the volumes                                               |                       |



## Example

create a secret with the encryption password:

```bash
kubectl create secret generic <my-secret-name> --from-literal=password=<encryption-password>
```

example values.yaml:
```yaml
image:
  repository: <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repository-name>
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 5432

env:
  - name: PASSWORD
    valueFrom:
      secretKeyRef:
        name: <my-secret-name>
        key: password
  - name: U
    value: "<user-value>"

imagePullSecrets:
  - name: <my-image-pull-secret-name>

```


## Running Custom Initialization Commands

Any additional SQL commands that are required to run after the clone is up (for example, creating a database) can be updated in the configmap.yaml file under the init_clone.sql section.

Example configmap.yaml:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dubhub-init-sql
data:
  init_clone.sql: |
    -- Example: create app DB
    CREATE DATABASE appdb;

```

## Installing the Chart

To install the chart, use the following command:

    helm install <my-release> ./clone

## Uninstalling the Chart

To uninstall/delete the deployment, use the following command:

    helm delete <my-release>

This command removes all the Kubernetes components associated with the chart and deletes the release.