# 🛠️ DevOps Technical Task: Simulated EKS-Style Secure Deployment Using Minikube

This project simulates a secure Kubernetes-based microservices environment using **Minikube**, modeled after real-world **AWS EKS production** practices. It includes a working gateway, internal-only services, MinIO as a mock S3 store, and full observability and security policies.

---

## 📌 Setup Instructions

### ✅ Prerequisites

Install the following tools:

- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [Docker](https://docs.docker.com/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)
- (Optional but useful): `k9s`, `OPA`, `Kyverno`, `Falco`

### ✅ Minikube Cluster Setup

```bash
minikube start --cpus=2 --memory=4096 --addons=ingress,metrics-server
```

#### ✅ Add Prometheus Community Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

---

## ⚠️ Part 1: Minikube Gateway Access Issue

**Problem**: NodePort / Ingress Not Accessible on Windows (Docker Driver)

When running Minikube with the Docker driver on Windows, the following access URLs do not work:

- `http://192.168.49.2:30080` (NodePort)
- `http://gateway.192.168.49.2.nip.io` (Ingress)

### ✅ Working Solution:

Only the following command works to access the Gateway service:

```bash
minikube service gateway -n application --url
```

➡️ It returns something like `http://127.0.0.1:53505`, which works because Minikube sets up a local proxy tunnel between your host (Windows) and the Kubernetes cluster.

**Why Others Don’t Work?**

Minikube’s Docker driver does not expose NodePort or Ingress IP to the host directly, unlike the VirtualBox driver.

**Alternate Option**:
Run:

```bash
minikube tunnel
```

This will route LoadBalancer-type services and ingress traffic correctly — but requires admin access and runs in a separate terminal.

---

## 🧱 Architecture Overview

```
                    +-----------------------------+
                    |      Gateway (Ingress)      |
                    |    (nginxdemos/hello)       |
                    +--------------+--------------+
                                   |
             +--------------------+--------------------+
             |                                         |
+------------------------+              +------------------------+
|     Auth Service       |              |     Data Service       |
| (kennethreitz/httpbin) |              | (hashicorp/http-echo)  |
+------------------------+              +------------------------+
             |                                         |
             |                                         |
       (Internal only)                        +----------------+
                                              |     MinIO      |
                                              | (Mocked S3 API)|
                                              +----------------+
```

---

## 🎯 Task Objectives

### 🔹 Part 1: Microservice Stack

- Gateway is exposed externally via Ingress.
- `auth-service` and `data-service` are internal-only.
- All services are deployed in the `application` namespace.
- Liveness and readiness probes configured.
- Proper resource requests/limits are set.

---

### 🔹 Part 2: Simulate IAM with MinIO

- MinIO deployed using `minio/minio` image.
- Created a Kubernetes Secret to store MinIO credentials.
- `data-service` has a ServiceAccount that can access MinIO.
- `auth-service` is restricted and cannot access MinIO.
- **Bonus**: Enforced this policy using OPA/Kyverno.

**Debugging Note:**

To verify MinIO access from inside the cluster:

```bash
kubectl run -i --tty awscli --image=amazon/aws-cli --restart=Never -- sh
```

Inside the pod:

```bash
aws s3 ls --endpoint-url http://minio.application.svc.cluster.local:9000
```

For external access (from host):

```bash
kubectl port-forward svc/minio -n application 9000:9000
aws s3 ls --endpoint-url http://localhost:9000
```

---

### 🔹 Part 3: Security Incident Simulation

📜 **Scenario**: `auth-service` logs sensitive headers and can access unauthorized services.

### ✅ Fixed:

- Used `/headers` endpoint of HTTPBin to simulate leak.
- Found logs using:

```bash
kubectl logs <auth-pod-name> -n application
```

- Removed logging of Authorization header.
- Applied `NetworkPolicy`:
  - `auth-service` can only talk to `data-service`.
  - All egress is blocked unless explicitly allowed.

**Bonus**: Used OPA to detect if any pod logs `Authorization` headers.

---

### 🔹 Part 4: Observability with Prometheus & Grafana

Installed via Helm:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack
```

**Dashboard tracks**:

- Pod CPU/memory usage
- HTTP request counts
- Pod restarts & failed probes



---

### 🔹 Part 5: Failure Simulation

✅ Simulated pod failure:

```bash
kubectl delete pod <pod-name>
```

- Observed recovery using ReplicaSets.
- Confirmed alerts and Grafana metrics showed the failure spike.



## ✅ Known Issues & Assumptions

- NodePort access doesn't work due to Docker driver limitations. Fixed by using `minikube service --url`.
- MinIO IAM simulation is scoped to namespace-level restrictions.
- Grafana dashboards are preconfigured using Prometheus metrics only.
