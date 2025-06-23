# **Loki Stack on Kubernetes using Helm**

## **Overview**

This project provides a comprehensive solution for log aggregation, visualization, and monitoring using **Loki**, **Promtail**, **Grafana**, and **Prometheus** on a Kubernetes cluster. The stack is installed using **Helm**, and each component is configured with a custom `values.yaml` file for easy customization.

### **Components**:

* **Loki**: A horizontally scalable, highly available log aggregation system.
* **Promtail**: An agent for collecting and pushing logs to Loki.
* **Grafana**: A visualization tool for displaying logs stored in Loki.
* **Prometheus**: A toolkit for monitoring and alerting.

## **Prerquisites**

Before you begin, make sure you have the following tools installed:

* **Kubernetes Cluster**: Ensure you have access to a running Kubernetes cluster (e.g., Minikube, GKE, EKS, AKS).
* **Helm**: Install Helm, a Kubernetes package manager. You can follow the [installation guide here](https://helm.sh/docs/intro/install/).
* **kubectl**: Command-line tool to interact with your Kubernetes cluster. Install it from [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
* **AWS Credentials (Optional)**: For S3-based storage in Loki, ensure you have valid AWS credentials.

## **Project Structure**

This project deploys the following components using Helm:

1. **Loki** for log aggregation and storage.
2. **Promtail** for collecting and forwarding logs to Loki.
3. **Grafana** for visualizing logs and metrics.
4. **Prometheus** for monitoring and alerting based on metrics.

## **Installation**

Follow these steps to install and configure the Loki Stack on your Kubernetes cluster using Helm.

### **Step 1: Add the Helm Chart Repository**

Add the Grafana Helm chart repository to your Helm configuration and update it:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### **Step 2: Create a Custom `values.yaml` Configuration**

Create a `values.yaml` file to configure the Loki stack components. Below is a sample `values.yaml` file that configures Loki, Promtail, Grafana, and Prometheus:

```yaml
# values.yaml

loki:
  enabled: true
  isDefault: true
  url: "http://{{ .Release.Name }}:{{ .Values.loki.service.port }}"

  readinessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45

  livenessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45

  persistence:
    enabled: true
    storageClassName: gp2
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    mountPath: /data

  config:
    schema_config:
      configs:
        - from: 2021-01-01
          store: boltdb-shipper
          object_store: s3
          schema: v12
          index:
            prefix: index_
            period: 24h
    storage_config:
      aws:
        s3: https://s3.ap-south-1.amazonaws.com/your-bucket
        s3forcepathstyle: true
        region: ap-south-1
        access_key_id: YOUR_ACCESS_KEY
        secret_access_key: YOUR_SECRET_KEY
        insecure: false
      boltdb_shipper:
        active_index_directory: /data/index
        cache_location: /data/cache
        shared_store: s3

  compactor:
    working_directory: /data/compactor
    shared_store: s3

promtail:
  enabled: true
  config:
    logLevel: info
    serverPort: 3101
    clients:
      - url: "http://{{ .Release.Name }}:3100/loki/api/v1/push"
        tenant_id: "my-tenant"

grafana:
  enabled: true
  sidecar:
    datasources:
      label: ""
      labelValue: ""
      enabled: true
      maxLines: 1000
      enableLogVolume: true
  image:
    tag: latest

prometheus:
  enabled: true
  persistence:
    enabled: true
    storageClassName: gp2
    accessModes:
      - ReadWriteOnce
    size: 50Gi
    mountPath: /prometheus
```

### **Step 3: Install the Loki Stack Using Helm**

Install the Loki stack by running the following command:

```bash
helm install loki-stack grafana/loki-stack -f values.yaml
```

This command will:

* Install the `loki-stack` chart from the **grafana** repository.
* Use the custom `values.yaml` configuration for all the components (Loki, Promtail, Grafana, and Prometheus).

### **Step 4: Verify the Installation**

Check the status of the deployed components using the following commands:

#### Check the pods:

```bash
kubectl get pods
```

You should see the following pods running:

* Loki
* Promtail
* Grafana
* Prometheus

#### Check the services:

```bash
kubectl get svc
```

To access the services (Grafana, Loki, Prometheus), use **port-forwarding** for local development or configure LoadBalancer/NodePort in production.

#### Access Grafana (via port-forwarding):

```bash
kubectl port-forward svc/loki-stack-grafana 3000:80
```

Once port-forwarded, navigate to `http://localhost:3000` to access the Grafana dashboard. The default login credentials are:

* **Username**: `admin`
* **Password**: `admin`

#### Access Loki Logs:

In Grafana, go to **Explore** and choose **Loki** as the data source to query logs.

---

## **Updating the Configuration**

To make changes to the configuration, such as modifying resource limits, adding new storage settings, or updating components:

1. Edit the `values.yaml` file with the desired changes.
2. Apply the changes by running the following Helm upgrade command:

```bash
helm upgrade loki-stack grafana/loki-stack -f values.yaml
```

This command will update the deployment with the new configuration.

---

## **Uninstalling the Loki Stack**

If you want to remove the Loki stack, run:

```bash
helm uninstall loki-stack
```

This will delete all the resources created by the Helm chart, including Loki, Promtail, Grafana, and Prometheus.

---

## **Conclusion**

You have now successfully installed and configured the **Loki Stack** on your Kubernetes cluster using Helm. The stack includes:

* **Loki** for log aggregation and storage.
* **Promtail** for log collection.
* **Grafana** for visualizing logs.
* **Prometheus** for monitoring and metrics collection.

You can now query logs via Grafana, visualize and explore metrics, and monitor your system. This setup is highly scalable and can be expanded based on your infrastructure needs.

---

### **Additional Resources**

* [Loki Documentation](https://grafana.com/docs/loki/latest/)
* [Helm Documentation](https://helm.sh/docs/)
* [Grafana Documentation](https://grafana.com/docs/grafana/latest/)

---
---

## **Components and Their Importance**

### 1. **Loki** - Log Aggregation

#### **Use**:

* **Loki** is the heart of the logging stack. It is responsible for aggregating logs from various services and applications running within your Kubernetes cluster.
* It stores logs in a highly optimized, compressed, and cost-efficient manner.

#### **Importance**:

* **Efficient log storage**: Unlike traditional logging systems that use indexing for full-text search, Loki uses a simpler index structure that groups logs by time and labels. This reduces the storage overhead and makes it more efficient for large-scale log aggregation.
* **Scalability**: Loki is designed to scale horizontally and works well in cloud-native environments, handling logs from multiple microservices without significant overhead.
* **Integration with Grafana**: Loki is designed to integrate seamlessly with **Grafana** for real-time log exploration and monitoring.

### 2. **Promtail** - Log Collection

#### **Use**:

* **Promtail** is the agent responsible for collecting logs from various sources and forwarding them to Loki for storage and analysis.
* It scrapes log files from Kubernetes pods, Docker containers, and other log sources.

#### **Importance**:

* **Log forwarding**: Promtail ensures that logs from applications and services running inside Kubernetes pods are forwarded to Loki in real-time.
* **Kubernetes integration**: Promtail integrates well with Kubernetes, extracting relevant metadata such as pod names, labels, namespaces, and container names to enrich logs with useful context.
* **Labeling and filtering**: Promtail can label logs with relevant metadata and apply filters, ensuring that logs are categorized and stored correctly in Loki.

### 3. **Grafana** - Data Visualization

#### **Use**:

* **Grafana** is a powerful visualization and dashboarding tool used to create interactive and customizable dashboards for visualizing log data stored in Loki.
* It provides a user-friendly interface for exploring logs and creating visualizations like time series charts, tables, and histograms.

#### **Importance**:

* **Log exploration**: Grafana allows users to search and explore logs stored in Loki, making it easier to troubleshoot issues by providing rich, detailed views of log data.
* **Alerting and monitoring**: In addition to visualizing logs, Grafana can integrate with Prometheus to visualize metrics, set up alerts, and monitor the health of your Kubernetes applications and infrastructure.
* **Real-time insights**: Grafana's dashboards can be updated in real-time, offering quick insights into system health, logs, and performance metrics.

### 4. **Prometheus** - Monitoring and Metrics

#### **Use**:

* **Prometheus** is a monitoring and alerting toolkit designed for collecting and storing time-series metrics.
* It collects data on the performance of your Kubernetes cluster, application services, and infrastructure, such as CPU usage, memory usage, request rates, and error rates.

#### **Importance**:

* **Metrics collection**: Prometheus is critical for gathering metrics that can be used to monitor the health and performance of your applications and Kubernetes cluster. It supports multi-dimensional data collection, providing detailed insights into system performance.
* **Alerting**: Prometheus can trigger alerts based on predefined thresholds (e.g., when CPU usage exceeds a certain level or when the request error rate spikes), ensuring that your team is notified of any issues promptly.
* **Integration with Loki and Grafana**: Prometheus can be integrated with Loki for a holistic view of both logs and metrics, and with Grafana to create powerful dashboards for both log data and performance metrics.


------
________
--------

# **Kubernetes Resource Types and Pod Names for the Loki Stack Deployment**

### **1. Loki**

* **Kubernetes Resource**: **StatefulSet**

  * **Reason**: **Loki** is stateful as it stores logs and requires persistent storage. Using a **StatefulSet** ensures stable network identities and persistent storage for the logs.
* **Pod Name Example**:

  * `loki-0`
* **Explanation**: The **StatefulSet** will create a pod named `loki-0` (or `loki-<pod-id>` depending on the scale). The **StatefulSet** ensures that the **Loki** pod has persistent storage and consistent naming.

---

### **2. Promtail**

* **Kubernetes Resource**: **DaemonSet**

  * **Reason**: **Promtail** is a log collector that runs on each node in the Kubernetes cluster to collect logs from all containers. The **DaemonSet** ensures that there is exactly one **Promtail** pod running on each node.

* **Pod Names Example**:

  * `loki-promtail-xyz`

* **Explanation**: **Promtail** runs as a **DaemonSet** to ensure that it collects logs from every node in the cluster. The pods are created dynamically, and their names are based on the DaemonSet template with unique identifiers.

---

### **3. Grafana**

* **Kubernetes Resource**: **Deployment**

  * **Reason**: **Grafana** is a stateless service, meaning it can be scaled up or down easily, and it doesn’t require persistent storage. A **Deployment** is used to ensure high availability and easy scaling.

* **Pod Name Example**:

  * `loki-grafana-b98d4759f-fhtng`

* **Explanation**: **Grafana** is deployed using a **Deployment** to ensure that there is always a replica of **Grafana** running. The pod name will be automatically generated by Kubernetes using the Deployment's label and a hash of the pod's spec.

---

### **4. Prometheus**

* **Kubernetes Resource**: **StatefulSet** (for Prometheus server) and **DaemonSet** (for node-exporter)

  * **Prometheus Server**: **StatefulSet**

    * **Reason**: Prometheus requires persistent storage to retain monitoring data, so it is deployed as a **StatefulSet**.
    * **Pod Name Example**: `loki-prometheus-server-6d76d4cc7-kmg6z`

  * **Node Exporter**: **DaemonSet**

    * **Reason**: The **Node Exporter** collects metrics about each node and runs on every node in the Kubernetes cluster. It’s best deployed as a **DaemonSet**.
    * **Pod Names Example**:

      * `loki-prometheus-node-exporter-xyz`

* **Explanation**: Prometheus server is stateful, and thus, a **StatefulSet** is used. The **Node Exporter** is stateless, running on each node as a **DaemonSet** to gather metrics from all nodes.

---

### **5. Alertmanager**

* **Kubernetes Resource**: **StatefulSet**

  * **Reason**: **Alertmanager** is used for managing and routing alerts, and it requires persistent storage for alert history and configurations.

* **Pod Name Example**:

  * `loki-alertmanager-0`

* **Explanation**: **Alertmanager** is deployed as a **StatefulSet** to ensure it has persistent storage and a stable network identity for processing and managing alerts.

---

### **6. Kube-State-Metrics**

* **Kubernetes Resource**: **Deployment**

  * **Reason**: **Kube-State-Metrics** is a monitoring service for exposing the state of Kubernetes objects as metrics. It is stateless and can be managed with a **Deployment**.

* **Pod Name Example**:

  * `loki-kube-state-metrics-58c656c44c-2swjd`

* **Explanation**: **Kube-State-Metrics** is deployed as a **Deployment** since it does not require persistent storage.

---

### **7. Prometheus Pushgateway**

* **Kubernetes Resource**: **Deployment**

  * **Reason**: **Prometheus Pushgateway** is a stateless component that allows you to push metrics. It is deployed with a **Deployment** to ensure availability and scalability.

* **Pod Name Example**:

  * `loki-prometheus-pushgateway-7c8c99858c-lqqpj`

* **Explanation**: **Prometheus Pushgateway** is deployed as a **Deployment**, as it is stateless and doesn't require persistent storage.

---

## **Summary of Resource Types and Pod Names**

| **Component**                | **Kubernetes Resource** | **Pod Name Example**                                                              |
| ---------------------------- | ----------------------- | --------------------------------------------------------------------------------- |
| **Loki**                     | StatefulSet             | `loki-0`                                                                          |
| **Promtail**                 | DaemonSet               | `loki-promtail-8zpj6`, `loki-promtail-q9j72`, ...                                 |
| **Grafana**                  | Deployment              | `loki-grafana-b98d4759f-fhtng`                                                    |
| **Prometheus Server**        | StatefulSet             | `loki-prometheus-server-6d76d4cc7-kmg6z`                                          |
| **Prometheus Node Exporter** | DaemonSet               | `loki-prometheus-node-exporter-tkhgs`, `loki-prometheus-node-exporter-45qb6`, ... |
| **Alertmanager**             | StatefulSet             | `loki-alertmanager-0`                                                             |
| **Kube-State-Metrics**       | Deployment              | `loki-kube-state-metrics-58c656c44c-2swjd`                                        |
| **Prometheus Pushgateway**   | Deployment              | `loki-prometheus-pushgateway-7c8c99858c-lqqpj`                                    |

---
----

# **Ingress Configuration for Grafana with AWS ACM Certificate ARN and Route 53**

## **Overview**

This guide helps you set up an **Ingress** for **Grafana** using an **AWS ACM Certificate ARN** for SSL termination, with DNS routing through **Route 53** to the domain `Reetesh.example.com`.

## **Prerequisites**

1. **AWS ACM Certificate** for `Reetesh.example.com`.
2. **AWS ALB Ingress Controller** installed in your Kubernetes cluster.
3. **Route 53** hosted zone for `example.com`.
4. **Grafana Service** exposed on port `3000`.

---

## **Ingress Configuration**

Create an `Ingress` resource (`grafana-ingress.yaml`) to expose **Grafana** securely using the ACM certificate for HTTPS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: default
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account-id:certificate/certificate-id
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: Reetesh.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: loki-grafana
            port:
              number: 3000
```

### **Key Points**:

* **Annotations**:

  * `alb.ingress.kubernetes.io/certificate-arn`: Uses AWS ACM certificate ARN for SSL termination.
  * `nginx.ingress.kubernetes.io/ssl-redirect`: Redirects HTTP to HTTPS.
  * `nginx.ingress.kubernetes.io/backend-protocol`: Ensures backend uses HTTPS.
* **Spec**: Routes traffic from `Reetesh.example.com` to the **Grafana service** (`loki-grafana`).

---

## **Step 1: Apply the Ingress**

```bash
kubectl apply -f grafana-ingress.yaml
```

---

## **Step 2: Route Traffic via Route 53**

1. In **Route 53**, create an **A record** for `Reetesh.example.com` pointing to the **ALB DNS name** from AWS.

---

## **Step 3: Access Grafana**

Once DNS propagates, access **Grafana** via:

```
https://Reetesh.example.com
```

---

