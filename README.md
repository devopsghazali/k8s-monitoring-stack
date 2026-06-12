# Kubernetes Monitoring Stack

Professional Kubernetes monitoring stack using Prometheus, Grafana, kube-state-metrics, node-exporter, and Metrics Server.

## Repository Structure

```text
monitoring-stack/
├── helm/
│   ├── kube-prometheus-stack-values.yaml
│   └── metrics-server-values.yaml
├── manifests/
│   ├── alerts/
│   │   └── basic-alerts.yaml
│   ├── dashboards/
│   │   └── workload-overview-dashboard.yaml
│   ├── namespaces/
│   │   └── monitoring.yaml
│   └── sample-app/
│       └── load-test-app.yaml
└── README.md
```

## Prerequisites

- Kubernetes cluster access with `kubectl`
- Helm 3 installed
- Argo CD is optional; this stack can be deployed directly with Helm

## 1. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
```

## 2. Create Namespace

```bash
kubectl apply -f manifests/namespaces/monitoring.yaml
```

## 3. Install Metrics Server

The provided values include `--kubelet-insecure-tls`, which is useful for local clusters such as Minikube, Kind, or self-signed kubelet certificates.

```bash
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  -f helm/metrics-server-values.yaml
```

Verify:

```bash
kubectl top nodes
kubectl top pods -A
```

## 4. Install kube-prometheus-stack

Use release name `monitoring`. The alert rules in this repository use the label `release: monitoring`, which lets the Prometheus instance discover them.

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f helm/kube-prometheus-stack-values.yaml
```

Wait for pods:

```bash
kubectl get pods -n monitoring
```

## 5. Apply Dashboards And Alerts

```bash
kubectl apply -f manifests/dashboards/workload-overview-dashboard.yaml
kubectl apply -f manifests/alerts/basic-alerts.yaml
```

## 6. Access Grafana

Default credentials from the values file:

- Username: `admin`
- Password: `admin123`

Port-forward Grafana:

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

The dashboard appears under the `Kubernetes` folder as `Workload Overview`.

## 7. Test Monitoring With A Sample App

Deploy the sample app:

```bash
kubectl apply -f manifests/sample-app/load-test-app.yaml
```

Generate traffic:

```bash
kubectl run load-generator \
  --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://sample-nginx.demo-load.svc.cluster.local >/dev/null; done"
```

In another terminal, watch CPU and memory:

```bash
kubectl top pods -n demo-load
```

Force a restart to test the restart alert:

```bash
kubectl delete pod -n demo-load -l app=sample-nginx
```

To create repeated restarts, temporarily patch the deployment with a bad command:

```bash
kubectl patch deployment sample-nginx -n demo-load \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command","value":["/bin/sh","-c","exit 1"]}]'
```

Restore:

```bash
kubectl rollout undo deployment/sample-nginx -n demo-load
```

## 8. Cleanup

```bash
kubectl delete -f manifests/sample-app/load-test-app.yaml
kubectl delete -f manifests/alerts/basic-alerts.yaml
kubectl delete -f manifests/dashboards/workload-overview-dashboard.yaml
helm uninstall monitoring -n monitoring
helm uninstall metrics-server -n kube-system
kubectl delete namespace monitoring
```

## Notes

- For production, replace the Grafana admin password with a Kubernetes Secret or external secret manager.
- Keep `--kubelet-insecure-tls` only when your cluster requires it. For hardened clusters, configure valid kubelet serving certificates instead.
- The CPU alert uses container CPU limits. Workloads without CPU limits may not be included in that alert ratio.
