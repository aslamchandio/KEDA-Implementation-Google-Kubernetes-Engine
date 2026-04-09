# Event‑Driven Autoscaling on GKE using KEDA (Production‑Ready Implementation)

## Project Overview

This project demonstrates a **production‑grade implementation of KEDA (Kubernetes Event‑Driven Autoscaling)** on a **Google Kubernetes Engine (GKE) Private Cluster** using modern cloud‑native best practices.

Unlike traditional **Horizontal Pod Autoscaler (HPA)** which scales only on CPU/Memory, **KEDA enables autoscaling based on external event sources** such as:

* Google Pub/Sub
* Prometheus Metrics
* Kafka
* Redis
* HTTP queue length
* Custom external metrics

This architecture enables **real-world event-driven microservices scaling** suitable for production workloads.

---

## Key Features of This Implementation

✔ Private GKE Cluster compatible
✔ Workload Identity enabled (no service account keys)
✔ Helm-based KEDA installation
✔ Event-driven autoscaling via Pub/Sub
✔ Prometheus metric-based scaling example
✔ Automatic HPA creation via KEDA
✔ Secure IAM integration
✔ Terraform-ready modular structure
✔ Production-grade best practices included

---

## Architecture Diagram

```
                Google Cloud Project
                        |
        ---------------------------------------
        |                                     |
   VPC Network                          Pub/Sub Topic
        |                                     |
        |                              Subscription Queue
        |
   Private GKE Cluster
        |
        |---- KEDA Operator
        |---- KEDA Metrics Adapter
        |
        |---- Application Deployment
                |
                |---- ScaledObject
                |
                |---- External Metrics Trigger

Optional:
Prometheus / Managed Service for Prometheus
Cluster Autoscaler
Cloud Monitoring
```

---

## Repository Structure

```
.
├── k8s/
│   ├── deployment.yaml
│   ├── scaledobject.yaml
│
├── helm/
│   └── keda-install.sh
│
├── terraform/
│   ├── gke-private-cluster
│   ├── workload-identity
│   ├── pubsub
│   └── monitoring
│
└── README.md
```

---

## Prerequisites

Install required tooling:

```
Google Cloud SDK
kubectl
Helm v3+
Terraform (optional but recommended)
```

Required IAM roles:

```
roles/container.admin
roles/iam.serviceAccountUser
roles/pubsub.subscriber
roles/monitoring.viewer
```

---

## Step 1: Authenticate to GCP

```
gcloud auth login
```

Set project:

```
gcloud config set project PROJECT_ID
```

Connect cluster:

```
gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION
```

Verify access:

```
kubectl get nodes
```

Update cluster with Gateway API:

```
gcloud container clusters update CLUSTER_NAME \
    --location=CLUSTER_LOCATION\
    --gateway-api=standard
```
---

## Step 2: Install KEDA using Helm

Add Helm repo:

```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm repo list
```

Create namespace & Install KEDA:

```
helm install keda kedacore/keda \
  --namespace keda
```

Verify installation:

```
kubectl get pods -n keda
```

Expected components:

```
keda-operator
keda-metrics-apiserver
```

```
gcloud projects describe $PROJECT_ID --format 'get(projectNumber)'

gcloud projects list

echo $PROJECT_NUMBER

gcloud projects add-iam-policy-binding projects/${PROJECT_ID} \
     --role roles/monitoring.viewer \
     --member=principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${PROJECT_ID}.svc.id.goog/subject/ns/keda/sa/keda-operator

k get all -n keda

```

---

## Step 3: Deploy Sample Application with Cron

Save this as deployment.yaml or use the file in the 01-keda-for-cronjob folder::

```
apiVersion: apps/v1
kind: Deployment
metadata:
  #Dictionary
  name: myapp4-deployment
spec:
  # Dictionary
  replicas: 0
  selector:
    matchLabels:
      app: myapp4
  template:
    metadata:
      # Dictionary
      name: myapp4-pod
      labels:
        # Dictionary
        app: myapp4 # Key value pairs
    spec:
      containers:
      # List
      - name: myapp4-container
        image: aslam24/nginx-web-makaan:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "20Mi" #"256Mi" # 128 MebiByte is equal to 135 Megabyte (MB)
            cpu: "20m" #"250m" # `m` means milliCPU
          limits:
            memory: "30Mi" #"512Mi"
            cpu: "30m" #"400m" # 1000m is equal to 1 VCPU core 
---

apiVersion: v1
kind: Service
metadata:
  name: myapp4-service
spec:
  type: ClusterIP # ClusterIP, # NodePort # LoadBalancer
  selector:
    app: myapp4
  ports:
  - name: http
    port: 80 # Service Port
    targetPort: 80 # Container Port

--- 

apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: mygateway1-global-staticip
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: http
    protocol: HTTP
    port: 80

--- 

apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: route-external-global-http-staticip
spec:
  parentRefs:
  - kind: Gateway
    name: mygateway1-global-staticip
  rules:
  - backendRefs:
    - name: myapp4-service
      port: 80
--- 

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaled-object
  namespace: default
spec:
  scaleTargetRef:
    name: myapp4-deployment
  minReplicaCount: 0
  maxReplicaCount: 5
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Karachi
      start: "0 9 * * *"
      end: "0 02 * * *"
      desiredReplicas: "3"


```

