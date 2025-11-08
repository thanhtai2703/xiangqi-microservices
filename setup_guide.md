# Xiangqi Project Monitoring Setup Guide - Phase 1

This guide documents the steps to set up k3s, Grafana, and Prometheus for monitoring the Xiangqi project.

## Prerequisites

- Linux system with curl and bash
- sudo privileges for system installations
- kubectl installed locally

## Step 1: Install k3s

k3s is a lightweight Kubernetes distribution that we'll use for our cluster.

```bash
curl -sfL https://get.k3s.io | sh -
```

This command downloads and installs k3s with default settings. The installation includes:

- Kubernetes cluster components
- Container runtime (containerd)
- Default network policies

## Step 2: Verify k3s Cluster

Verify that the cluster is running and accessible:

```bash
kubectl get nodes
```

Expected output should show one node in "Ready" status:

```
NAME     STATUS   ROLES                  AGE     VERSION
debian   Ready    control-plane,master   2m13s   v1.33.5+k3s1
```

## Step 3: Install Helm

Helm is the package manager for Kubernetes that we'll use to install Prometheus and Grafana:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify Helm installation:

```bash
helm version
```

## Step 4: Add Prometheus Helm Repository

Add the Prometheus community Helm repository to access the kube-prometheus-stack chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update Helm repositories:

```bash
helm repo update
```

## Step 5: Create Monitoring Namespace

Create a dedicated namespace for all monitoring components:

```bash
kubectl create namespace monitoring
```

## Step 6: Install kube-prometheus-stack

Install the complete monitoring stack using Helm:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=ClusterIP \
  --set grafana.service.type=ClusterIP
```

This installs:

- Prometheus server for metric collection and storage
- Grafana for visualization
- Alertmanager for alert handling
- Node Exporter to collect node metrics
- Kube State Metrics to export Kubernetes object state
- Prometheus Operator for management

## Step 7: Verify Installation

Check that all pods in the monitoring namespace are running:

```bash
kubectl get pods -n monitoring
```

All pods should eventually show "Running" status:

```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          5m
prometheus-grafana-787844d559-f5t2r                      3/3     Running   0          5m
prometheus-kube-prometheus-operator-644c4f8c94-sm8nb     1/1     Running   0          5m
prometheus-kube-state-metrics-f7d8f5f9b-njkvt            1/1     Running   0          5m
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          5m
prometheus-prometheus-node-exporter-f6gqd                1/1     Running   0          5m
```

## Step 8: Get Grafana Password

Retrieve the default Grafana admin password:

```bash
kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

## Step 9: Configure Ingress for External Access

Create an ingress resource for Grafana:

1. Create `grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
  - host: grafana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-grafana
            port:
              number: 80
```

2. Create `prometheus-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
  - host: prometheus.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-kube-prometheus-prometheus
            port:
              number: 9090
```

Apply the ingress resources:

```bash
kubectl apply -f grafana-ingress.yaml
kubectl apply -f prometheus-ingress.yaml
```

## Step 10: Update Hosts File

Add entries to your local hosts file to map the domain names to your k3s cluster IP:

```bash
echo -e "192.168.0.7 grafana.local\n192.168.0.7 prometheus.local" | sudo tee -a /etc/hosts
```

Note: The IP address 192.168.0.7 should match your actual k3s cluster IP. Check with `kubectl get nodes -o wide`.

## Step 11: Access the Services

After updating the hosts file, you can access:

- Grafana at http://grafana.local (use admin as the username and the password retrieved in Step 8)
- Prometheus at http://prometheus.local

## Verification

At this point, you should have:

1. A running k3s cluster
2. A complete monitoring stack with Prometheus and Grafana
3. Ingress configured for external access to monitoring tools
4. The ability to access Grafana and Prometheus through browser using local domain names

This completes Phase 1 of the monitoring setup for the Xiangqi project.