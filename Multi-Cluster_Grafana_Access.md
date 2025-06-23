# Multi-Cluster Grafana Access Setup with AWS ALB Ingress Controller and Istio

This guide provides step-by-step instructions on setting up **multi-cluster Grafana access** using a **single URL** (`activity.v2.liquide.life`) for **dev**, **prod**, and **analytics** environments. The setup uses **AWS ALB Ingress Controller** for routing traffic to multiple Kubernetes clusters and **Istio** for cross-cluster communication. **AWS ACM certificate** is used for **SSL termination**.

### **Objective**

* Expose **Grafana** from multiple EKS clusters (dev, prod, and analytics) via a **single domain**.
* Route traffic to different Grafana instances based on URL paths: `/dev`, `/prod`, `/analytics`.
* Use **AWS ACM certificates** for SSL termination.

---

### **Prerequisites**

* AWS account with **IAM permissions** to create and manage ALBs and ACM certificates.
* **AWS CLI** and **kubectl** installed and configured.
* **EKS clusters** for **dev**, **prod**, and **analytics**.
* **Istio** installed in all clusters.
* **AWS ALB Ingress Controller** installed in all clusters.

---

### **Step 1: Install AWS ALB Ingress Controller**

The **AWS ALB Ingress Controller** is required to manage AWS ALBs and route traffic.

#### **Installation in Each Cluster (dev, prod, and analytics)**:

1. Run the following command to install the **AWS ALB Ingress Controller** in each of your EKS clusters:

   ```bash
   kubectl apply -k github.com/kubernetes-sigs/aws-alb-ingress-controller//deploy/kubernetes/overlays/aws/v2.0
   ```

