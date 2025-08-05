Centralized Monitoring Tools for CNA Environments

**Prometheus Deployment**
1. Deploy one Prometheus instance per customer environment, each responsible for scraping metrics locally (Kubernetes, apps, node, exporters).

Deploy Prometheus using Helm (recommended on OpenShift/Rancher):

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```
Configure values.yaml to:

Scrape K8s metrics

Integrate exporters (e.g., node_exporter, kube-state-metrics)

Add basic externalLabels for Thanos:

```
prometheus:
  prometheusSpec:
    externalLabels:
      cluster: customer-1
```

2. Integrate Thanos Sidecar (per Prometheus instance)

Deploy Thanos Sidecar as a sidecar container alongside Prometheus:

```
containers:
- name: prometheus
  image: prom/prometheus
- name: thanos-sidecar
  image: thanosio/thanos:latest
  args:
    - sidecar
    - --tsdb.path=/prometheus
    - --prometheus.url=http://localhost:9090
    - --grpc-address=0.0.0.0:10901
    - --http-address=0.0.0.0:10902
```
(Optional) Configure object storage (e.g., MinIO, S3) for long-term storage:

```
- --objstore.config-file=/etc/thanos/storage.yaml
```

3. Set up Thanos Query + Store + Compactor (Central Layer)
This is the central query + storage federation layer where you aggregate all customer metrics centrally, enabling multi-tenancy and long-term data retention.

Deploy Thanos Query:

```
thanos query \
  --http-address=0.0.0.0:9090 \
  --grpc-address=0.0.0.0:10901 \
  --store=sidecar1:10901 \
  --store=sidecar2:10901 \
```

4. Deploy Thanos Store (for object storage retrieval) and Deploy Thanos Compactor (optional for storage optimization)

Note: All Thanos components can run in a centralized Kubernetes cluster 

Set up Grafana (Multi-tenant Dashboard Layer)

5. Deploy Grafana
```
helm install grafana grafana/grafana
```

Add Thanos Query endpoint as a data source:

Type: Prometheus
URL: http://thanos-query.monitoring.svc:9090

Enable folder-level permissions or organizations for multi-tenant views:

Use Grafana’s organizations for customer separation

Role-based access control for dashboards and data sources

Import dashboards (Kubernetes, node, application, etc.)

6. Set up Alertmanager (for Alert Routing)

Integrated via kube-prometheus-stack by default

Create alert rules per tenant:

```
- alert: HighCPUUsage
  expr: node_cpu_seconds_total > 90
  labels:
    tenant: customer-1
 ``` 
Route alerts by tenant (e.g., via email, Slack, Opsgenie)

7. Enable Security, IAM & Compliance

Enable OAuth2 / LDAP in Grafana (per org)

Enable TLS for Thanos gRPC + HTTP endpoints

Use audit logging for Grafana & Prometheus access

Configure network policies between clusters (e.g., via Istio, Cilium)

If needed, integrate Vault for secrets management

8. Monitor Legacy Systems (e.g., Broadcom Supervisor)
Since Broadcom Supervisor isn’t cloud-native:

Use SNMP exporters, custom shell scripts, or Zabbix to export legacy metrics

Convert to Prometheus metrics via:

SNMP Exporter - https://github.com/prometheus/snmp_exporter

Script Exporter - https://github.com/ricoberger/script_exporter

Feed them into Prometheus like any other exporter


Architecture Diagram (Simplified)
```
+---------------------+       +---------------------+        +---------------------+
| Customer Cluster A  |       | Customer Cluster B  |  ...   | Customer Cluster N  |
| - Prometheus        |       | - Prometheus        |        | - Prometheus        |
| - Thanos Sidecar    |       | - Thanos Sidecar    |        | - Thanos Sidecar    |
+---------------------+       +---------------------+        +---------------------+
          \                            |                             /
           \___________________________|___________________________/
                                        |
                                 +-------------+
                                 | Thanos Query| ← connects to Sidecars
                                 +-------------+
                                        |
                     +-----------------+----------------+
                     |                                  |
              +-------------+                    +-------------+
              | Grafana     |                    | Alertmanager |
              +-------------+                    +-------------+

