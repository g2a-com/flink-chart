# Apache Flink Helm Chart

This is an implementation of https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/kubernetes.html

This chart will install session cluster https://ci.apache.org/projects/flink/flink-docs-stable/ops/deployment/kubernetes.html#flink-session-cluster-on-kubernetes.
If you are interested in supporting session/job clusters: https://github.com/GoogleCloudPlatform/flink-on-k8s-operator

## Prerequisites:

* Kubernetes 1.3 with alpha APIs enabled and support for storage classes

* PV support on underlying infrastructure

* Requires at least `v2.0.0-beta.1` version of helm to support
  dependency management with requirements.yaml

## StatefulSet Details

* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

## StatefulSet Caveats

* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations

## Chart Details

This chart will do the following:

* Implement a dynamically scalable Flink (Jobmanagers and Taskmanagers) cluster using Kubernetes StatefulSets

### Installing the Chart

To install the chart with the release name `my-flink` in the default
namespace:

```shell
$ helm repo add riskfocus https://riskfocus.github.io/helm-charts-public
$ helm repo update
$ helm install --name my-flink riskfocus/flink
```

If using a dedicated namespace(recommended) then make sure the namespace
exists with:

```shell
$ helm repo add riskfocus https://riskfocus.github.io/helm-charts-public
$ helm repo update
$ helm install --name my-flink --namespace flink riskfocus/flink
```

The chart can be customized using the following configurable parameters (other parameters can be found in `values.yaml`):

| Parameter                                | Description                                                                                                                                                              | Default                |
|------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------|
| `image.repository`                       | Flink Container image name                                                                                                                                               | `flink`                |
| `image.tag`                              | Flink Container image tag                                                                                                                                                | `1.10.0-scala_2.12`    |
| `image.PullPolicy`                       | Flink Containers pull policy                                                                                                                                             | `IfNotPresent`         |
| `flink.monitoring.enabled`               | Enables Flink monitoring                                                                                                                                                 | `true`                 |
| `jobmanager.highAvailability.enabled`    | Enables Jobmanager HA mode key                                                                                                                                           | `false`                |
| `jobmanager.highAvailability.storageDir` | storageDir for Jobmanager in HA mode                                                                                                                                     | `null`                 |
| `jobmanager.replicaCount`                | Jobmanagers count context                                                                                                                                                | `1`                    |
| `jobmanager.heapSize`                    | Jobmanager HeapSize options                                                                                                                                              | `1g`                   |
| `jobmanager.resources`                   | Jobmanager resources                                                                                                                                                     | `{}`                   |
| `taskmanager.resources`                  | Taskmanager Resources key                                                                                                                                                | `{}`                   |
| `taskmanager.heapSize`                   | Taskmanager heapSize mode                                                                                                                                                | `1g`                   |
| `jobmanager.replicaCount`                | Taskmanager count context                                                                                                                                                | `1`                    |
| `taskmanager.numberOfTaskSlots`          | Number of Taskmanager taskSlots resources                                                                                                                                | `1`                    |
| `taskmanager.resources`                  | Taskmanager resources                                                                                                                                                    | `{}`                   |
| `secrets.bitnamiSealedSecrets.enabled`   | Enables creation of [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)                                                                                     | `false`                |

### High Availability

Current implementation supports only Kubernetes based HA. To enable it you have to create RoleBinding with cluster-admin role for Flink's ServiceAccounts.

```shell
$ kubectl create rolebinding flink-prod-taskmanager-admin --clusterrole=cluster-admin --serviceaccount=flink-prod:flink-taskmanager
$ kubectl create rolebinding flink-prod-jobmanager-admin --clusterrole=cluster-admin --serviceaccount=flink-prod:flink-jobmanager
```

Then enable HA itself.

```yaml
  highAvailability:
    enabled: false
    clusterId: flinkcluster01
    # storageDir for Jobmanagers. DFS expected.
    # Docs - Storage directory (required): JobManager metadata is persisted in the file system storageDir
    storageDir:
```

It is highly recommended that you configure section (below) responsible for storage shared between JobManagers and TaskManagers.
Without this JARs will be uploaded only to one JobManager and if it fails you will need to re-upload them.
Moreover, the LoadBalancer will be switching between all the available JobManagers. Therefore, as JARs will be on single one
of JobManager you will see them disappearing from the "Uploaded Jars" list in the Flink's UI.

```yaml
# Configures storage for JAR files shared between JobManagers and TaskManagers.
# Configures Flink parameter "web.upload.dir" accordingly.
sharedWebUploadDir:
  enabled: false
  initialContainerImage: busybox:1.33
  path: /mnt/flink-storage

  # Used (and required) only if useExistingPVC is set to "false"
  storageClass:
  size: 5Gi

  # If set to "true" use already existing PVC if false create a new one upon chart installation.
  # [WARNING] storage content is bound to PVC, so if you uninstall chart (while useExistingPVC is "false") you will loose your data.
  # There is no such issue with a external PVC (useExistingPVC set to "true").
  useExistingPVC: false
  # Name of the existing PVC
  pvcName:
```