Deploy application:

```
kubectl apply -f deployment.yaml
```

---

## Step 4: Scale your Pub/Sub workload to zero

Deploy a Pub/Sub workload:

```
gcloud pubsub topics create keda-echo
gcloud pubsub subscriptions create keda-echo-read --topic=keda-echo
```

Bind Pub/Sub role:

```
gcloud projects add-iam-policy-binding projects/${PROJECT_ID}  \
    --role=roles/pubsub.subscriber \
  --member=principal://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/${PROJECT_ID}.svc.id.goog/subject/ns/keda-pubsub/sa/keda-pubsub-sa

kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/kubernetes-engine-samples/refs/heads/main/cost-optimization/gke-keda/cloud-pubsub/deployment/keda-pubsub-with-workload-identity.yaml
```

Configure scale-to-zero:

```
curl https://raw.githubusercontent.com/GoogleCloudPlatform/kubernetes-engine-samples/refs/heads/main/cost-optimization/gke-keda/cloud-pubsub/deployment/keda-pubsub-scaledobject.yaml | envsubst | kubectl apply -f -
```

This creates the following object:

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-pubsub
  namespace: keda-pubsub
spec:
  maxReplicaCount: 5
  scaleTargetRef:
    name: keda-pubsub
  triggers:
    - type: gcp-pubsub
      authenticationRef:
        name: keda-auth
      metadata:
        subscriptionName: "projects/${PROJECT_ID}/subscriptions/keda-echo-read"
```

```
kubectl get hpa keda-hpa-keda-pubsub -n keda-pubsub -o yaml
```

```
kubectl describe hpa keda-hpa-keda-pubsub -n keda-pubsub
```

Enqueue messages on the Pub/Sub topic:

```
for num in {1..20}
do
  gcloud pubsub topics publish keda-echo --project=${PROJECT_ID} --message="Test"
done
```

Verify that the Deployment is scaling up:

```
kubectl get deployments -n keda-pubsub
```



---

## Step 5: Create KEDA ScaledObject (Pub/Sub Trigger)

Example configuration:

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: pubsub-scaledobject
spec:
  scaleTargetRef:
    name: sample-app
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: gcp-pubsub
    metadata:
      subscriptionName: example-subscription
      subscriptionSize: "5"
```

Apply configuration:

```
kubectl apply -f scaledobject.yaml
```

Verify:

```
kubectl get scaledobject
```

---

## Step 6: Prometheus Metric Scaling Example

Example trigger:

```
triggers:
- type: prometheus
  metadata:
    serverAddress: http://prometheus-server.monitoring.svc.cluster.local
    metricName: http_requests_total
    threshold: "100"
    query: sum(rate(http_requests_total[1m]))
```

---

## Step 7: Validate Autoscaling

Check generated HPA:

```
kubectl get hpa
```

Watch scaling events live:

```
kubectl get pods -w
```

Describe scaling object:

```
kubectl describe scaledobject pubsub-scaledobject
```

---

## Observability Integration

Recommended production monitoring stack:

```
Managed Service for Prometheus
Cloud Monitoring
Cloud Logging
Grafana dashboards (optional)
```

Monitor operator logs:

```
kubectl logs deployment/keda-operator -n keda
```

---

## Production Best Practices Implemented

This project follows real enterprise deployment patterns:

✔ Private cluster networking
✔ Workload Identity authentication
✔ External metrics autoscaling
✔ Namespace isolation support
✔ Helm-based lifecycle management
✔ Secure IAM role binding
✔ Compatible with Cluster Autoscaler
✔ Terraform-ready infrastructure automation

---

## Terraform Automation Support (Optional Extension)

Typical IaC structure:

```
terraform/
 ├── networking
 ├── gke-private-cluster
 ├── workload-identity
 ├── pubsub
 └── monitoring
```

Enables full environment provisioning using Infrastructure as Code.

---

## Troubleshooting Guide

Check metrics adapter availability:

```
kubectl get apiservices | grep external.metrics
```

Inspect ScaledObject status:

```
kubectl describe scaledobject pubsub-scaledobject
```

Check operator logs:

```
kubectl logs deployment/keda-operator -n keda
```

---

## Cleanup Resources

Remove autoscaling config:

```
kubectl delete -f scaledobject.yaml
```

Uninstall KEDA:

```
helm uninstall keda -n keda
```

---

## Real‑World Use Cases Supported by This Architecture

Event-driven scaling is ideal for:

* Queue consumers
* Streaming workloads
* Background job processors
* ML inference workers
* Notification services
* Payment processing pipelines

---

## Author

Aslam Chandio
Cloud & DevOps Engineer
Specializing in Kubernetes • GKE • Terraform • Event‑Driven Autoscaling

---

⭐ If this project helped you, consider starring the repository.
