## 📧 SMTP Configuration for Grafana

### 🔍 What is SMTP in Grafana?

SMTP (Simple Mail Transfer Protocol) allows **Grafana to send emails**. This includes:

* 📬 User invites & password reset emails
* 🚨 Alert notifications (e.g., threshold breach)
* 📝 Contact point emails (in alerting UI)
* ✅ Welcome emails on new sign-ups

Without SMTP configured, you'll see errors like:

```
SMTP not configured, check your grafana.ini config file's [smtp] section
```

---

### 💡 Why SMTP is Important?

| Feature                    | Without SMTP | With SMTP |
| -------------------------- | ------------ | --------- |
| Invite users via email     | ❌            | ✅         |
| Reset forgotten passwords  | ❌            | ✅         |
| Alert via email            | ❌            | ✅         |
| Email confirmation/welcome | ❌            | ✅         |

> 📌 **Important for production environments** where automated alerting or password recovery is expected.

---

### 🛠️ How to Configure SMTP in Grafana (via Helm `values.yaml`)

Update your `values.yaml` under `grafana.grafana.ini`:

```yaml
grafana:
  grafana.ini:
    smtp:
      enabled: true
      host: smtp.gmail.com:587
      user: your-email@xyz.abc
      password: your-app-password  # Use App Password if Gmail
      from_address: your-email@xyz.abc
      from_name: "Grafana Alerts"
      skip_verify: false
      startTLS_policy: "OpportunisticStartTLS"

    emails:
      welcome_email_on_sign_up: true
```

#### 📌 Replace with:

* `user` → your real SMTP email (e.g., `alerts@xyz.abc`)
* `password` → **App Password** if Gmail (create at [Google App Passwords](https://myaccount.google.com/apppasswords))
* `from_name` → How it appears in the recipient’s inbox

---

### 📦 Deploy the Change

```bash
helm upgrade --install loki grafana/loki-stack -f values.yaml -n monitoring
```

---

### ✅ Test Email in Grafana

1. Go to **Alerting → Contact Points → New Contact Point**
2. Choose **email**
3. Enter recipient and send **Test Notification**

You should receive an email if SMTP is working.

---

### 🧪 Troubleshooting

* Check `grafana.log` for any email errors.
* Ensure your cloud provider (AWS, GCP, etc.) allows outbound traffic on port `587`.
* If using Gmail, verify:

  * App Password is enabled.
  * 2FA is enabled on your account.
