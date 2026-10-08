# Kubernetes ConfigMap: Quick Guide

## What is it?
A ConfigMap stores **non-secret settings** (key-value pairs) outside your app image.
Change the config without rebuilding the image.

- Use for: app settings, URLs, log level, config files
- Do NOT use for: passwords, tokens (use **Secret**)
- Max size: 1 MiB

---

## Step 1: Create a ConfigMap

**From literals (quick):**
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=dev \
  --from-literal=LOG_LEVEL=debug
```

**From a file:**
```bash
echo "db.host=mysql" > app.properties
kubectl create configmap file-config --from-file=app.properties
```

**From YAML (best for real projects):**

`configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "dev"          # values must be strings, so quote numbers/booleans
  LOG_LEVEL: "debug"
  app.properties: |       # | = multi-line file content
    db.host=mysql
    db.port=3306
```
```bash
kubectl apply -f configmap.yaml
```

---

## Step 2: Check it
```bash
kubectl get cm                      # list
kubectl describe cm app-config      # readable details
kubectl get cm app-config -o yaml   # full YAML
```

---

## Step 3: Use it in a Pod

### A) As environment variables

`pod-env.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "env && sleep 3600"]
      envFrom:                    # ALL keys become env vars
        - configMapRef:
            name: app-config
      env:                        # OR pick one key
        - name: MY_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```
```bash
kubectl apply -f pod-env.yaml
kubectl exec env-pod -- printenv LOG_LEVEL
```

### B) As a file (volume)

`pod-volume.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vol-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cfg
          mountPath: /etc/config      # each key becomes a file here
  volumes:
    - name: cfg
      configMap:
        name: app-config
```
```bash
kubectl apply -f pod-volume.yaml
kubectl exec vol-pod -- ls /etc/config
kubectl exec vol-pod -- cat /etc/config/app.properties
```

---

## Step 4: Update it
```bash
kubectl edit cm app-config          # edit live
# or edit configmap.yaml and:
kubectl apply -f configmap.yaml
```

| Used as | Auto-updates? |
|---|---|
| Env variables | No. Restart: `kubectl rollout restart deploy <name>` |
| Mounted volume | Yes (about 1 min), but not with `subPath` |

---

## Step 5: Clean up
```bash
kubectl delete pod env-pod vol-pod
kubectl delete cm app-config file-config
```

---

## Most-used commands

| Task | Command |
|---|---|
| Create (literal) | `kubectl create cm NAME --from-literal=K=V` |
| Create (file) | `kubectl create cm NAME --from-file=FILE` |
| Create (YAML) | `kubectl apply -f cm.yaml` |
| Generate YAML only | `kubectl create cm NAME --from-literal=K=V --dry-run=client -o yaml` |
| List / details / YAML | `kubectl get cm` / `describe cm NAME` / `get cm NAME -o yaml` |
| Edit | `kubectl edit cm NAME` |
| Delete | `kubectl delete cm NAME` |
| Check env in Pod | `kubectl exec POD -- env` |
| Restart Deployment | `kubectl rollout restart deploy NAME` |

---

## Common errors

| Error | Fix |
|---|---|
| `CreateContainerConfigError` | ConfigMap or key name is wrong or missing. Check `kubectl describe pod POD` |
| `configmap not found` | Wrong namespace. Use `-n NAMESPACE` |
| Old value after update | Env var or `subPath`. Restart the Pod/Deployment |
| Number/boolean error in YAML | Quote it: `"8080"`, `"true"` |

---

## Remember
1. ConfigMap = non-secret settings. Secrets = passwords.
2. Create with YAML, use as **env vars** or **files**.
3. Env vars need a restart to update. Volume files update on their own.
