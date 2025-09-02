# hello-app · Deploy, Test, Analyze

This repo contains a Java/Spring demo app and a Helm chart (`./java-app`) that
shows how to operate a “black-box” service safely: liveness/readiness probes,
JMX metrics, and *pre-restart thread dumps* for incident analysis.

---

## What’s here

- **App**: simple Spring Boot service exposing `GET /hello` on port `8080`.
  After ~100s, a background thread grabs a lock and blocks future requests (no crash).
- **Helm chart**: [`./java-app`](./java-app)
  - Probes pre-configured (readiness fails first, then liveness restarts)
  - Optional **JMX Exporter** sidecar (`jvmExporter.enabled=true`)
  - Optional **ServiceMonitor** for Prometheus Operator
  - Optional **preRestartDump** to collect `jstack` before termination
- **CI**: GH Actions pipeline (GHCR only) builds/pushes image and can
  run `helm lint/template` (offline) and an optional cluster dry-run.

---

## Quick test (local)

### 1) Build and run with Docker
```bash
# From repo root
docker build -t ghcr.io/fenganthony/spring-hello-problem:dev .
docker run --rm -p 8080:8080 ghcr.io/fenganthony/spring-hello-problem:dev
```
or
| Command           | Description                                                |
|------------------|------------------------------------------------------------|
| `make build`     | Build the Docker image using the specified image name and tag |
| `make run`       | Run the container locally, exposing it on port 8080         |
| `make stop`      | Stop the running container                                  |
| `make logs`      | Stream logs from the container                              |
| `make push`      | Push the built image to your Docker registry                |
| `make clean`     | Remove the local Docker image                               |

> Default image: `ghcr.io/fenganthony/spring-hello-problem:1.0.0`  
> Exposed port: `8080` refer from the instruction

### 2) Test Locally

```bash
curl http://localhost:8080/hello
```
If you see a response like `{"message":"Hello, World!"}`, then the service is working correctly.
Note that this works only because we have the Docker container running and accessible on `localhost:8080`.

## Deploy with Helm (recommend)
Requirements: kubectl, helm, and a cluster context (e.g. minikube, kind, or a cloud provider).

### 0) Choose the image you want to deploy
From CI: ghcr.io/fenganthony/spring-hello-problem:<tag>
default branch pushes use v1.0.0 (configurable in workflow)
manual runs can set version, tag pushes (vX.Y.Z) publish the same tag

Or build locally and push your own tag to GHCR.

### 1) Deploy the Helm chart
Installation:
```bash
helm upgrade --install {app_name_you_preferred} ./java-app
```

With Ingress Controller:
```bash
helm upgrade --install {app_name_you_preferred} ./java-app \
  --set image.repository={url_to_repo, defaultly ghcr.io/fenganthony/spring-hello-problem},image.tag= <version_tag, defaultly v1.0.0> \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host= {ingress_host_domain, by default example.host.com} \
  --set ingress.hosts[0].paths[0].path=/hello
```
Note: The cluster must have an Ingress Controller installed and configured to use this feature.

Enable JMX exporter and ServiceMonitor:
```bash
helm upgrade --install {app_name_you_preferred} ./java-app \
  --set jvmExporter.enabled=true \
  --set jvmExporter.crdInstall=true \
  --set jvmExporter.mode={sidecar/agent, defaultly sidecar}
```
Note: 
1. Such configurations help in monitoring the JVM metrics and managing the application effectively, but need to have Prometheus Operator CRDs exist in your cluster context.
2. jvmExporter mode is a self-defined variable, with sidecar mode, a Prometheus httpsserver created to collect metrics from the application; agent mode is to run the JMX exporter as a Java agent within the application instead.
3. jvmExporter will expose the metrics at `/metrics` endpoint, 9404 port by default.

A simple table of configurable parameters (more variables could be seen in the values.yaml):
| Key                      | Type   | Purpose                                          |
| ------------------------ | ------ | ------------------------------------------------ |
| `image.repository`       | string | Docker image repo (default uses GHCR fixed name) |
| `image.tag`              | string | Version tag (e.g. `v1.0.0`)                      |
| `ingress.enabled`        | bool   | Creates an Ingress for `/hello`                  |
| `probes.enabled`         | bool   | Readiness & liveness probes                      |
| `jvmExporter.enabled`    | bool   | Expose metrics via JMX exporter                  |
| `jvmExporter.crdInstall` | bool   | Create `ServiceMonitor` if CRDs exist            |
| `jvmExporter.mode`       | string | JMX exporter mode (sidecar/agent)                |
| `preRestartDump.enabled` | bool   | Capture thread dumps pre-termination             |

### 2) Testing & Metric Accessing

For Main Service:
From Ingress Controller:
```bash
curl http://<ingress_host>/hello
```
From kubectl (Only for testing locally, w/o Ingress Controller):
```bash
kubectl -n default port-forward <svc/java-app> 8080:8080
curl http://localhost:8080/hello
```

For Pod's metrics access:

From kubectl:
```bash
kubectl -n default port-forward <svc/java-app> 9404:9404
curl http://localhost:9404/metrics
```
(to be implemented) Could potentially also be accessed via Ingress Controller
```bash
curl http://<ingress_host>/metrics
```

## Analyze & Debug

### A) Pre-restart thread dumps (recommended)
This procedure can be triggered in either:
#### 1. Automatic mode
preRestartDump.enabled=true adds a tiny jdk-tools container and a preStop
hook that runs a dump script before termination. Dumps are written to the node
via hostPath (default /var/log/java-dump/<pod-name>.log).

Enable it (it’s enabled by default in values, but if not, you can enable it manuall):
```bash
helm upgrade --install {app_name_you_preferred} ./java-app \
  --set preRestartDump.enabled=true \
  --set preRestartDump.hostPathDir=/var/log/java-dump
```
When triggering a restart (e.g., scale to 0 or kill a Pod, or from readiness probe failure (our main concern)), the dump is automatically fetched:

SSH to the node where the k8s cluster ran and read /var/log/java-dump/<pod>.log

#### 2. Manual mode
Exec into the jdk-tools container and run the script on demand, for example:

```bash
kubectl -n default exec -it deploy/java-app -c jdk-tools -- /bin/sh
/dump/dump.sh
```

### B) Classic logs & probes

If no thread dump is available, you can check the standard logs:

```bash
kubectl logs -n default deploy/java-app -c app --timestamps

kubectl describe pod -n default -l app.kubernetes.io/name=hello-app
```
Watch readiness turn false, then liveness restart the Pod.

### C) Metrics (Prometheus)
With jvmExporter.enabled=true, scrape <prometheus_server>/metrics can also see some logs. (In current implementation, it may not include all application logs, but more likely to be a scalable solution)
