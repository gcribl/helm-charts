# OpenShift-Specific Info and Errata

This document describes how to run the Cribl Stream Leader and Worker Group Helm charts on Red Hat OpenShift. OpenShift enforces non-root UIDs and Security Context Constraints (SCCs) that require specific values and, in some cases, custom SCCs and ServiceAccount binding.

## Prerequisites

1. **Create the OpenShift project first** (do not rely on creating the namespace as part of `helm install`).
2. **Get the project UID range:** run `oc describe project <project-name>` and note the first UID in the `openshift.io/sa.scc.uid-range` annotation (e.g. `1000620000`). Use this value as `runAsUser`, `runAsGroup`, and `fsGroup` in the pod security context so that mounted volumes are accessible to the pod.

### Container image (CRI-O short-name mode)

OpenShift uses CRI-O, which may enforce **short name mode**. Unqualified image names such as `cribl/cribl:4.17.0` can fail with `ImageInspectError` / "returns ambiguous list". The chart defaults use a fully qualified repository:

```yaml
criblImage:
  repository: docker.io/cribl/cribl
  tag: 4.17.0
```

## Stream Leader chart on OpenShift

### Namespace and UID alignment

Set the leader pod security context in `values.yaml` so the container runs with an allowed UID/GID and volumes are accessible:

```yaml
podSecurityContext:
  runAsUser: <project-uid>   # e.g. 1000620000
  runAsGroup: <project-uid>
  fsGroup: <project-uid>

serviceAccount:
  create: true
  name: cribl-leader-sa
```

### Writable configuration volume

Because `/opt/cribl` is root-owned in the image and OpenShift runs with an arbitrary UID, the leader's state/config must live on a writable volume. Set `config.criblVolumeDir` to a path where the config-storage volume is mounted (e.g. `/tmp`):

```yaml
config:
  criblVolumeDir: /tmp   # config-storage is mounted here; CRIBL_VOLUME_DIR is set to this path
```

The chart mounts the existing config-storage volume (PVC or emptyDir) at this path and sets `CRIBL_VOLUME_DIR` accordingly.

### SCC and ServiceAccount binding

Create a custom SCC that allows `persistentVolumeClaim` (and optionally `hostPath` only if required), and uses `RunAsAny` for `runAsUser`, `fsGroup`, and `supplementalGroups`. Bind it to the chart's ServiceAccount:

```bash
oc adm policy add-scc-to-user <your-custom-scc> -z cribl-leader-sa -n <namespace>
```

### Capabilities (securityContext)

For low ports or diagnostics, add capabilities in the leader values:

```yaml
securityContext:
  capabilities:
    add:
      - CAP_NET_BIND_SERVICE
      - CAP_SYS_PTRACE
      - CAP_DAC_READ_SEARCH
      - CAP_NET_ADMIN
```

### Service type and Ingress / Route

OpenShift often disallows `LoadBalancer`. Override the external service type:

```yaml
service:
  externalType: ClusterIP   # or NodePort
```

Expose the leader via OpenShift Route or Ingress (e.g. `ingress.enable: true` and configure `ingress.annotations` / `ingress.labels` for your ingress controller).

### TLS and health checks

When the leader API is served over TLS, set `config.healthScheme: HTTPS` so liveness and readiness probes use HTTPS. Otherwise the probes keep using HTTP and can fail, causing restarts.

```yaml
config:
  healthScheme: HTTPS
```

Provide TLS assets (e.g. via `extraConfigmapMounts` / `extraSecretMounts` and env or pre-start commands) as required by your TLS setup.

## Stream Worker Group chart on OpenShift

### Pod security context and writable volume

Apply the same UID/GID pattern as the leader:

```yaml
podSecurityContext:
  runAsUser: <project-uid>
  runAsGroup: <project-uid>
  fsGroup: <project-uid>

config:
  criblVolumeDir: /tmp
  # criblVolumeStorage: {}   # omit for emptyDir; or existingClaim: "<pvc-name>" for PVC
```

When `config.criblVolumeDir` is set, the chart adds a volume (emptyDir or PVC from `config.criblVolumeStorage.existingClaim`), mounts it at that path, and sets `CRIBL_VOLUME_DIR`.

### SCC and ServiceAccount

Configure the workergroup ServiceAccount in values and bind your custom SCC to it:

```yaml
serviceAccount:
  create: true
  name: cribl-worker-sa
```

```bash
oc adm policy add-scc-to-user <your-custom-scc> -z cribl-worker-sa -n <namespace>
```

### Capabilities

Same as leader when workers need low ports or debugging:

```yaml
securityContext:
  capabilities:
    add:
      - CAP_NET_BIND_SERVICE
      - CAP_SYS_PTRACE
      - CAP_DAC_READ_SEARCH
      - CAP_NET_ADMIN
```

### Service type and ports

Override the default `LoadBalancer` service type:

```yaml
service:
  type: ClusterIP   # or NodePort; use Route/Ingress or external LB as needed
```

The chart's service supports **TCP only**; UDP is not supported in the service definition.

### Connectivity to leader

Set `config.leaderReleaseName` to the **leader Helm release name** so workers connect to `<leaderReleaseName>-leader-internal`. If you use `config.host` instead, it must be exactly `<release>-leader-internal` (e.g. `myrelease-leader-internal`). Otherwise workers can fail with DNS resolution errors (e.g. `getaddrinfo ENOTFOUND logstream-leader-internal`).

For TLS-enabled leaders, set `config.tlsLeader.enable: true` and the relevant TLS options (caPath, certPath, etc.) to match the leader.

### Resource-constrained clusters (e.g. CRC, single node)

The workergroup chart defaults enable **HorizontalPodAutoscaler** with **`minReplicas: 2`** and each pod **`resources.requests.cpu: 1250m`**. That reserves **2.5 vCPU** for workers alone, which often fails scheduling with `Insufficient cpu` after the leader and OpenShift system pods.

For local OpenShift (CRC), disable autoscaling, use one replica, and lower CPU **requests** (limits can stay higher):

```yaml
autoscaling:
  enabled: false
replicaCount: 1
resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 4096Mi
```

Then `helm upgrade` the release. A ready-made overlay is in [examples/logstream-workergroup-local-openshift.values.yaml](examples/logstream-workergroup-local-openshift.values.yaml).

### Ingress labels and annotations

OpenShift Route/ingress controllers may require specific labels or annotations. Use `ingress.labels` and `ingress.annotations` (and per-ingress labels/annotations where the chart supports them) to satisfy your cluster's policies.

## hostPath

Many OpenShift clusters disallow `hostPath` volumes via SCC. Use PVC or `emptyDir` in `extraVolumeMounts` and for the config/cribl volume. Avoid `hostPath` unless you have a custom SCC that allows it.

## Persistent Queues (PQ) and autoscaling

When using persistent queues on Kubernetes/OpenShift:

- **Prefer StatefulSet** deployment mode for workers so pods have stable identity and volume attachment.
- **Avoid aggressive scale-in**; sudden scale-down can leave PQ data on detached volumes.
- **extraVolumeMounts** are used both for PQ storage and, in non-root/OpenShift setups, for writable storage—combine them as needed and see the chart README and values for `criblVolumeDir` / `extraVolumeMounts`.

Documentation notes that persistent queuing in Kubernetes is possible but behavior can vary with storage and scaling; extra care is needed when combining PQ with autoscaling and non-root/OpenShift patterns.
