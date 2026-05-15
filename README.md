🔭 OpenTelemetry Astronomy Shop — DevOps Implementation

A production-grade observability setup built on top of the OpenTelemetry Demo by the CNCF community.
This repo documents my hands-on implementation — including Kubernetes deployment, CI/CD pipeline, monitoring stack, and distributed tracing across 15+ polyglot microservices.


📌 What This Project Is
The OpenTelemetry Astronomy Shop is a microservice-based e-commerce application (selling astronomy equipment) designed to demonstrate real-world observability using OpenTelemetry.
I deployed and configured this end to end on AWS EKS, wired up a Jenkins CI/CD pipeline, and set up the full Prometheus + Grafana + Jaeger observability stack — treating it like a real production system.

🏗️ Architecture
                        ┌─────────────────────────────────────────────────────┐
                        │                  User / Browser                     │
                        └─────────────────────┬───────────────────────────────┘
                                              │
                        ┌─────────────────────▼───────────────────────────────┐
                        │            Frontend Proxy (Envoy)                   │
                        │              Traffic routing layer                  │
                        └──────────┬──────────────────────────┬───────────────┘
                                   │                          │
               ┌───────────────────▼──────┐    ┌─────────────▼───────────────┐
               │   Frontend (TypeScript)  │    │  Load Generator (Locust)    │
               └───────────────────┬──────┘    └─────────────────────────────┘
                                   │
        ┌──────────────────────────▼──────────────────────────────────────────┐
        │                  Application Microservices                          │
        │                                                                     │
        │  ┌────────────┐ ┌──────────────┐ ┌──────────┐ ┌────────────────┐  │
        │  │  Checkout  │ │Product Catalog│ │   Cart   │ │    Payment     │  │
        │  │    (Go)    │ │    (Go)      │ │  (.NET)  │ │  (JavaScript)  │  │
        │  └────────────┘ └──────────────┘ └──────────┘ └────────────────┘  │
        │                                                                     │
        │  ┌────────────┐ ┌──────────────┐ ┌──────────┐ ┌────────────────┐  │
        │  │  Shipping  │ │   Currency   │ │  Email   │ │ Recommendation │  │
        │  │   (Rust)   │ │    (C++)     │ │  (Ruby)  │ │   (Python)     │  │
        │  └────────────┘ └──────────────┘ └──────────┘ └────────────────┘  │
        │                                                                     │
        │  ┌────────────┐ ┌──────────────┐ ┌──────────┐ ┌────────────────┐  │
        │  │ Ad Service │ │Fraud Detection│ │Accounting│ │     Quote      │  │
        │  │   (Java)   │ │   (Kotlin)   │ │  (.NET)  │ │    (PHP)       │  │
        │  └────────────┘ └──────────────┘ └──────────┘ └────────────────┘  │
        │                                                                     │
        │            ┌────────────────────────────────┐                      │
        │            │     Kafka Message Queue         │                      │
        │            │   Async service communication   │                      │
        │            └────────────────────────────────┘                      │
        └──────────────────────────┬──────────────────────────────────────────┘
                                   │ OTLP (gRPC 4317 / HTTP 4318)
        ┌──────────────────────────▼──────────────────────────────────────────┐
        │                     OTel Collector                                  │
        │           Receives · Processes · Exports telemetry                  │
        └────────┬──────────────┬──────────────────┬──────────────┬───────────┘
                 │              │                  │              │
        ┌────────▼───┐  ┌───────▼──────┐  ┌───────▼──────┐  ┌───▼──────────┐
        │   Jaeger   │  │  Prometheus  │  │  OpenSearch  │  │   Grafana    │
        │  (Traces)  │  │  (Metrics)   │  │   (Logs)     │  │ (Dashboards) │
        └────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

🛠️ Tech Stack
CategoryToolsOrchestrationKubernetes (AWS EKS)Service Mesh / ProxyEnvoyInstrumentationOpenTelemetry SDKs (10 languages)Telemetry PipelineOpenTelemetry Collector (OTLP)Distributed TracingJaegerMetricsPrometheusLog AggregationOpenSearchDashboardsGrafanaMessage QueueApache KafkaCacheValkey (Redis-compatible)Feature FlagsFlagdCI/CDJenkins + GitHubInfrastructureTerraform, AWS (EKS, VPC, IAM)Load TestingLocust

⚙️ Setup & Deployment
Prerequisites
Make sure the following are installed on your local machine:
bash# Verify each tool is installed
kubectl version --client
helm version
aws --version
terraform version
docker version
ToolVersionInstallkubectl>= 1.28sudo snap install kubectl --classicHelm>= 3.0curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bashAWS CLI>= 2.0pip install awscliTerraform>= 1.6sudo apt install terraformDocker>= 24.0sudo apt install docker.io

Step 1 — Clone this repo
bashgit clone https://github.com/I-am-MaazAnsari/otel-astronomy-shop.git
cd otel-astronomy-shop

Step 2 — Provision EKS cluster with Terraform
bashcd terraform/
terraform init
terraform validate
terraform plan
terraform apply
After apply completes, update your kubeconfig:
bashaws eks update-kubeconfig --name otel-demo-cluster --region us-east-1
kubectl get nodes   # verify nodes are Ready

