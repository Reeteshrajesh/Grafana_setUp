# 1
## 📘 Grafana Google OAuth + Role-Based Access Control (RBAC)

> Secure your Grafana dashboard with Google OAuth login and assign roles (Admin, Editor, Viewer) based on the user's email domain.

---

### 🎯 Objective

* Allow login **only to users with `@xyz.abc` domain** (e.g. `user1@xyz.abc`).
* Assign roles dynamically:

  * `GrafanaAdmin` → [user1@xyz.abc](mailto:user1@xyz.abc)
  * `Editor` → [user2@xyz.abc](mailto:user2@xyz.abc), [user3@xyz.abc](mailto:user3@xyz.abc), [user4@xyz.abc](mailto:user4@xyz.abc)
  * `Viewer` → everyone else under `@xyz.abc`
* Expose Grafana securely over HTTPS: `https://grafana.example.xyz`
* Auto-provision users with correct roles on first login via Google

---

### ✅ Prerequisites

* Kubernetes cluster (e.g., AWS EKS)
* Helm installed and configured
* Domain: `grafana.example.xyz` (routed via Ingress & DNS)
* TLS Certificate (e.g., via AWS ACM or Cert Manager)
* [Google OAuth2 credentials](https://console.developers.google.com/)

  * Authorized Redirect URI: `https://grafana.example.xyz/login/generic_oauth`

---

### 📁 File: `values.yaml`

Here’s the `values.yaml` snippet to use with the `grafana/loki-stack` Helm chart:

```yaml
grafana:
  enabled: true

  image:
    tag: latest

  persistence:
    enabled: true
    size: 10Gi
    accessModes:
      - ReadWriteOnce
    storageClassName: gp3

  env:
    GF_SERVER_ROOT_URL: "https://grafana.example.xyz"
    GF_LOG_LEVEL: "debug"
    GF_AUTH_GENERIC_OAUTH_LOGGING: true

  grafana.ini:
    server:
      root_url: "https://grafana.example.xyz"
      serve_from_sub_path: false

    auth.generic_oauth:
      enabled: true
      name: Google
      allow_sign_up: true
      client_id: <your-google-client-id>
      client_secret: <your-google-client-secret>
      scopes: openid profile email
      auth_url: https://accounts.google.com/o/oauth2/auth
      token_url: https://oauth2.googleapis.com/token
      api_url: https://www.googleapis.com/oauth2/v3/userinfo
      use_refresh_token: true
      auto_login: false
      email_attribute_path: email
      allowed_domains: xyz.abc

      role_attribute_path: "email == 'user1@xyz.abc' ? 'GrafanaAdmin' : (email == 'user2@xyz.abc' || email == 'user3@xyz.abc' || email == 'user4@xyz.abc') ? 'Editor' : 'Viewer'"
      allow_assign_grafana_admin: true

  sidecar:
    datasources:
      enabled: true
      label: ""
      labelValue: ""
      maxLines: 1000
      enableLogVolume: true
```

---

### 🧠 Configuration Breakdown

| Config                        | Description                                               |
| ----------------------------- | --------------------------------------------------------- |
| `GF_SERVER_ROOT_URL`          | Tells Grafana to use this domain for redirects            |
| `client_id` / `client_secret` | OAuth credentials from Google Developer Console           |
| `allowed_domains`             | Only allows users from `@xyz.abc` to log in               |
| `email_attribute_path`        | Extracts email from the UserInfo endpoint                 |
| `role_attribute_path`         | Maps user emails to Grafana roles using ternary logic     |
| `allow_assign_grafana_admin`  | Required to assign the GrafanaAdmin role programmatically |

---

### 🔑 Role Mapping Logic (Explained)

```yaml
role_attribute_path: >
  email == 'user1@xyz.abc' ? 'GrafanaAdmin' :
  (email == 'user2@xyz.abc' || email == 'user3@xyz.abc' || email == 'user4@xyz.abc') ? 'Editor' :
  'Viewer'
```

This behaves like:

| Email                                               | Role           |
| --------------------------------------------------- | -------------- |
| `user1@xyz.abc`                                     | `GrafanaAdmin` |
| `user2@xyz.abc` / `user3@xyz.abc` / `user4@xyz.abc` | `Editor`       |
| Anyone else @xyz.abc                                | `Viewer`       |
| Any other domain                                    | ❌ **Blocked**  |

---

### 🚀 Deploy Helm Chart

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana/loki-stack -f values.yaml -n monitoring --create-namespace
```

---

### 🌐 Ingress (Sample)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: <your-acm-certificate-arn>
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
spec:
  rules:
    - host: grafana.example.xyz
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: loki-grafana
                port:
                  number: 80
```

---

### 🧪 Test the Setup

1. Visit: `https://grafana.example.xyz`
2. Click **Login with Google**
3. Try:

   * [user1@xyz.abc](mailto:user1@xyz.abc) → should become **GrafanaAdmin**
   * user2–[user4@xyz.abc](mailto:user4@xyz.abc) → should be **Editor**
   * anyone else with @xyz.abc → **Viewer**
   * others → **Access Denied**

---

### 🧹 Troubleshooting Tips

| Problem                       | Solution                                      |
| ----------------------------- | --------------------------------------------- |
| Everyone gets `Viewer`        | Check `role_attribute_path` syntax            |
| Login fails                   | Make sure `allowed_domains` matches exactly   |
| Admin role not applied        | Set `allow_assign_grafana_admin: true`        |
| OAuth loop or redirect errors | Validate `GF_SERVER_ROOT_URL` and Ingress DNS |

---

### ✅ Summary

With this setup:

* 🔐 Only your team can access Grafana
* 🧑‍💻 Roles are auto-assigned on login
* ✅ Uses secure Google OAuth login


-------------
-----------------
---------------
# 2
## 🛠️ Manual Role Assignment in Grafana (Disable Role Sync)

In some cases, you may want to **manually manage user roles** (Admin, Editor, Viewer) directly from the Grafana UI instead of assigning them automatically via OAuth login.

### 🔍 Problem:

By default, Grafana's `role_attribute_path` automatically assigns roles at login, based on user email or attributes. This makes it impossible to change roles from the Grafana UI, as roles get overwritten on each login.

---

### ✅ Solution: Disable Role Sync

To allow **manual role management**, update the `grafana.ini` section in your Helm `values.yaml` like this:

```yaml
grafana:
  grafana.ini:
    server:
      root_url: "https://grafana-analytics.example.xyz"
      serve_from_sub_path: false

    auth.generic_oauth:
      enabled: true
      name: Google
      allow_sign_up: true
      client_id: <your-client-id>
      client_secret: <your-client-secret>
      scopes: openid profile email
      auth_url: https://accounts.google.com/o/oauth2/auth
      token_url: https://oauth2.googleapis.com/token
      api_url: https://www.googleapis.com/oauth2/v3/userinfo
      use_refresh_token: true
      auto_login: false
      email_attribute_path: email
      allowed_domains: xyz.abc

      # ✅ Disable role sync for manual UI-based role control
      skip_org_role_sync: true
      allow_assign_grafana_admin: true  # Optional: allows manual admin assignment
```

> 🔁 **Note**: Remove or comment out any existing `role_attribute_path` setting when using this approach.

---

### 👤 Manual Role Assignment Steps

1. Deploy or upgrade your Grafana Helm chart:

   ```bash
   helm upgrade --install loki grafana/loki-stack -f values.yaml -n monitoring
   ```

2. Login via Google OAuth using any email from the allowed domain (e.g., `@xyz.abc`).

3. In Grafana:

   * Go to **Administration → Users**
   * Select the user
   * Change their role to **Admin**, **Editor**, or **Viewer**

---

### 🎯 Benefit

This method gives you full control over role assignment without editing the Helm config or redeploying each time a user’s role changes.


