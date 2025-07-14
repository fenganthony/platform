# 🧪 Platform Engineer Take-Home Assignment – Java Lock Simulation & Helm Observability

## 📌 Objective

This take-home assignment is designed to evaluate your ability to deploy, monitor, and operate a Java-based application on Kubernetes using Helm and diagnostic tools. You will simulate a production scenario where a service becomes unresponsive due to internal thread contention. Your task is to make it observable, recoverable, and operationally safe.

---

## 🧱 Provided Application

You are given a prebuilt Java Spring Boot application (or can build it yourself from `Dockerfile`) with the following behavior:

- Exposes a simple `/hello` endpoint on port `8080`
- After **100 seconds**, a background thread acquires a lock and enters an **infinite wait**
- All future `/hello` requests become **blocked** due to synchronization on the same lock
- The app **does not crash**, and logs show no error
- The issue is only observable via degraded latency or hanging requests

---

## ✅ Your Tasks

### 1️⃣ Helm Chart Setup (**45 points**)

Design and submit a Helm chart that includes:

- `Deployment.yaml` with proper probes and resource settings
- `Service.yaml` to expose the application
- `Ingress.yaml` for `/hello` routing
- `values.yaml` to allow configurability
- Liveness/readiness probes that eventually detect `/hello` is unresponsive and restart the container

Your values.yaml must support configuration flexibility. The following are required or recommended:

| Key                      | Type   | Description                                                                  |
| ------------------------ | ------ | ---------------------------------------------------------------------------- |
| `image.repository`       | string | Docker image repo                                                            |
| `image.tag`              | string | Image version                                                                |
| `resources`              | object | CPU/memory requests & limits                                                 |
| `ingress.enabled`        | bool   | Whether to create an ingress                                                 |
| `service.type`           | string | ClusterIP, NodePort, or LoadBalancer                                         |
| `probes.enabled`         | bool   | Whether to enable liveness/readiness probes                                  |
| `jvmExporter.enabled`    | bool   | Whether to enable jmx-exporter sidecar (if implemented)                      |
| `jvmExporter.crdInstall` | bool   | ⚠️ If jmx-exporter CRD is required (e.g. for ServiceMonitor), allow toggling |

---

### 2️⃣ Pre-Restart Thread Dump Capture (**20 points**)

Before the container is restarted, you must:

- Use `jstack` (or equivalent) to dump the JVM thread stack
- Save the output to the **Kubernetes node's local disk**
- Path format (example): `/var/log/java-dump/<pod-name>.log`

---

### 3️⃣ Basic JVM Observability (**15 bonus points**)

Enhance observability by:

- Adding `jmx_exporter` or similar
- Exposing JVM thread/GC metrics via Prometheus
- Showing how alerts can be defined for thread blocking

---

### 4️⃣ CI/CD Pipeline (**20 bonus points**)

Create a GitHub Actions workflow that:

- Builds the Java app into a Docker image
- Pushes it to Docker Hub or GitHub Container Registry
- (Optional) Validates Helm templates or performs dry-run deploys

> Place your workflow in `.github/workflows/pipeline.yml`

---

## 🔍 Reasonability Review

All requirements are considered **realistic and production-relevant** for a Platform Engineer:

| Area                    | Justification                                                                 |
|-------------------------|-------------------------------------------------------------------------------|
| Helm templating         | Industry standard for Kubernetes deployments                                  |
| Probes & resource limits| Essential for platform-managed app health and autoscaling                    |
| Pre-restart diagnostics | Required in real-world SRE/on-call workflows                                 |
| JVM observability       | Crucial for diagnosing GC/Thread contention in production Java systems        |
| CI/CD pipeline          | Demonstrates automation readiness and delivery process awareness              |

---

## 📝 Deliverables

1. A GitHub repository containing:
   - Helm chart (`charts/java-app/`)
   - Sample `values.yaml`
   - Optional: `.github/workflows/` CI/CD workflow
   - README explaining how to test, deploy, and analyze
2. Logs and dumps outputted by the app `logs/<pod-name>.log`
3. A short `SUMMARY.md` with:
   - Your design decisions
   - Any assumptions or limitations
   - What you’d improve in a production version

---

## 👤 Notes for Non-Java Engineers

If you're not familiar with Java debugging tools:

- Most JDKs support:
  ```bash
  jps           # Lists JVM PIDs
  jstack <pid>  # Dumps thread state
  ```

- If unavailable, you may:
  - Use `/proc/<pid>/stack`
  - Use a debugging sidecar container with tools installed
  - Inject `openjdk` tooling via a custom image

Clearly explain your intent if you're unable to implement full tooling — **communication and reasoning are more important than completeness**.

## 🧠 Scoring Summary

| Section                                  | Points |
|------------------------------------------|--------|
| Helm chart (Deployment, Service, Ingress)| 45     |
| Pre-restart stack dump to node log       | 20     |
| JVM observability (Prometheus, exporter) | +15    |
| CI/CD pipeline (GitHub Actions)          | +20    |
| **Total**                                | **100+** |

---

## 🚀 Testing Instructions

```bash
# After deploying via Helm:
curl http://<your-ingress-host>/hello

# ✅ Initial behavior:
# You should see: "Hello World"

# 🕒 After ~100 seconds:
# The endpoint will hang due to a background thread holding a global lock.
# You should begin seeing timeouts or delayed responses.

# 🧪 Next:
# - Use jstack to inspect thread state
# - Save the output to /var/log/java-dump/<pod-name>.log on the node
# - Observe probe-triggered restarts if implemented
```

---
## 🍀 Good luck!

We're not evaluating how well you write Java —  
We're evaluating **how well you deploy, diagnose, and recover black-box applications in production**.

Show us your thought process, observability design, and operational maturity.

We’re excited to see your engineering judgment in action!