Step 3 — Deploy the application with Helm
bash# Add the OpenTelemetry Helm repo
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

# Create namespace
kubectl create namespace otel-demo

# Deploy the full application
helm install otel-demo open-telemetry/opentelemetry-demo \
  --namespace otel-demo \
  --values helm/my-values.yaml

Step 4 — Verify all pods are running
bashkubectl get pods -n otel-demo

# Expected output — all pods should show Running status
# NAME                                      READY   STATUS    RESTARTS
# otel-demo-frontend-xxx                    1/1     Running   0
# otel-demo-checkout-xxx                    1/1     Running   0
# otel-demo-productcatalog-xxx              1/1     Running   0
# otel-demo-otelcol-xxx                     1/1     Running   0
# ...

Step 5 — Access the services
bash# Frontend (the shop UI)
kubectl port-forward svc/otel-demo-frontend 8080:8080 -n otel-demo

# Jaeger (distributed traces)
kubectl port-forward svc/otel-demo-jaeger-query 16686:16686 -n otel-demo

# Grafana (dashboards)
kubectl port-forward svc/otel-demo-grafana 3000:3000 -n otel-demo

# Prometheus (metrics)
kubectl port-forward svc/otel-demo-prometheus 9090:9090 -n otel-demo
ServiceURLCredentialsShop Frontendhttp://localhost:8080-Jaeger UIhttp://localhost:16686-Grafanahttp://localhost:3000admin / adminPrometheushttp://localhost:9090-

Step 6 — Trigger a test failure using Feature Flags
One of the most powerful features of this demo is the ability to simulate production failures:
bash# Access the feature flag UI
kubectl port-forward svc/otel-demo-flagd 8013:8013 -n otel-demo
Open http://localhost:8013 and enable productCatalogFailure.
Now open Jaeger at http://localhost:16686 and watch the trace show the failure propagating from the Product Catalog service through to the Checkout service in real time.

📊 Observability Highlights
Distributed Tracing in Jaeger
Every user request generates a trace that shows the full journey across services.
Example — a single "Place Order" action creates spans across:
Frontend → Checkout → Product Catalog → Cart → Currency → Payment → Shipping → Email
Each span shows duration, status, and any errors. If one service is slow, you can pinpoint exactly which one without guessing.
Metrics in Prometheus + Grafana
Key metrics collected:

http_server_duration_milliseconds — request latency per service
rpc_server_requests_total — total gRPC requests between services
process_cpu_seconds_total — CPU usage per pod
kafka_consumer_lag — message queue depth

Log Aggregation in OpenSearch
All service logs are shipped to OpenSearch via the OTel Collector pipeline.
Search across all 15+ services in one place.

🔄 CI/CD Pipeline (Jenkins)
Developer pushes code
        ↓
Jenkins detects push via GitHub webhook
        ↓
Jenkins builds Docker image (tagged with git commit SHA)
        ↓
Image pushed to Docker Hub
        ↓
Kubernetes manifests updated with new image tag
        ↓
kubectl apply rolls out the update to EKS
        ↓
Prometheus detects new pod, begins scraping metrics
Jenkins pipeline file is at Jenkinsfile in the root of this repo.

🔐 Security Considerations

All secrets managed via Kubernetes Secrets (base64 encoded, KMS encrypted at rest)
IAM Roles for Service Accounts (IRSA) used for pod-level AWS access
Network Policies restrict inter-pod communication to required paths only
Security Groups on EKS node groups limit external traffic to ports 80/443


📁 Repository Structure
otel-astronomy-shop/
├── terraform/              # EKS cluster + VPC provisioning
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── modules/
│       ├── eks/
│       └── vpc/
├── helm/
│   └── my-values.yaml      # Custom Helm values for the demo
├── k8s/
│   ├── namespace.yaml
│   ├── network-policy.yaml
│   └── prometheus-rules.yaml
├── jenkins/
│   └── Jenkinsfile
├── docs/
│   └── architecture.png    # Architecture diagram
└── README.md

🧠 Key Learnings

Why the OTel Collector exists — without it, every service would need a direct connection to every backend (Jaeger, Prometheus, etc). The collector centralises this — one endpoint for all services, fan-out to all backends.
Traces vs Metrics vs Logs — they answer different questions. Traces tell you where something went wrong. Metrics tell you how bad it is. Logs tell you what exactly happened.
Kafka's role — async communication between services like Checkout → Accounting means the checkout doesn't block waiting for accounting to finish. The trace still connects them end to end via trace context propagation.
Feature flags for incident simulation — being able to trigger a controlled failure and trace it in Jaeger is the closest thing to a real incident drill you can do safely.


🙏 Credits
This project is based on the OpenTelemetry Demo maintained by the CNCF OpenTelemetry community.
My contributions include the Terraform EKS provisioning, Jenkins CI/CD pipeline, custom Helm values, Prometheus alert rules, and all documentation in this repo.

📬 Contact
Maaz Ansari — DevOps Engineer
📧 maazansari.86@gmail.com
🔗 LinkedIn
💻 GitHub
