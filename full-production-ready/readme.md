# Loki Observability Stack – Helm Deployment Guide

This repository provides a **Helm-based deployment** of a complete observability stack for Kubernetes, including:

*  Loki log storage backed by S3
*  Promtail log collection
*  Grafana dashboards with Google OAuth and SMTP alerts
*  Prometheus metrics
*  Ingress with ALB and automatic HTTPS redirects

Everything is configured **in a single `values.yaml`** for simplicity.

---

##  What’s Included

* **Loki** – Central log storage in S3
* **Promtail** – Kubernetes log shipping
* **Grafana** – Secure dashboards with OAuth login and email notifications
* **Prometheus** – Metrics monitoring
* **ALB Ingress** – Secure HTTPS access with automatic redirect

---

##  Prerequisites

Before you deploy, ensure:

* You have a **Kubernetes cluster** (EKS recommended)
* **Helm v3** is installed
* You created an **S3 bucket for Loki storage**
* The **AWS credentials you configure have read and write permissions to this bucket**
  *(permissions needed: `s3:PutObject`, `s3:GetObject`, `s3:ListBucket`)*
* A **DNS domain** routed to your ALB Ingress
* A **TLS certificate** (via ACM or cert-manager)
* **Google OAuth credentials** for Grafana login
* **SMTP credentials** (optional, for email alerts)

---

##  Quick Start – Default Namespace

* **Add the Grafana Helm repository:**

```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

* **Deploy the observability stack:**

Everything will run in the **default namespace**:

```
helm upgrade --install observability grafana/loki-stack \
  -f values.yaml
```

* **Apply your Ingress manifest:**

Make sure your ALB Ingress has the annotation for HTTPS redirect:

```
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

> **Note:** This ensures any HTTP traffic is automatically redirected to HTTPS.

* **Verify:**

* Pods are running (`kubectl get pods`)
* ALB is provisioned and DNS resolves to your endpoint
* You can log in to Grafana using Google OAuth

---

##  Configuration Overview

Everything is defined in **your single `values.yaml`** file, including:

---

###  Loki S3 Storage

Loki will store logs in S3.
**Important:** The credentials in `values.yaml` must have permissions for:

* `s3:PutObject`
* `s3:GetObject`
* `s3:ListBucket`

This is required for Loki to write log chunks and retrieve indexes.

---

###  SMTP Email Alerts

SMTP settings allow Grafana to send alert notifications.

**Example Configuration in `values.yaml`:**

* **Host:** `smtp.gmail.com:587`
* **User:** Your email (e.g., `alerts@yourcompany.com`)
* **Password:** Gmail App Password (never use your main password)
* **From Name:** "Grafana Alerts"

---

###  Google OAuth Login

OAuth secures Grafana dashboards to your team.

**Key settings:**

* **client\_id** and **client\_secret**: from Google Developer Console
* **allowed\_domains**: restricts login to your domain
* **GF\_SERVER\_ROOT\_URL**: must match your DNS (e.g., `https://grafana.yourdomain.com`)
* **Redirect URI:**

  ```
  https://<your-domain>/login/generic_oauth
  ```

---

###  Ingress with ALB & HTTPS Redirect

This deployment uses **AWS ALB Ingress**.

* Critical annotation:

```
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

This automatically redirects all HTTP requests to HTTPS, keeping credentials secure.

* **Make sure:**

* Your ALB Ingress is linked to a valid TLS certificate
* Your DNS domain points to the ALB
* You validate the certificate status before logging in

---

##  Persistent Storage

Each component will use persistent volumes:

| Component  | Storage Class | Size | Purpose               |
| ---------- | ------------- | ---- | --------------------- |
| Loki       | gp3           | 50Gi | Log data & indexes    |
| Grafana    | gp3           | 50Gi | Dashboards & settings |
| Prometheus | gp3           | 50Gi | Metrics time series   |

Adjust these in `values.yaml` if you expect higher log volume or retention.

---

##  Security Best Practices

* **S3 credentials:** Use Kubernetes Secrets instead of plaintext
* **OAuth:** Always restrict to your domain
* **Ingress TLS:** Always enable certificates
* **SSL Redirect:** Ensure ALB annotation is active
* **RBAC:** Use least privilege for service accounts
* **Rotate credentials regularly**

---

##  Upgrading

To upgrade your deployment:

```
helm upgrade observability grafana/loki-stack \
  -f values.yaml
```

Since everything is in the **default namespace**, no extra flags are needed.

If you update Ingress or secrets, re-apply them separately.

---

##  References

* [Grafana Loki Documentation](https://grafana.com/docs/loki/latest/)
* [Promtail Docs](https://grafana.com/docs/loki/latest/clients/promtail/)
* [Grafana OAuth Setup](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/#configure-oauth)
* [Helm Charts](https://github.com/grafana/helm-charts)
* [AWS ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
