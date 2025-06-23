# **Complete Guide: Setting Up IAM Roles for Service Accounts (IRSA) in Amazon EKS for Kubernetes Deployments**

## **Overview**

This guide provides detailed steps to configure **IAM Roles for Service Accounts (IRSA)** in **Amazon EKS**. With IRSA, your **Kubernetes pods** can securely access **AWS resources (like S3)** without the need to hardcode AWS credentials. This guide includes setting up the IAM role with appropriate permissions, linking it to Kubernetes service accounts, and deploying applications like **Loki** using **Helm**.

---

## **Table of Contents**

1. [Pre-requisites](#pre-requisites)
2. [Step 1: Enable OIDC Provider for EKS](#step-1-enable-oidc-provider-for-eks)
3. [Step 2: Create IAM Role with S3 Permissions](#step-2-create-iam-role-with-s3-permissions)
4. [Step 3: Create and Annotate Kubernetes Service Account](#step-3-create-and-annotate-kubernetes-service-account)
5. [Step 4: Update Helm Chart (`values.yaml`) for Loki](#step-4-update-helm-chart-valuesyaml-for-loki)
6. [Step 5: Deploy the Application Using Helm](#step-5-deploy-the-application-using-helm)
7. [Step 6: Verify the Setup](#step-6-verify-the-setup)
8. [Troubleshooting](#troubleshooting)
9. [Conclusion](#conclusion)

---

## **1. Pre-requisites**

Before you begin, ensure that the following are in place:

1. **Amazon EKS Cluster** with **OIDC provider** enabled.
2. **AWS CLI** installed and configured on your local machine.
3. **Helm** (version 3 or above) installed and configured.
4. **kubectl** installed and configured for interacting with the Kubernetes cluster.
5. **IAM Policies** that allow access to AWS resources such as **S3**, **DynamoDB**, etc.
6. The **IAM role** should be associated with the **Service Account** that will be used in your Kubernetes pods.

---

## **2. Step 1: Enable OIDC Provider for EKS**

The **OIDC provider** in EKS allows Kubernetes pods to authenticate with IAM roles using web identity tokens. This is required to set up **IAM Roles for Service Accounts (IRSA)**.

1. **Go to the Amazon EKS Console**.
2. Select your **EKS cluster**.
3. Under **Configuration**, click **OIDC provider**.
4. If the OIDC provider is not yet enabled, click **Associate OIDC provider** to link your EKS cluster with IAM.

Alternatively, you can use the **AWS CLI** to associate the OIDC provider with your EKS cluster:

```bash
eksctl utils associate-iam-oidc-provider --region <region> --cluster <cluster-name> --approve
```

---

## **3. Step 2: Create IAM Role with S3 Permissions**

### a. **Create the IAM Role**

1. **Go to the IAM Console** → **Roles** → **Create role**.
2. Select **Web Identity** as the trusted entity type.
3. Choose your **OIDC provider** (created in Step 1) and your **EKS cluster**.
4. In the **Audience** field, use the default `sts.amazonaws.com`.
5. **Role Name**: Name the role (e.g., `eks-loki-role`).
6. Click **Next** to proceed.

### b. **Attach S3 Permissions to IAM Role**

You can either use existing policies like `AmazonS3ReadOnlyAccess` or create a custom policy for fine-grained permissions. Here’s an example of a **custom policy** for allowing access to a specific S3 bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::<your-bucket-name>",
        "arn:aws:s3:::<your-bucket-name>/*"
      ]
    }
  ]
}
```

* Replace `<your-bucket-name>` with the actual name of your S3 bucket.

After creating the policy, attach it to the IAM role.

### c. **Set Trust Relationship for the IAM Role**

This trust policy allows the **Kubernetes service account** to assume this IAM role using the **OIDC provider**. Here’s an example of a trust relationship:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<region>.amazonaws.com/id/<eks-cluster-id>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

---

## **4. Step 3: Create and Annotate Kubernetes Service Account**

### a. **Create the Service Account**

If you haven’t already, create a **Kubernetes service account** in your desired namespace:

```bash
kubectl create serviceaccount <service-account-name> -n <namespace>
```

### b. **Annotate the Service Account**

Link the **service account** to the **IAM role** by adding the following annotation:

```bash
kubectl annotate serviceaccount <service-account-name> eks.amazonaws.com/role-arn=arn:aws:iam::<account-id>:role/<iam-role-name> -n <namespace>
```

* Replace `<service-account-name>`, `<account-id>`, and `<iam-role-name>` with the correct values.

---

## **Installation**

Follow these steps to install and configure the Loki Stack on your Kubernetes cluster using Helm.

### **Step 1: Add the Helm Chart Repository**

Add the Grafana Helm chart repository to your Helm configuration and update it:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## **5. Step 4: Update Helm Chart (`values.yaml`) for Loki**

Once the service account is set up, update your Helm chart's `values.yaml` file to reference the service account and use IRSA for AWS credentials management.

### Full Example `values.yaml`

```yaml
# Loki configuration values for Helm
loki:
  enabled: true
  isDefault: true
  url: http://{{(include "loki.serviceName" .)}}:{{ .Values.loki.service.port }}

  # Service Account configuration
  serviceAccount:
    create: false  # We are not creating the service account here; it already exists
    name: <service-account-name>  # Reference the service account name created earlier and annotated with IAM Role

  # Loki storage configuration (S3 setup with AWS IRSA)
  storage_config:
    aws:
      s3: https://s3.ap-south-1.amazonaws.com/l*******s-bucket  # Replace with your actual S3 bucket name
      s3forcepathstyle: true
      region: ap-south-1
      # Credentials management handled by IRSA, so no need for access_key_id and secret_access_key
      # The pod will automatically assume the IAM role attached to the service account

    boltdb_shipper:
      active_index_directory: /data/index
      cache_location: /data/cache
      shared_store: s3

  # Compactor settings for Loki
  compactor:
    working_directory: /data/compactor
    shared_store: s3

  # Loki persistence configuration (to store Loki data in persistent storage)
  persistence:
    enabled: true
    storageClassName: gp2  # Ensure this is set to the correct storage class for your environment
    accessModes:
      - ReadWriteOnce
    size: 10Gi  # Adjust as needed based on storage requirements

  # Probes configuration for pod readiness and liveness checks
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

  # Loki configuration options (example)
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
        s3: https://s3.ap-south-1.amazonaws.com/l*******s-bucket  # Ensure this matches your bucket
        s3forcepathstyle: true
        region: ap-south-1
        insecure: false
      boltdb_shipper:
        active_index_directory: /data/index
        cache_location: /data/cache
        shared_store: s3

    compactor:
      working_directory: /data/compactor
      shared_store: s3

# Promtail configuration for collecting logs and sending them to Loki
promtail:
  enabled: true
  config:
    logLevel: info
    serverPort: 3101
    clients:
      - url: http://{{ .Release.Name }}:3100/loki/api/v1/push
        tenant_id: my-tenant
  tolerations:
    - key: "redis-only"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
    - key: "resource_type"
      operator: "Equal"
      value: "jitsu"
      effect: "NoExecute"
    - key: "resource_type"
      operator: "Equal"
      value: "jitsu"
      effect: "NoSchedule"

# Grafana configuration for monitoring with Loki
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

# Prometheus configuration (optional if needed for monitoring)
prometheus:
  enabled: true
  isDefault: false
  url: http://{{ include "prometheus.fullname" .}}:{{ .Values.prometheus.server.service.servicePort }}{{ .Values.prometheus.server.prefixURL }}
  datasource:
    jsonData: "{}"
  persistence:
    enabled: true
    storageClassName: gp2
    accessModes:
      - ReadWriteOnce
    size: 50Gi  # Adjust storage size for production environments
    mountPath: /prometheus

# Optional filebeat configuration (if needed)
filebeat:
  enabled: false
  filebeatConfig:
    filebeat.yml: |
      # logging.level: debug
      filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
      output.logstash:
        hosts: ["logstash-loki:5044"]

# Optional logstash configuration (if needed)
logstash:
  enabled: false
  image: grafana/logstash-output-loki
  imageTag: 1.0.1
  filters:
    main: |-
      filter {
        if [kubernetes] {
          mutate {
            add_field => {
              "container_name" => "%{[kubernetes][container][name]}"
              "namespace" => "%{[kubernetes][namespace]}"
              "pod" => "%{[kubernetes][pod][name]}"
            }
            replace => { "host" => "%{[kubernetes][node][name]}"}
          }
        }
        mutate {
          remove_field => ["tags"]
        }
      }
  outputs:
    main: |-
      output {
        loki {
          url => "http://loki:3100/loki/api/v1/push"
        }
      }

# Proxy configuration (if needed)
proxy:
  http_proxy: ""
  https_proxy: ""
  no_proxy: ""

```

---

## **6. Step 5: Deploy the Application Using Helm**

Once your `values.yaml` file is updated, use Helm to deploy Loki or any other service that needs to interact with AWS services.

```bash
helm install loki-stack grafana/loki-stack -f values.yaml
```

---

## **7. Step 6: Verify the Setup**

### a. **Verify the Service Account in the Pod**

Ensure that your pods are using the correct service account by running the following command:

```bash
kubectl get pod <loki-pod-name> -o=jsonpath='{.spec.serviceAccountName}'
```

This should return the name of the service account that is assigned to the pod.

### b. **Test AWS S3 Access**

To confirm that the pod can access the S3 bucket (using the permissions granted via IRSA), run the following command:

```bash
kubectl exec -it <loki-pod-name> -- aws s3 ls s3://<your-bucket-name>
```

This will list the objects in the specified S3 bucket if the IRSA configuration is set up correctly.

---

## **8. Troubleshooting**

* **Service Account Not Found**: Make sure the service account is correctly created and annotated.
* **No Access to S3**: If the pod cannot access S3, ensure that the IAM role attached to the service account has the correct S3 permissions.
* **Trust Relationship Issues**: Verify that the IAM trust relationship is correctly configured for the Kubernetes service account.

---

## **9. Conclusion**

By following these steps, you have successfully configured **IAM Roles for Service Accounts (IRSA)** in Amazon EKS. This allows your Kubernetes workloads to access AWS services (such as S3) securely without the need for hardcoded AWS credentials.

The guide covered everything from enabling OIDC for EKS, creating IAM roles with appropriate permissions, associating the IAM role with a Kubernetes service account, and deploying an application (Loki) via Helm using **IRSA**.

If you need more help or additional configurations, refer to the [AWS IRSA documentation](https://docs.aws.amazon.com/eks/latest/userguide/pod-configuration.html).

_____-----
-----_____

### **Setup Ingress for Grafana with ALB**
### Create an Ingress Resource for Grafana**

Now, you can create an **Ingress** resource that exposes Grafana using the **ALB**. This Ingress will route traffic from the ALB to the Grafana service in your EKS cluster.

#### a. **Create a new `grafana-ingress.yaml` file**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: <namespace>  # Replace with the namespace where Grafana is deployed
  annotations:
    kubernetes.io/ingress.class: "alb"  # ALB Ingress controller
    alb.ingress.kubernetes.io/scheme: "internet-facing"  # Publicly accessible (use "internal" if private ALB is needed)
    alb.ingress.kubernetes.io/healthcheck-path: "/api/health"  # Health check path
    alb.ingress.kubernetes.io/target-type: "ip"  # Use IP as the target type
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'  # Listen on HTTP port 80 (change if needed)
    alb.ingress.kubernetes.io/group.name: "grafana-ingress-group"  # Group for the ALB Ingress controller to use
spec:
  rules:
    - host: Reetesh.example.com  # Replace with your ALB DNS name (e.g., grafana.example.com)
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana  # Replace with the Grafana service name
                port:
                  number: 3000  # Default Grafana port
```

### **Key Points:**

* **Annotations**: These annotations configure the ALB Ingress controller to route traffic to your Grafana service. The `alb.ingress.kubernetes.io/scheme: "internet-facing"` makes the ALB public. Change to `"internal"` for a private ALB.
* **Health Check**: The `/api/health` path is used to monitor the health of the Grafana service. You can adjust this if necessary based on how your application handles health checks.
* **ALB DNS**: Replace `Reetesh.example.com` with your actual ALB DNS name (e.g., `grafana.example.com`). You’ll need to configure this DNS with your domain.

---

### **3. Apply the Ingress Resource**

Once the **`grafana-ingress.yaml`** file is ready, apply it to your EKS cluster:

```bash
kubectl apply -f grafana-ingress.yaml
```

This will create the **Ingress resource** and expose your Grafana dashboard through the **ALB**.

---

### **4. Update DNS Records**

If you are using a custom domain for Grafana (e.g., `grafana.example.com`), you’ll need to create a **DNS record** pointing to your **ALB**.

* **Go to your DNS provider** (Route 53 if you're using AWS).
* Create a **CNAME record** pointing your subdomain (e.g., `grafana.example.com`) to the **ALB DNS name**.

---
