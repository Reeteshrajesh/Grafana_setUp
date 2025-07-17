# 📄 Promtail Configuration for Full EKS Cluster Log Ingestion

## 🔍 Overview

This configuration sets up **Promtail** to collect **all logs across your Kubernetes cluster**, including:

- **Application logs**
- **System components**
- **ALB (AWS Load Balancer) logs**
- Logs from **all pods, nodes, and namespaces**

Logs are sent to **Loki**, which acts as the centralized log aggregation backend, enabling powerful filtering and querying via **Grafana**.

---

## 🎯 Why This Matters

Modern Kubernetes clusters generate logs from dozens of sources:
- Microservices
- Gateways and Ingresses
- Controllers (e.g., cert-manager, autoscaler)
- CoreDNS, kubelet, etc.
- External components like AWS ALB

This setup ensures:
- You get **full observability** across your cluster.
- You can filter logs using meaningful labels (`app`, `namespace`, `statusCode`, `route`, etc.).
- You can detect errors, bottlenecks, or anomalies **faster**.
- Logs are **structured and queryable**, not just plain text blobs.

---

## 🧩 Promtail Configuration

```yaml
promtail:
  enabled: true
  config:
    serverPort: 3101
    logLevel: info

    clients:
      - url: http://loki:3100/loki/api/v1/push
        tenant_id: my-tenant  # Optional multi-tenancy

    positions:
      filename: /tmp/positions.yaml

    scrape_configs:
      - job_name: kubernetes-pods
        pipeline_stages:
          # Stage 1: JSON Parsing
          - json:
              expressions:
                statusCode: "statusCode"
                route: "route"
                error: "error"

          # Stage 2: If not JSON, treat as raw text
          - regex:
              expression: '^(?P<message>.*)$'

          # Stage 3: Add custom labels from JSON
          - labels:
              statusCode: "{{.statusCode}}"
              route: "{{.route}}"
              error: "{{.error}}"

          # Stage 4: Output formatting
          - output:
              log_message: "{{.message}}"

        kubernetes_sd_configs:
          - role: pod

        relabel_configs:
          # Only collect logs from containers (not init or pause containers)
          - source_labels: [__meta_kubernetes_pod_container_name]
            action: keep
            regex: .*

          # Log file path
          - source_labels: [__meta_kubernetes_pod_uid, __meta_kubernetes_pod_container_name]
            action: replace
            target_label: __path__
            replacement: /var/log/pods/*/*/*.log

        # Add Kubernetes metadata
        additional_scrape_configs:
          - job_name: kubernetes-alb-logs
            static_configs:
              - targets: ['localhost']
                labels:
                  job: alb
                  __path__: /var/log/alb/*.log
````

---

## 🏷️ Labels Automatically Attached

This config will extract or attach these labels to logs:

* `job`
* `instance`
* `filename`
* `namespace`
* `pod`
* `container`
* `node_name`
* `app`
* `component`
* `statusCode`, `route`, `error` (from JSON logs)
* `stream` (stdout/stderr)

These labels help build powerful Grafana queries like:

```logql
{app="gateway", statusCode="500"}
{namespace="payments", error!="nil"}
{job="alb"} |= "502"
```

---

## 🚀 Benefits

* 🔧 **Pipeline stages** handle structured logs and fallback to text
* 📊 **Query anything** from login failures to HTTP status codes
* 🧠 **Intelligent metadata** enables fast root cause analysis
* 🔐 Ready for **multi-tenant Loki** (with optional `tenant_id`)

---

## ✅ Tips

* Make sure `/var/log/pods` and `/var/log/alb` are available inside the Promtail pod via hostPath volumes.
* Ensure Promtail has proper RBAC permissions to read pod metadata.
* Use `kubectl logs -n loki -l app=promtail` to verify it’s running.
* Visualize logs in Grafana using Loki as a data source.

---

## 💡 Example Use Case

Let’s say your API Gateway logs JSON like this:

```json
{"statusCode":500, "route":"/api/users", "error":"DBError: timeout"}
```

With this config, Promtail will extract:

* `statusCode="500"`
* `route="/api/users"`
* `error="DBError: timeout"`

You can now search in Grafana:

```logql
{route="/api/users", statusCode="500"}
```

---

## 📚 Related Tools

* [Grafana Loki](https://grafana.com/oss/loki/)
* [Promtail Docs](https://grafana.com/docs/loki/latest/clients/promtail/)
* [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)

---

## 🛠 Future Improvements

* Add log retention in Loki
* Enable compression + deduplication
* Ship logs to long-term storage (e.g., S3 via Loki BoltDB Ship)

---

Made for a high-traffic microservices Kubernetes cluster with ❤️
