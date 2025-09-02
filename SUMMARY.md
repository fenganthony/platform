# SUMMARY

## Design decisions
- **Fixed image name (GHCR)**: `ghcr.io/fenganthony/spring-hello-problem` to decouple deployments from repo renames and keep manifests stable.
- **Clear versioning model**: default tag **`v1.0.0`** on branch pushes; manual runs can override via `version`; pushing `vX.Y.Z` creates the same image tag.
- **Offline Helm validation by default**: CI runs `helm lint` + `helm template` so it never fails due to missing cluster access; a **cluster dry-run** can be enabled via inputs when a kubeconfig is provided.
- **JMX exporter modes**:
  - **Sidecar (httpserver)** by default, connecting to the app’s JMX port (Pod-IP substituted at runtime to keep RMI stable).
  - **Agent mode** available behind a values flag to run the exporter inside the JVM when desired. (currently not compatible with sidecar mode due to hostport setting conflict, may need to disable it when using agent mode)
- **Pre-restart diagnostics**: lightweight `jdk-tools` sidecar + `preStop` hook captures `jstack`/`jcmd` before termination for incident forensics.
- **Config-first chart**: all knobs live in `values.yaml`. Required settings are documented; a few future-facing options are scaffolded for growth.

## Assumptions / limitations
- **Cluster access**: CI has no default kubeconfig; cluster dry-run only runs when launched manually with a provided base64 kubeconfig input.
- **Prometheus Operator**: `ServiceMonitor` creation assumes CRDs are present; otherwise scrape the Service/Pod directly.
- **Ingress**: host/class are environment-specific. On-prem dev uses `minikube tunnel`.
- **Probes**: readiness/liveness thresholds are fixed today; they’re tunable via `values.yaml`.
- **Thread dumps storage**: dumps land on a hostPath; fetching them typically requires node access (or run the dump script inside the Pod on demand).
- **Image build**: single-arch (`linux/amd64`) unless Buildx multi-arch is enabled.
- **Pinned tooling**: provider/image versions are fixed for reproducibility. For example, swapping JMX exporter distributions may require chart adjustments, making it relatively difficult to upgrade.
- **Service address allocation issue**: when using the sidecar mode, there may be address allocation issues due to the way the JMX exporter connects to the application, still need to address this in the future.
- **Single environment**: all resources are deployed into the `default` namespace; no separate staging/production environments.

## Production improvements
- **Security & supply chain**: SBOM/provenance (e.g., `syft`, `cosign`), image signing, non-root runtime, pinned base images.
- **Multi-arch & caching**: enable `linux/amd64, linux/arm64` builds and tune Buildx cache.
- **Stronger tests**: Helm unit tests (`helm-unittest`), template tests, and smoke tests.
- **Observability**: structured logs, app metrics, dashboards, and alerts (e.g., blocked threads, restarts).
- **Rollout strategy**: canary/blue-green or progressive delivery (Argo Rollouts/Flagger).
- **Cluster auth**: OIDC to managed clusters instead of embedding kubeconfigs.
- **Online/Offline workflows**: formalize both paths so the same chart passes offline CI and online pre-deploy checks.
- **Elastic choice of repository**: clearly define the criteria for choosing between public and private repositories for storing Helm charts, considering factors like security, accessibility, and compliance.
- **Documentation & examples**: improve documentation and provide more examples to help users understand how to use the Helm chart effectively.