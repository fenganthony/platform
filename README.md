
# hello-app Helm Chart

## Install

```bash
helm install my-hello ./hello-app
```

Override image and ingress:

```bash
helm install my-hello ./hello-app \
  --set image.repository=quasooony/hello-app,image.tag=latest \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=hello.host.com \
  --set ingress.hosts[0].paths[0].path=/hello
```

Enable JMX exporter and ServiceMonitor:

```bash
helm upgrade --install my-hello ./hello-app \
  --set jvmExporter.enabled=true \
  --set jvmExporter.crdInstall=true
```

## Values (key highlights)

- `image.repository` (string) — Docker image repo
- `image.tag` (string) — Image tag
- `resources` (object) — CPU/memory requests & limits
- `ingress.enabled` (bool) — Create an Ingress for /hello
- `service.type` (string) — ClusterIP, NodePort, LoadBalancer
- `probes.enabled` (bool) — Enable readiness/liveness (default true)
- `jvmExporter.enabled` (bool) — Enable JMX exporter sidecar
- `jvmExporter.crdInstall` (bool) — Create ServiceMonitor (Prometheus Operator CRD must be installed)
