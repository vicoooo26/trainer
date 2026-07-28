# training-operator

![Version: v1.9.3](https://img.shields.io/badge/Version-v1.9.3-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: v1.9.3](https://img.shields.io/badge/AppVersion-v1.9.3-informational?style=flat-square)

A Helm chart for deploying the Kubeflow Training Operator on Kubernetes. The Training Operator manages distributed training jobs for machine learning frameworks including JAX, MPI, PaddlePaddle, PyTorch, TensorFlow, and XGBoost.

**Homepage:** <https://github.com/kubeflow/training-operator/tree/release-1.9>

## Introduction

This Helm chart installs the Kubeflow Training Operator to your Kubernetes cluster. The Training Operator provides Kubernetes custom resources that make it easy to run distributed training workloads. It supports JAXJob, MPIJob, PaddleJob, PyTorchJob, TensorFlowJob, and XGBoostJob CRDs.

## Prerequisites

- Helm >= 3
- Kubernetes >= 1.19

**Note:** the `training-operator` requires the following **cluster-scoped** RBAC permissions, including but not limited to:
  - Jobs: `create`, `get`, `list`, `watch`, `update`, `delete`
  - Pods: `create`, `get`, `list`, `watch`, `delete`
  - Services: `create`, `get`, `list`, `watch`

## Usage

### Install the chart

Clone this repository and install from the local chart:

```shell
git clone https://github.com/AliyunContainerService/trainer.git
cd trainer
helm install [RELEASE_NAME] ./charts/training-operator
```

For example, if you want to create a release with name `training-operator` in the `kubeflow` namespace:

```shell
helm install training-operator ./charts/training-operator \
    --namespace kubeflow \
    --create-namespace
```

Note that by passing the `--create-namespace` flag to the `helm install` command, `helm` will create the release namespace if it does not exist.

See [helm install](https://helm.sh/docs/helm/helm_install) for command documentation.

### Upgrade the chart

```shell
helm upgrade [RELEASE_NAME] ./charts/training-operator [flags]
```

See [helm upgrade](https://helm.sh/docs/helm/helm_upgrade) for command documentation.

#### PyTorch init container template

By default, this chart creates a ConfigMap that overrides the PyTorch init container template to use `getent hosts` (forcing DNS A-record lookups). This prevents spurious failures in IPv4-only environments. If your environment relies on other DNS record types (e.g., AAAA for IPv6), disable this feature:

```shell
helm upgrade [RELEASE_NAME] ./charts/training-operator \
    --set pytorchInitContainer.customTemplate=false
```

### Uninstall the chart

```shell
helm uninstall [RELEASE_NAME]
```

This removes all the Kubernetes resources associated with the chart and deletes the release, except for the CRDs, which must be removed manually.

See [helm uninstall](https://helm.sh/docs/helm/helm_uninstall) for command documentation.

### Upgrading CRDs

CRDs under `crds/` are not managed by Helm's upgrade process. When upgrading between chart versions, apply CRD updates manually.

### High Availability

To run the Training Operator with high availability (avoid single point of failure), set `replicas` to 2 or more:

```shell
helm install training-operator ./charts/training-operator \
    --namespace kubeflow \
    --set replicas=2
```

When `replicas > 1`:
- **Leader election** is enabled by default (`leaderElection.enable: true`), passing `--leader-elect` to the operator so only one replica actively reconciles CRs while others stand by.
- **Pod anti-affinity** is auto-generated (preferredDuringScheduling, weight 100) to spread replicas across nodes. To use custom affinity rules, set the `affinity` value — it takes full precedence over the auto-generated defaults.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| nameOverride | string | `""` | String to partially override release name. |
| fullnameOverride | string | `""` | String to fully override release name. |
| replicas | int | `2` | Number of deployment replicas. Set to 2+ for HA. |
| image.repository | string | `"registry-cn-beijing.ack.aliyuncs.com/acs/training-operator"` | Image repository. |
| image.tag | string | `"15cc1de-aliyun"` | Image tag. |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy. |
| imagePullSecrets | list | `[]` | List of image pull secret names for private registries. |
| pytorchInitContainer.customTemplate | bool | `true` | When true, creates a ConfigMap with a custom init container template (getent hosts) and mounts it at /etc/config/initContainer.yaml. When false, the operator uses its built-in Go template. |
| pytorchInitContainer.image.repository | string | `"registry-cn-beijing.ack.aliyuncs.com/acs/alpine"` | Init container image repository. |
| pytorchInitContainer.image.tag | string | `"3.22.2"` | Init container image tag. |
| pytorchInitContainer.imagePullPolicy | string | `"IfNotPresent"` | Image pull policy for the init container. |
| pytorchInitContainer.maxTries | int | `100` | Number of DNS resolution attempts for the init container. Passed to the operator via the --pytorch-init-container-max-tries CLI flag. |
| pytorchInitContainer.sleepSeconds | int | `2` | Sleep interval (seconds) between DNS resolution attempts. |
| pytorchInitContainer.resources | object | `{"limits":{"cpu":"100m","memory":"20Mi"},"requests":{"cpu":"50m","memory":"10Mi"}}` | Resource requests and limits for the init container. |
| mpiKubectlDeliveryImage.repository | string | `"registry-cn-beijing.ack.aliyuncs.com/acs/kubectl-delivery"` | MPI kubectl delivery image repository. |
| mpiKubectlDeliveryImage.tag | string | `"15cc1de-aliyun"` | MPI kubectl delivery image tag. |
| mpi.disableRBACManagement | bool | `false` | When true, operator will not create SA/Role/RoleBinding for MPIJobs. |
| serviceAccount.create | bool | `true` | Create a new service account. |
| serviceAccount.name | string | `""` | Service account name. If empty, a name is derived from the release name. |
| service.type | string | `"ClusterIP"` | Service type. |
| service.metricsPort | int | `8080` | Metrics service port. |
| service.webhookPort | int | `443` | Webhook service port. |
| service.annotations | object | `{"prometheus.io/path":"/metrics","prometheus.io/scrape":"true","prometheus.io/port":"8080"}` | Service annotations. |
| webhook.timeoutSeconds | int | `30` | Timeout for admission webhook requests in seconds. |
| podAnnotations | object | `{"sidecar.istio.io/inject":"false"}` | Pod annotations. |
| podSecurityContext | object | `{}` | Pod security context. |
| securityContext | object | `{"allowPrivilegeEscalation":false,"runAsNonRoot":true,"runAsUser":65532,"readOnlyRootFilesystem":true,"seccompProfile":{"type":"RuntimeDefault"}}` | Container security context. |
| resources | object | `{"requests":{"memory":"128Mi","cpu":"100m"},"limits":{"memory":"512Mi","cpu":"500m"}}` | Resource requests and limits. |
| nodeSelector | object | `{}` | Node selector. |
| tolerations | list | `[]` | Tolerations. |
| affinity | object | `{}` | Affinity rules. When empty and replicas > 1, a podAntiAffinity is auto-generated. |
| podDisruptionBudget.enable | bool | `true` | Enable PodDisruptionBudget for the operator deployment. |
| podDisruptionBudget.minAvailable | int | `1` | Minimum number of available pods. Cannot be set together with maxUnavailable. |
| leaderElection.enable | bool | `true` | Enable leader election. Passes `--leader-elect` to the operator. Required when replicas > 1. |
| terminationGracePeriodSeconds | int | `10` | Termination grace period seconds. |
| testImage.repository | string | `"registry-cn-beijing.ack.aliyuncs.com/acs/busybox"` | Test pod image repository. |
| testImage.tag | string | `"stable"` | Test pod image tag. |

## Maintainers

| Name | Url |
| ---- | --- |
| Kubeflow Training | <https://github.com/kubeflow/training-operator> |

## Links

- **Upstream project:** <https://github.com/kubeflow/training-operator/tree/release-1.9>
- **Kubeflow Trainer documentation:** <https://github.com/kubeflow/training-operator>
