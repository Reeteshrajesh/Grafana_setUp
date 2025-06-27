# Enforcing HTTPS in AWS ALB Ingress for Kubernetes

---

## 🚀 Overview

This documentation explains **why enforcing HTTPS globally is essential when using AWS Application Load Balancer (ALB) Ingress Controller** with Kubernetes, and exactly how to do it.

---

## 🎯 Problem Statement

Many teams set up HTTPS termination with ALB, but **forget to enforce HTTP-to-HTTPS redirection everywhere**.  

A common misconfiguration looks like this:
- The home page (`/`) loads over plain `http://`.
- Other routes redirect to `https://`.

This **partial HTTPS setup is highly insecure.**

---

## ⚠️ Critical Problems If You Don't Enforce HTTPS Everywhere

**If any request is served over HTTP, you expose your users and application to serious risks:**

1. 🔓 **No Encryption**
   - Traffic between the user and ALB is **plain text**.
   - Anyone on the network (e.g., public Wi-Fi) can intercept all data.

2. 🕵️ **Session Hijacking**
   - Cookies and session tokens are sent unencrypted.
   - Attackers can steal them and impersonate users.

3. ⚠️ **Man-in-the-Middle Attacks**
   - An attacker can inject malicious scripts or modify page content.
   - This is especially dangerous if you load JavaScript libraries over HTTP.

4. 🚫 **Compliance Violations**
   - Fails basic security requirements of:
     - GDPR
     - SOC2
     - PCI DSS
     - HIPAA
   - You could face legal and financial penalties.

5. ❌ **Browser Warnings**
   - Modern browsers display:
     > “Not Secure”
   - Users lose trust in your application.

6. 🧨 **HSTS Ineffectiveness**
   - Without enforcing HTTPS and HSTS headers, browsers will not automatically upgrade insecure requests.

> **In short:** Even a single unsecured HTTP page can compromise your entire application.

---

## ✅ Best Practice: Always Enforce HTTPS with AWS ALB Ingress

To protect your users and infrastructure, follow these steps:

---

### 1️⃣ Terminate TLS at the ALB

In your Kubernetes Ingress manifest annotations, configure HTTPS termination:

```yaml
annotations:
  kubernetes.io/ingress.class: alb
  alb.ingress.kubernetes.io/target-type: ip
  alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<your-region>:<your-account>:certificate/<certificate-id>
  alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
  alb.ingress.kubernetes.io/scheme: internet-facing
```

This ensures the ALB can serve HTTPS traffic using your ACM certificate.

---

### 2️⃣ Enforce HTTP-to-HTTPS Redirect

**This is the critical step most people forget.**

Add this annotation to **force all HTTP traffic to redirect automatically to HTTPS**:

```yaml
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

✅ **Result:**

* Every `http://` request receives an automatic `301 Moved Permanently` redirect to `https://`.
* Users cannot browse any unsecured pages.

---

### 3️⃣ (Optional) Enable HSTS Headers

Set this response header in your application or ingress:

```yaml
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

This instructs browsers to:

* Always use HTTPS
* Never downgrade to HTTP
* Preload your domain in browser lists

---

## 🔍 Example: Full Ingress YAML for AWS ALB Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:123456789012:certificate/abcdefg-1234-5678
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
  - host: example.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## 🔑 Quick Testing Checklist

✅ Try `http://yourdomain.com` – confirm automatic redirect to HTTPS
✅ Try `https://yourdomain.com` – confirm it loads securely
✅ Verify no pages are accessible over plain HTTP
✅ Check for `Strict-Transport-Security` headers
✅ Confirm your cookies have the `Secure` flag

---

## 💡 Takeaway

> **If even one route is accessible over HTTP, your entire Kubernetes application behind ALB is at risk.**

✅ Always enforce HTTPS globally.
✅ Always redirect HTTP to HTTPS.
✅ Always test your configuration end to end.

---

## 🙌 Contributing

Have ideas or improvements for this documentation?
Feel free to open a PR or an issue!

---

## 📚 Further Reading

* [AWS ALB Ingress Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)
* [Mozilla Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security)
* [OWASP Cheat Sheet: Transport Layer Protection](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)

---


---

✅ **Pro Tips for Your Repo:**
- Name it: `aws-alb-ingress-https-enforcement`
- Add a screenshot of browser warnings when loading HTTP
- Include example curl commands to test redirects

This will make it **clear, authoritative, and useful** for anyone deploying Kubernetes with ALB.  