2. **Set up IAM permissions** for the ALB Ingress Controller. Follow the [official AWS ALB Ingress Controller documentation](https://github.com/kubernetes-sigs/aws-alb-ingress-controller/blob/main/docs/install/iam_policy.md) to create the necessary IAM role and permissions.

---

### Step 1: Install Istio on All Clusters (dev, prod, and analytics)

Start by installing **Istio** in each of your three clusters (dev, prod, and analytics).

1. **Download and Install Istio**:

   ```bash
   curl -L https://istio.io/downloadIstio | sh -
   cd istio-<version>/bin
   ```

2. **Install Istio in your clusters**:

   * First, **select your context for each cluster** using `kubectl`:

     ```bash
     kubectl config use-context <dev-cluster-context>
     istioctl install --set profile=demo -y
     ```

   * Repeat the above installation steps for the **prod** and **analytics** clusters.

3. **Verify Istio is running**:
   Check that Istio components are running in all clusters:

   ```bash
   kubectl get pods -n istio-system
   ```

### Step 3: Create AWS ACM Certificate and Secret in Kubernetes

To enable **SSL/TLS** termination with your AWS ACM certificate:

1. **Get your ACM certificate ARN** from AWS Console (or use AWS CLI).

2. **Create a Kubernetes Secret** for the ACM certificate.

   **Export the ACM certificate using AWS CLI**:

   ```bash
   aws acm export-certificate --certificate-arn <your-certificate-arn> --passphrase fileb://password.txt --output text
   ```

3. **Create Kubernetes Secret for the ACM certificate**:

   ```bash
   kubectl create secret tls grafana-tls \
     --cert=certificate.pem \
     --key=private-key.pem \
     --namespace=default
   ```
---

### **Step 3: Set Up Ingress Gateway and VirtualService in the Prod Cluster**

In the **prod cluster**, we will configure the **Ingress Gateway** and **VirtualService** for handling traffic and routing it to the **dev** and **analytics** clusters.

#### **1. Create Ingress Gateway for SSL Termination (Prod Cluster)**

This configuration allows the **prod cluster** to terminate SSL traffic using the **ACM certificate ARN**.

**`grafana-gateway.yaml`** (for **prod cluster**):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grafana-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "activity.v2.liquide.life"
    tls:
      mode: SIMPLE
      credentialName: grafana-tls  # ACM certificate stored as a Kubernetes secret
      minProtocolVersion: TLSV1_2
```

#### **2. Create VirtualService to Route Traffic Based on Path (Prod Cluster)**

The **VirtualService** will route traffic to the correct Grafana service based on the URL path (e.g., `/prod`, `/dev`, `/analytics`).

**`grafana-virtualservice.yaml`** (for **prod cluster**):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana-virtualservice
  namespace: default
spec:
  hosts:
  - "activity.v2.liquide.life"
  http:
  - match:
    - uri:
        prefix: "/analytics"
    route:
    - destination:
        host: grafana-analytics-service.default.svc.cluster.local
        port:
          number: 80
  - match:
    - uri:
        prefix: "/dev"
    route:
    - destination:
        host: grafana-dev-service.default.svc.cluster.local
        port:
          number: 80
  - match:
    - uri:
        prefix: "/prod"
    route:
    - destination:
        host: grafana-prod-service.default.svc.cluster.local
        port:
          number: 80
```

#### **Apply Gateway and VirtualService**:

```bash
kubectl apply -f grafana-gateway.yaml
kubectl apply -f grafana-virtualservice.yaml
```

---

### **Step 4: Expose Grafana Services in Each Cluster**

You need to expose **Grafana** services in the **dev**, **prod**, and **analytics** clusters.

#### **Example Service for Prod Cluster (grafana-prod-service.yaml)**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-prod-service
  namespace: default
spec:
  selector:
    app: grafana-prod  # Match this with the label of your Grafana pod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000  # Default Grafana port
```

#### **Apply Service**:

```bash
kubectl apply -f grafana-prod-service.yaml  # Repeat for dev and analytics services
```

---

### **Step 5: Set Up DNS for a Single URL**

The **prod cluster** is the entry point, and the **ALB** created by the **AWS ALB Ingress Controller** will route traffic to the correct cluster based on the paths.

1. **Find the ALB's DNS name**:
   Run the following command to get the DNS name of the ALB:

   ```bash
   kubectl describe ingress grafana-ingress -n default
   ```

   You will see the ALB DNS name, which will look something like:

   ```text
   activity-v2-liquide-life-abcde.elb.amazonaws.com
   ```

2. **Update DNS**:
   In your DNS provider (e.g., Route 53), create a record for **`activity.v2.liquide.life`** and point it to the **ALB's DNS name**.

   Example:

   ```text
   activity.v2.liquide.life -> activity-v2-liquide-life-abcde.elb.amazonaws.com
   ```

---

### **Step 6: Verify the Setup**

Once everything is deployed and DNS is configured, verify that the routing is working correctly:

1. **Test each environment**:

   * `https://activity.v2.liquide.life/analytics` → Should route to the **Grafana Analytics** instance.
   * `https://activity.v2.liquide.life/dev` → Should route to the **Grafana Dev** instance.
   * `https://activity.v2.liquide.life/prod` → Should route to the **Grafana Prod** instance.

2. **Verify SSL/TLS**:

   * Ensure that traffic is served over **HTTPS** using your **ACM certificate**.

3. **Monitor Logs**:

   * Use **Istio logs** and **AWS ALB Ingress Controller logs** to troubleshoot any routing issues.

---

### **Step 7: Optional Enhancements**

1. **Authentication**: Enable **authentication** on your **Grafana** instances if needed (e.g., using **OAuth**, **LDAP**).

2. **Scaling**: If you expect heavy traffic, scale **Grafana** replicas in each cluster using the `replicas` setting in the **Grafana Deployment**.

---

### **Summary**

* **Single URL**: `activity.v2.liquide.life` is used for accessing all three Grafana instances across dev, prod, and analytics.
* **AWS ALB** in **prod cluster** routes traffic to different services (Grafana) in other clusters based on paths (`/dev`, `/prod`, `/analytics`).
* **SSL/TLS termination** is handled by **AWS ACM certificates**.
* **Istio** handles cross-cluster routing, ensuring traffic is directed to the appropriate cluster.

