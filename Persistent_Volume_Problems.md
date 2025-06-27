# 🎯 Common Kubernetes Persistent Volume Problems and How to Fix Them

Persistent storage is critical for stateful workloads like:

* Databases
* Prometheus
* Grafana
* Loki
* Elasticsearch

…but **it’s also the #1 source of confusion and cluster issues**.

This guide lists the **most common problems**, **why they happen**, and **how to solve them step by step**.

---

## 🟢 Problem 1: PVC Stuck in Pending

**Symptoms:**

* `kubectl get pvc` shows `STATUS: Pending`
* No pods start, or pods stuck waiting for volume

**Why This Happens:**
✅ You didn’t specify a `storageClassName`
✅ The default StorageClass is missing or misconfigured
✅ The volume can’t be provisioned in the right Availability Zone
✅ Your AWS EBS limits were reached

**How to Fix:**

1. **Check which storage class is assigned:**

   ```bash
   kubectl describe pvc <pvc-name>
   ```

   Look for:

   ```
   StorageClass: gp2
   ```
2. **Set `storageClassName` explicitly in your YAML:**

   ```yaml
   storageClassName: gp3
   ```
3. **Verify your default StorageClass:**

   ```bash
   kubectl get storageclass
   ```
4. **Delete the stuck PVC and re-apply:**

   ```bash
   kubectl delete pvc <pvc-name>
   kubectl apply -f your-pvc.yaml
   ```

**Important Tip:**

> Always **explicitly set `storageClassName`**. Never rely on defaults.

---

## 🟢 Problem 2: Read-Only Filesystem Errors

**Symptoms:**

* Pods crash with messages like:

  ```
  permission denied
  read-only file system
  ```
* Logs show errors writing to disk

**Why This Happens:**
✅ `securityContext` is set to `readOnlyRootFilesystem: true`
✅ No persistent volume mounted at the directory Loki, Prometheus, or your app tries to write

**How to Fix:**

1. **Enable write access:**

   ```yaml
   securityContext:
     readOnlyRootFilesystem: false
   ```
2. **Make sure `persistence.enabled: true`**
3. **Match `mountPath` with your app configuration:**
   For example:

   ```yaml
   persistence:
     mountPath: /loki
   ```

   and in your config:

   ```yaml
   active_index_directory: /loki/index
   ```

**Important Tip:**

> If you mount `/data`, all directories should be inside `/data` (e.g., `/data/index`, `/data/cache`).

---

## 🟢 Problem 3: Grafana (or any app) Loses Data After Restart

**Symptoms:**

* Dashboards disappear after pod restart
* Application state resets to default

**Why This Happens:**
✅ No PersistentVolumeClaim attached
✅ Persistence disabled in Helm values

**How to Fix:**

1. Enable persistence:

   ```yaml
   persistence:
     enabled: true
     storageClassName: gp3
     size: 10Gi
   ```
2. Reinstall or upgrade your Helm release:

   ```bash
   helm upgrade my-release my-chart -f values.yaml
   ```
3. Verify the PVC is bound:

   ```bash
   kubectl get pvc
   ```

---

## 🟢 Problem 4: Availability Zone Mismatch

**Symptoms:**

* PVC Pending with events:

  ```
  0/3 nodes are available: 3 node(s) had no available volume zone
  ```
* EBS volume created in a zone where no pods are scheduled

**Why This Happens:**
✅ Your EBS volume was provisioned in AZ `us-east-1a`, but your pod is scheduled on a node in `us-east-1b`.

**How to Fix:**

1. Ensure your StorageClass uses:

   ```yaml
   volumeBindingMode: WaitForFirstConsumer
   ```

   This delays volume provisioning until the pod is scheduled.

2. If using Helm, most charts default to this. Otherwise, define a custom StorageClass:

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: gp3
   provisioner: ebs.csi.aws.com
   volumeBindingMode: WaitForFirstConsumer
   parameters:
     type: gp3
   ```

**Important Tip:**

> Avoid pre-creating PVs if your nodes span multiple AZs.

---

## 🟢 Problem 5: Multiple Components Using Different StorageClasses

**Symptoms:**

* Some PVCs bind (`Bound`)
* Others remain pending
* Hard to track which components failed

**Why This Happens:**
✅ Each component (Prometheus, Grafana, Alertmanager) has **its own storage configuration block**.

**How to Fix:**
Always specify `storageClassName` for **every component**. For example:

```yaml
grafana:
  persistence:
    enabled: true
    storageClassName: gp3

prometheus:
  server:
    persistentVolume:
      enabled: true
      storageClass: gp3

  alertmanager:
    persistentVolume:
      enabled: true
      storageClass: gp3

loki:
  persistence:
    enabled: true
    storageClassName: gp3
```

**Important Tip:**

> Helm charts don’t share a single storage class setting across all subcharts.

---

## 🟢 Checklist Before Deploying Stateful Apps

✅ Confirm your StorageClasses:

```bash
kubectl get storageclass
```

✅ Decide which StorageClass you want (`gp3` recommended).

✅ For each component:

* **Set `persistence.enabled: true`.**
* **Set `storageClassName: gp3`.**
* **Set a reasonable size.**

✅ If using Helm, do:

```bash
helm upgrade --install my-release my-chart -f values.yaml
```

✅ After deployment, always check:

```bash
kubectl get pvc
kubectl get pods
kubectl describe pvc <name>
```

---

## 🟢 Pro Tips

🌟 Always use `volumeBindingMode: WaitForFirstConsumer` in AWS EBS.
🌟 Prefer `gp3` over `gp2` for performance and cost.
🌟 Don’t rely on default storage classes.
🌟 Validate your node AZs align with your volumes.
🌟 If upgrading Helm releases, delete `Pending` PVCs first.

---

## 🟢 Helpful Commands Cheat Sheet

**List PVCs and their status:**

```bash
kubectl get pvc
```

**Describe why a PVC is pending:**

```bash
kubectl describe pvc <pvc-name>
```

**Patch a StorageClass to remove default:**

```bash
kubectl patch storageclass gp2 -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

**Make gp3 the default:**

```bash
kubectl patch storageclass gp3 -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

**Delete stuck PVCs:**

```bash
kubectl delete pvc <name>
```

**Upgrade Helm release with changes:**

```bash
helm upgrade my-release my-chart -f values.yaml
```

---

✅ **Remember:**
Persistent storage problems are usually **misconfigured storage classes, missing permissions, or AZ mismatches.**
Check logs, describe PVCs, and explicitly define `storageClassName` everywhere.
