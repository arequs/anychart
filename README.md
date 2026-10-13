# Anychart

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/anychart)](https://artifacthub.io/packages/search?repo=anychart)

Anychart is a general-purpose Helm chart for deploying a containerized application
as a Deployment, StatefulSet, DaemonSet, or scheduled CronJob, together with optional
networking, storage, ConfigMaps, secrets, RBAC, monitoring, and autoscaling resources.
Templates are self-contained and do not use named Helm helpers.

## Requirements

- Helm 3.8+ or Helm 4; Kubernetes 1.25+.
- Kubernetes 1.27+ when setting `cronjob.timeZone`.
- An ingress controller for Ingress; Traefik and its CRDs for IngressRoute.
- Secrets Store CSI Driver, its CRDs, and the chosen provider for SecretProviderClass.
- Prometheus Operator and its CRDs for ServiceMonitor.
- A metrics provider for HorizontalPodAutoscaler; resource utilization metrics
  also require corresponding container resource requests.

The chart does not install controllers or CRDs.

Use `examples/statefulset.yaml` for a StatefulSet with a headless Service, or
`examples/cronjob.yaml` for a task every 15 minutes. See the
[Artifact Hub listing](https://artifacthub.io/packages/helm/anychart/anychart) for
hosted chart installation details. Artifact Hub is a catalog, not a Helm
repository endpoint.

## Configuration

See [values.yaml](values.yaml) for defaults and [examples](examples/) for working
configurations. Enable at most one workload per release. Supporting resources
can also be deployed without a workload.

| Values section | Resource / behavior |
| --- | --- |
| `deployment` | Deployment; image required when enabled |
| `statefulset` | StatefulSet; image required; defaults to the release name for its governing Service |
| `daemonset` | DaemonSet; image required; one pod per eligible node |
| `configMap` | ConfigMap with string `data`, base64 `binaryData`, and optional immutability |
| `serviceMonitor` | Prometheus ServiceMonitor; nonempty `endpoints` required |
| `role` / `roleBinding` | Namespaced RBAC rules and bindings |
| `clusterRole` / `clusterRoleBinding` | Cluster-scoped RBAC rules and bindings |
| `middleware` | Traefik HTTP Middleware; nonempty `spec` required |
| `ingressrouteTCP` | Traefik TCP routing with optional TLS |
| `ingressrouteUDP` | Traefik UDP routing; no match rules or TLS |
| `cronjob` | CronJob; image and schedule required |
| `service` | Service; nonempty `ports` required; `clusterIP: None` creates a headless Service |
| `persistence` | Standalone PVC; default 1Gi, ReadWriteOnce |
| `serviceAccount` | Create an account or reference an existing account using `name` |
| `priorityClass` | Create a cluster-scoped class or reference an existing class using `name` |
| `autoscaler` | HPA for the enabled Deployment or StatefulSet, selected by `targetKind` |
| `secrets.list` | Named Secrets with `data` (base64) and/or `stringData` |
| `secretProviderClass` | CSI secret provider configuration; `provider` required |
| `ingress` | Standard Kubernetes Ingress; nonempty `rules` required |
| `ingressroute` | Traefik IngressRoute; nonempty `routes` required |

Workloads support `command`, `args`, `env`, `envFrom`, `resources`, explicit
container `ports`, `imagePullSecrets`, `runtimeClassName`, `nodeSelector`,
`tolerations`, `affinity`, `hostIPC`, `initContainers`, `podSecurityContext`,
`securityContext`, `podAnnotations`, probes, `volumeMounts`, and `volumes`.
Values are literal Kubernetes YAML; embedded Helm expressions are not evaluated.

Service ports and container ports are configured independently, so named
`targetPort` values work. A numeric Service target does not require an explicit
container port. Custom labels do not change workload selectors or Service
routing. Workload labels must not override the reserved
`app.kubernetes.io/component` and `app.kubernetes.io/instance` keys.

Most resource names equal the release name. `serviceAccount.name`,
`priorityClass.name`, and `secretProviderClass.name` override their names;
Secrets use each list entry's `name`. A named account or priority class is
referenced by all workloads even when its creation is disabled. Avoid sharing
created cluster-scoped PriorityClass names between releases.

`replicas: 0` is supported. When autoscaling is enabled, the target workload omits
`replicas` so upgrades do not reset the HPA's desired count. A created service
account defaults to `automountServiceAccountToken: false`; an existing account
keeps its own settings.

PVC creation does not automatically mount storage. Configure workload volumes
and volume mounts, or use StatefulSet `volumeClaimTemplates`. Omitting
`persistence.storageClass` uses the cluster default; setting it to `""` explicitly
disables dynamic provisioning.

### Configuration, node agents, and RBAC

[examples/daemonset.yaml](examples/daemonset.yaml) combines a DaemonSet,
ConfigMap, ServiceAccount, Role/RoleBinding, and ClusterRole/ClusterRoleBinding.
DaemonSets support the shared pod/container options, `updateStrategy`,
`minReadySeconds`, and `revisionHistoryLimit`. They have no replica count or HPA.
Only one of Deployment, StatefulSet, DaemonSet, and CronJob may be enabled.

`configMap.data` values must be strings, including quoted numbers and booleans.
Reference the ConfigMap explicitly through `envFrom`, `env`, or a volume; creating
it does not mount it or automatically restart pods when its contents change.
An immutable ConfigMap must be replaced under a new name to change its data.

Each RBAC section creates one object and accepts `name`, `labels`, and
`annotations`. Roles require nonempty `rules`. Bindings default to the enabled or
named `serviceAccount` in the release namespace; otherwise explicit `subjects`
are required. Set subjects explicitly to bind users, groups, or accounts in other
namespaces (include `namespace` for ServiceAccount subjects).

RoleBinding accepts `roleRef.kind: Role` or `ClusterRole`; ClusterRoleBinding
always references a ClusterRole. `roleRef.name` can reference an existing role.
Defaults follow `role.name` or `clusterRole.name`. Creating a binding does not
create its role unless that role is also enabled. Binding role references are
immutable in Kubernetes.

ClusterRole and ClusterRoleBinding names default to `<namespace>-<release>`;
all other new resource names default to the release name. Explicit names override
these defaults. A RoleBinding to a ClusterRole grants its namespaced permissions
only within the binding's namespace; a ClusterRoleBinding grants cluster-wide
access. Choose rules and subjects accordingly. Applications accessing the API
also need a token; set `serviceAccount.automountServiceAccountToken: true` when
using the chart-created account for that purpose.

### Monitoring and Traefik routing

[examples/monitoring-routing.yaml](examples/monitoring-routing.yaml) shows all
three Traefik routing kinds, an HTTP Middleware, and a ServiceMonitor. Backend
Service names and Traefik entrypoints in that example must exist in your setup.

The existing `ingressroute` remains the HTTP/HTTPS router. `ingressrouteTCP` and
`ingressrouteUDP` are separate resources and can coexist with it. TCP supports
`tls`, including passthrough; UDP routes contain `services` without HTTP/TCP
`match`, `kind`, or `middlewares` fields. Configure UDP backend Service ports with
`protocol: UDP`. See the upstream [TCP](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/tcp/ingressroutetcp/)
and [UDP](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/crd/udp/ingressrouteudp/) references.

`middleware.spec` is a literal Traefik HTTP Middleware specification. Reference
its name in `ingressroute.routes[].middlewares`; creation alone does not attach
it. HTTP Middleware cannot be referenced as TCP middleware: Traefik uses a
separate `MiddlewareTCP` kind, which this chart does not create.

ServiceMonitor defaults to selecting this release's Service labels in the same
namespace. Override `selector` and `namespaceSelector` to monitor other Services.
Each endpoint's `port` is a **Service port name**, not a container port number.
Expose a real metrics endpoint through a Service and set `serviceMonitor.labels`
to match your Prometheus instance's ServiceMonitor selector. Optional fields are
`jobLabel`, `targetLabels`, `podTargetLabels`, and `sampleLimit`; endpoint objects
pass through Kubernetes YAML for TLS, authentication, and relabeling settings.
See the [Prometheus Operator guide](https://prometheus-operator.dev/docs/developer/getting-started/).

### Scheduled jobs

```yaml
cronjob:
  enabled: true
  image: busybox:1.37
  schedule: "0 * * * *"
  timeZone: Etc/UTC
  command: ["/bin/sh", "-c"]
  args: ["echo hourly task"]
  concurrencyPolicy: Forbid
  restartPolicy: Never
  backoffLimit: 0
  successfulJobsHistoryLimit: 0
  failedJobsHistoryLimit: 1
```

CronJobs support `Allow`, `Forbid` (default), or `Replace` concurrency policies;
`Never` or `OnFailure` restart policies; suspension, start/execution deadlines,
history limits, and finished-job TTL. Zero retry/history limits are preserved.
Release names must be at most 52 characters. Without `timeZone`, scheduling uses
the controller's timezone. Tasks should tolerate retries and duplicate execution.
See the [Kubernetes CronJob documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).
