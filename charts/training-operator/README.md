# training-operator

![Version: v1.9.3](https://img.shields.io/badge/Version-v1.9.3-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: v1.9.3](https://img.shields.io/badge/AppVersion-v1.9.3-informational?style=flat-square)

A Helm chart for deploying the Kubeflow Training Operator on Kubernetes. The Training Operator manages distributed training jobs for machine learning frameworks including JAX, MPI, PaddlePaddle, PyTorch, TensorFlow, and XGBoost.

**Homepage:** <https://github.com/kubeflow/training-operator/tree/release-1.9>

---

## Notes

To mitigate security risks from over-privileged access, this component has refined its auto-created RBAC rules by removing broad, high-privilege configurations. Since the Launcher Pod requires read and exec access to Worker Pods, users must manually manage permissions for the ServiceAccount used by MPIJobs. Alternatively, you can use the [arena CLI](https://github.com/kubeflow/arena/tree/develop-v2), which automatically manages the ServiceAccount and RBAC permissions required by MPIJobs. Please refer to the example below, which lists the minimum required permissions:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: mpi
  namespace: default
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resourceNames:
  #  <job-name>-worker-<index>
  - mpi-worker-0
  - mpi-worker-1
  resources:
  - pods/exec
  verbs:
  - create
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mpi
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mpi
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mpi
subjects:
- kind: ServiceAccount
  name: mpi
  namespace: default
---
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: mpi
  namespace: default
spec:
  mpiReplicaSpecs:
    Launcher:
      template:
        spec:
          ...
          serviceAccountName: mpi
      ...
    Worker:
      replicas: 2
      ...
  ...
```

**Users assume full responsibility for any security issues arising from manually granting elevated permissions.**

---

## Introduction

This Helm chart installs the Kubeflow Training Operator to your Kubernetes cluster. The Training Operator provides Kubernetes custom resources that make it easy to run distributed training workloads. It supports JAXJob, MPIJob, PaddleJob, PyTorchJob, TensorFlowJob, and XGBoostJob CRDs.

## Prerequisites

- Helm >= 3
- Kubernetes >= 1.19

 **Note:** this chart installs cluster-scoped resources (CRDs, ClusterRole, ValidatingWebhookConfiguration) and therefore requires cluster-scoped RBAC permissions. It is intended to be installed once per cluster; installing multiple releases will result in resource name conflicts.
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

### Submit training job

After installing the Training Operator, you can submit distributed training jobs by creating CRs (e.g., `PyTorchJob`, `MPIJob`, `TFJob`) directly. See the [Kubeflow Trainer examples](https://github.com/kubeflow/trainer/tree/release-1.9/examples) for CR examples.

Alternatively, you can use [arena-v2](https://github.com/kubeflow/arena/blob/develop-v2/README.md), a command-line tool that simplifies submitting and managing distributed training jobs on Kubernetes. It wraps the CRs above into a friendly CLI, lowering the barrier to running distributed training workloads.

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

| Key | Description | Default |
|-----|-------------|---------|
| nameOverride | String to partially override release name. | `""` |
| fullnameOverride | String to fully override release name. | `""` |
| replicas | Number of deployment replicas. Set to 2+ for HA. | `2` |
| image.repository | Image repository. | `"registry-cn-beijing.ack.aliyuncs.com/acs/training-operator"` |
| image.tag | Image tag. | `"15cc1de-aliyun"` |
| image.pullPolicy | Image pull policy. | `"IfNotPresent"` |
| imagePullSecrets | List of image pull secret names for private registries. | `[]` |
| pytorchInitContainer.customTemplate | When true, creates a ConfigMap with a custom init container template (getent hosts) and mounts it at /etc/config/initContainer.yaml. When false, the operator uses its built-in Go template. | `true` |
| pytorchInitContainer.image.repository | Init container image repository. | `"registry-cn-beijing.ack.aliyuncs.com/acs/alpine"` |
| pytorchInitContainer.image.tag | Init container image tag. | `"3.22.2"` |
| pytorchInitContainer.imagePullPolicy | Image pull policy for the init container. | `"IfNotPresent"` |
| pytorchInitContainer.maxTries | Number of DNS resolution attempts for the init container. Passed to the operator via the --pytorch-init-container-max-tries CLI flag. | `100` |
| pytorchInitContainer.sleepSeconds | Sleep interval (seconds) between DNS resolution attempts. | `2` |
| pytorchInitContainer.resources | Resource requests and limits for the init container. | `{"limits":{"cpu":"100m","memory":"20Mi"},"requests":{"cpu":"50m","memory":"10Mi"}}` |
| mpiKubectlDeliveryImage.repository | MPI kubectl delivery image repository. | `"registry-cn-beijing.ack.aliyuncs.com/acs/kubectl-delivery"` |
| mpiKubectlDeliveryImage.tag | MPI kubectl delivery image tag. | `"15cc1de-aliyun"` |
| mpiDisableRBACManagement | When true, disables auto-provisioning of ServiceAccounts for MPIJobs. This reduces operator's RBAC scope but requires manually setting `serviceAccountName` on the Launcher with a Role granting `get`, `list`, `watch` on `pods` and `create` on `pods/exec`. If unset, it falls back to the namespace's `default` ServiceAccount, which typically lacks permissions and causes job failure. | `false` |
| serviceAccount.create | Create a new service account. | `true` |
| serviceAccount.name | Service account name. If empty, a name is derived from the release name. | `""` |
| service.type | Service type. | `"ClusterIP"` |
| service.metricsPort | Metrics service port. | `8080` |
| service.webhookPort | Webhook service port. | `443` |
| service.annotations | Service annotations. | `{"prometheus.io/path":"/metrics","prometheus.io/scrape":"true","prometheus.io/port":"8080"}` |
| webhook.timeoutSeconds | Timeout for admission webhook requests in seconds. | `30` |
| podAnnotations | Pod annotations. | `{"sidecar.istio.io/inject":"false"}` |
| podSecurityContext | Pod security context. | `{}` |
| securityContext | Container security context. | `{"allowPrivilegeEscalation":false,"runAsNonRoot":true,"runAsUser":65532,"readOnlyRootFilesystem":true,"seccompProfile":{"type":"RuntimeDefault"}}` |
| resources | Resource requests and limits. | `{"requests":{"memory":"128Mi","cpu":"100m"},"limits":{"memory":"512Mi","cpu":"500m"}}` |
| nodeSelector | Node selector. | `{}` |
| tolerations | Tolerations. | `[]` |
| affinity | Affinity rules. When empty and replicas > 1, a podAntiAffinity is auto-generated. | `{}` |
| podDisruptionBudget.enable | Enable PodDisruptionBudget for the operator deployment. | `true` |
| podDisruptionBudget.minAvailable | Minimum number of available pods. Cannot be set together with maxUnavailable. | `1` |
| leaderElection.enable | Enable leader election. Passes `--leader-elect` to the operator. Required when replicas > 1. | `true` |
| terminationGracePeriodSeconds | Termination grace period seconds. | `10` |
| testImage.repository | Test pod image repository. | `"registry-cn-beijing.ack.aliyuncs.com/acs/busybox"` |
| testImage.tag | Test pod image tag. | `"stable"` |

## Maintainers

| Name | Url |
| ---- | --- |
| Kubeflow Training | <https://github.com/kubeflow/training-operator> |

## Links

- **Upstream project:** <https://github.com/kubeflow/training-operator/tree/release-1.9>
- **Kubeflow Trainer documentation:** <https://github.com/kubeflow/training-operator>
