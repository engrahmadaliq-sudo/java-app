# Java App — Full Deployment & Monitoring Pipeline

GitHub → Jenkins → Docker → Kubernetes → HAProxy → Prometheus/Grafana/Nagios

## Structure
```
.
├── pom.xml                    # Maven build config (Spring Boot + Actuator + Micrometer)
├── src/                       # Sample Spring Boot app
├── Dockerfile                 # Multi-stage build
├── Jenkinsfile                 # CI/CD pipeline
├── k8s/
│   ├── deployment.yaml        # K8s Deployment (3 replicas, probes, resource limits)
│   ├── service.yaml           # NodePort service
│   └── hpa.yaml               # Autoscaling based on CPU/memory
├── haproxy/
│   ├── haproxy.cfg            # Load balancer config in front of K8s nodes
│   └── docker-compose.yml     # Runs HAProxy + haproxy_exporter
└── monitoring/
    ├── prometheus.yml         # Scrape config (app pods + HAProxy)
    ├── alert_rules.yml        # Alerting rules
    ├── kube-prometheus-stack-values.yaml  # Helm values for full monitoring stack
    └── nagios-checks.cfg      # Host-level infra checks
```

## Setup Steps

### 1. Push this repo to GitHub
```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/yourusername/java-app.git
git push -u origin main
```
Add a webhook: GitHub repo → Settings → Webhooks → `http://<jenkins-ip>:8080/github-webhook/`

### 2. Jenkins
- Install plugins: Git, Pipeline, Docker Pipeline, Kubernetes CLI, Credentials Binding
- Add credentials:
  - `docker-hub-creds` (username/password) for Docker Hub
  - `kubeconfig` (secret file) — your cluster's kubeconfig
- Create a new Pipeline job pointing at this repo, using the `Jenkinsfile`
- Edit `IMAGE_NAME` in the Jenkinsfile to your Docker Hub username

### 3. Kubernetes
```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml
kubectl get pods -w
```
Update the image name in `k8s/deployment.yaml` to match your registry.

### 4. HAProxy
Edit `haproxy/haproxy.cfg` — replace `10.0.0.1/2/3` with your actual K8s node IPs.
```bash
cd haproxy
docker-compose up -d
```
Check stats: `http://<haproxy-ip>:8404/stats` (user: admin / pass: changeme123 — change this)

### 5. Monitoring
Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager) via Helm:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f monitoring/kube-prometheus-stack-values.yaml \
  -n monitoring --create-namespace
```
Get Grafana URL/password:
```bash
kubectl get svc -n monitoring
# default admin password set in the values file: changeme123
```
Import Grafana dashboard IDs:
- **4701** — JVM (Micrometer)
- **12693** — HAProxy

Deploy the HAProxy exporter (already included in `haproxy/docker-compose.yml`) so Prometheus can scrape LB metrics.

Apply alert rules by adding `monitoring/alert_rules.yml` as a `PrometheusRule` CRD, or reference it directly if running Prometheus standalone (not via Helm).

Nagios: copy `monitoring/nagios-checks.cfg` into `/usr/local/nagios/etc/objects/` on your Nagios server and restart the service. Install NRPE agents on each monitored host.

## End-to-End Flow
```
git push → GitHub webhook → Jenkins (build/test/dockerize/push) →
kubectl deploy (rolling update) → HAProxy health-checks & load-balances →
Prometheus scrapes app + HAProxy → Grafana dashboards →
Alertmanager → Slack/email → Nagios watches host-level health
```

## Rollback
```bash
kubectl rollout undo deployment/java-app-deployment
kubectl rollout history deployment/java-app-deployment
```

## Notes / Things to customize before production use
- Replace all placeholder IPs, image names, and credentials.
- Enable TLS on HAProxy frontend (cert-manager or manual cert).
- Move secrets (Docker creds, kubeconfig, Slack webhook) into Kubernetes Secrets / Jenkins Credentials — never commit them.
- Consider Ingress-NGINX or a cloud LB instead of/alongside HAProxy depending on where your cluster runs (on-prem vs cloud).
# java-app
