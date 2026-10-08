# Kubernetes ConfigMap: From Scratch to End

A beginner-friendly, copy-paste guide. Every step has: **what it does -> command -> what you should see**.

---

## Table of Contents

1. [What is a ConfigMap?](#1-what-is-a-configmap)
2. [When to use (and not use) a ConfigMap](#2-when-to-use-and-not-use-a-configmap)
3. [Prerequisites and setup](#3-prerequisites-and-setup)
4. [Step 1: Create a namespace](#step-1-create-a-namespace)
5. [Step 2: Create a ConfigMap (3 ways)](#step-2-create-a-configmap-3-ways)
6. [Step 3: View and inspect a ConfigMap](#step-3-view-and-inspect-a-configmap)
7. [Step 4: Use ConfigMap as environment variables](#step-4-use-configmap-as-environment-variables)
8. [Step 5: Use ConfigMap as a mounted file (volume)](#step-5-use-configmap-as-a-mounted-file-volume)
9. [Step 6: Update a ConfigMap](#step-6-update-a-configmap)
10. [Step 7: Immutable ConfigMap](#step-7-immutable-configmap)
11. [Step 8: Clean up](#step-8-clean-up)
12. [Full end-to-end example](#full-end-to-end-example-nginx--configmap)
13. [Important commands cheat sheet](#important-commands-cheat-sheet)
14. [Common errors and fixes](#common-errors-and-fixes)
15. [Quick revision](#quick-revision)

---

## 1. What is a ConfigMap?

A **ConfigMap** is a Kubernetes object that stores **non-secret configuration data** as **key-value pairs**.

Simple analogy: think of it as a **settings file that lives outside your app**.

Without ConfigMap:
```
App image has DB_HOST=localhost baked in -> to change it, rebuild the image.
```

With ConfigMap:
```
App image stays the same. DB_HOST comes from a ConfigMap -> change the ConfigMap, not the image.
```

**Key facts**
- Stores plain text data (not encrypted).
- Max size: **1 MiB**.
- Lives in a namespace. A Pod can only use ConfigMaps from its own namespace.
- Pods consume it as: **environment variables**, **files in a volume**, or **command-line arguments**.

---

## 2. When to use (and not use) a ConfigMap

| Use ConfigMap for | Do NOT use ConfigMap for |
|---|---|
| App settings (`LOG_LEVEL=debug`) | Passwords, API keys, tokens (use **Secret**) |
| Environment-specific values (dev/test/prod URLs) | Large binary files or big data (over 1 MiB) |
| Config files (`nginx.conf`, `app.properties`) | Data that must be encrypted |
| Feature flags | Frequently changing application data (use a database) |

---

## 3. Prerequisites and setup

You need:
- A running Kubernetes cluster (minikube, kind, Docker Desktop, or cloud)
- `kubectl` installed

**Check kubectl works:**
```bash
kubectl version --client
```

**Check the cluster is reachable:**
```bash
kubectl cluster-info
kubectl get nodes
```
You should see at least one node in `Ready` status.

**No cluster yet? Quickest option (minikube):**
```bash
minikube start
```

Create a working folder for this tutorial:
```bash
mkdir configmap-demo && cd configmap-demo
```

---

## Step 1: Create a namespace

**Why:** Keeps our practice objects separate so cleanup is easy.

```bash
kubectl create namespace demo
```

Set it as default for your session so you don't type `-n demo` every time:
```bash
kubectl config set-context --current --namespace=demo
```

Verify:
```bash
kubectl config view --minify | grep namespace
```

---

## Step 2: Create a ConfigMap (3 ways)

### Way A: From literal values (fastest, good for a few keys)

**Command:**
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=development \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=APP_PORT=8080
```

**Explanation:**
- `create configmap app-config` -> make a ConfigMap named `app-config`
- `--from-literal=KEY=VALUE` -> add one key-value pair; repeat for more keys

---

### Way B: From a file (good for existing config files)

**1. Create a config file:**
```bash
cat > app.properties <<'EOF'
database.host=mysql-service
database.port=3306
cache.enabled=true
EOF
```

**2. Create the ConfigMap from it:**
```bash
kubectl create configmap file-config --from-file=app.properties
```

**Explanation:**
- The **file name becomes the key** (`app.properties`).
- The **whole file content becomes the value**.
- To choose a custom key name: `--from-file=mykey=app.properties`
- To load a whole folder: `--from-file=./config-folder/`
- To load `KEY=VALUE` lines as separate keys: use `--from-env-file=app.env`

---

### Way C: From a YAML manifest (recommended for real projects)

**Why recommended:** it can be stored in Git, reviewed, and re-applied (Infrastructure as Code).

**Create `configmap.yaml`:**
```yaml
apiVersion: v1                 # ConfigMap belongs to the core API group "v1"
kind: ConfigMap                # The type of object
metadata:
  name: yaml-config            # Name of the ConfigMap
  namespace: demo              # Namespace it lives in
  labels:
    app: myapp                 # Optional labels for organizing
data:                          # All your config goes under "data"
  # Simple key-value pairs
  APP_ENV: "production"
  LOG_LEVEL: "info"
  APP_PORT: "8080"

  # A whole file as a value (note the | for multi-line text)
  app.properties: |
    database.host=mysql-service
    database.port=3306
    cache.enabled=true
```

**Apply it:**
```bash
kubectl apply -f configmap.yaml
```

**YAML explained line by line:**

| Field | Meaning |
|---|---|
| `apiVersion: v1` | API version for ConfigMap |
| `kind: ConfigMap` | Object type |
| `metadata.name` | Unique name inside the namespace |
| `data` | Key-value text data. **Values must be strings** (quote numbers/booleans: `"8080"`, `"true"`) |
| `\|` | Keeps line breaks (multi-line value) |
| `binaryData` | (Optional) for base64-encoded binary data |

**Tip:** generate YAML without creating anything (dry run):
```bash
kubectl create configmap demo-cm --from-literal=A=1 --dry-run=client -o yaml
```

---

## Step 3: View and inspect a ConfigMap

**List all ConfigMaps:**
```bash
kubectl get configmaps
# short form
kubectl get cm
```

**See details (human readable):**
```bash
kubectl describe configmap app-config
```

**See full YAML:**
```bash
kubectl get configmap app-config -o yaml
```

**Get one specific key:**
```bash
kubectl get configmap app-config -o jsonpath='{.data.LOG_LEVEL}'
```

**When to use which:**
- `get cm` -> quick list
- `describe` -> quick human-readable check
- `-o yaml` -> see exactly what is stored, or copy it to a file

---

## Step 4: Use ConfigMap as environment variables

### 4A: Inject ALL keys at once (`envFrom`)

**Create `pod-env-all.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-all-pod
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["sh", "-c", "env | sort && sleep 3600"]
      envFrom:
        - configMapRef:
            name: app-config        # every key in app-config becomes an env var
  restartPolicy: Never
```

**Apply and check:**
```bash
kubectl apply -f pod-env-all.yaml
kubectl logs env-all-pod | grep -E "APP_ENV|LOG_LEVEL|APP_PORT"
```
Expected:
```
APP_ENV=development
APP_PORT=8080
LOG_LEVEL=debug
```

---

### 4B: Inject only SPECIFIC keys (`valueFrom`)

Use this to pick keys and/or rename the variable.

**Create `pod-env-one.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-one-pod
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["sh", "-c", "echo LEVEL=$MY_LOG_LEVEL && sleep 3600"]
      env:
        - name: MY_LOG_LEVEL          # env var name inside the container
          valueFrom:
            configMapKeyRef:
              name: app-config        # which ConfigMap
              key: LOG_LEVEL          # which key
  restartPolicy: Never
```

```bash
kubectl apply -f pod-env-one.yaml
kubectl logs env-one-pod
```
Expected: `LEVEL=debug`

**Important:** env vars are read **only when the Pod starts**. If you change the ConfigMap later, the Pod does **not** see the new value until it is restarted.

---

## Step 5: Use ConfigMap as a mounted file (volume)

**Why:** Many apps read config from files (`nginx.conf`, `app.properties`). Mounting turns each ConfigMap key into a file.

**Create `pod-volume.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
    - name: demo
      image: busybox:1.36
      command: ["sh", "-c", "cat /etc/config/app.properties && sleep 3600"]
      volumeMounts:
        - name: config-volume         # must match the volume name below
          mountPath: /etc/config      # folder inside the container
  volumes:
    - name: config-volume
      configMap:
        name: file-config             # ConfigMap created in Step 2B
  restartPolicy: Never
```

**Apply and check:**
```bash
kubectl apply -f pod-volume.yaml
kubectl logs volume-pod
kubectl exec volume-pod -- ls /etc/config
kubectl exec volume-pod -- cat /etc/config/app.properties
```
Expected: the file `app.properties` exists and shows your 3 lines.

**Result:** each key becomes a file name, each value becomes the file content.

### Mount only one key (and rename it) with `items`
```yaml
  volumes:
    - name: config-volume
      configMap:
        name: file-config
        items:
          - key: app.properties
            path: settings.conf       # file will be /etc/config/settings.conf
```

### Mount a single file without hiding the rest of the folder (`subPath`)
```yaml
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config/settings.conf
          subPath: app.properties
```
**Warning:** files mounted with `subPath` do **not** auto-update when the ConfigMap changes.

---

## Step 6: Update a ConfigMap

**Edit directly:**
```bash
kubectl edit configmap app-config
```

**Or edit the YAML file and re-apply (best practice):**
```bash
kubectl apply -f configmap.yaml
```

**Or patch one key:**
```bash
kubectl patch configmap app-config --type merge -p '{"data":{"LOG_LEVEL":"warn"}}'
```

**What happens to running Pods?**

| How the Pod uses it | Auto-updates? |
|---|---|
| Environment variables | No. Restart the Pod/Deployment |
| Mounted volume | Yes, after about 1 minute (not for `subPath` mounts) |

**Restart a Deployment to pick up new env values:**
```bash
kubectl rollout restart deployment <deployment-name>
```

**Watch the volume update live:**
```bash
kubectl exec volume-pod -- cat /etc/config/app.properties
# change the ConfigMap, wait ~60s, run the same command again
```

---

## Step 7: Immutable ConfigMap

**When to use:** production configs that must never change by accident. Also reduces load on the API server.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: locked-config
data:
  APP_ENV: "production"
immutable: true        # cannot be edited after creation
```

```bash
kubectl apply -f locked-config.yaml
```

To change it, you must **delete and recreate** it (common practice: version names like `app-config-v2`).

---

## Step 8: Clean up

```bash
kubectl delete pod env-all-pod env-one-pod volume-pod
kubectl delete configmap app-config file-config yaml-config
# or remove everything at once:
kubectl delete namespace demo
```

Reset default namespace:
```bash
kubectl config set-context --current --namespace=default
```

---

## Full end-to-end example: Nginx + ConfigMap

**Goal:** Serve a custom web page from a ConfigMap and also pass an env var, all in one Deployment + Service.

### 1. Create the namespace
```bash
kubectl create namespace demo
```

### 2. Create `nginx-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: demo
data:
  # Env-style setting
  WELCOME_MESSAGE: "Hello from ConfigMap!"

  # A whole file: our custom web page
  index.html: |
    <!DOCTYPE html>
    <html>
      <head><title>ConfigMap Demo</title></head>
      <body>
        <h1>Hello from a Kubernetes ConfigMap!</h1>
        <p>This page is served from a ConfigMap, not baked into the image.</p>
      </body>
    </html>
```

### 3. Create `nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          env:
            - name: WELCOME_MESSAGE            # env var from ConfigMap
              valueFrom:
                configMapKeyRef:
                  name: nginx-config
                  key: WELCOME_MESSAGE
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html/index.html   # replace default page
              subPath: index.html
      volumes:
        - name: html-volume
          configMap:
            name: nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-svc
  namespace: demo
spec:
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### 4. Apply both files
```bash
kubectl apply -f nginx-configmap.yaml
kubectl apply -f nginx-deployment.yaml
```

### 5. Verify everything
```bash
kubectl get all -n demo
kubectl get cm -n demo
kubectl rollout status deployment/nginx-demo -n demo
```

### 6. Test the page
```bash
kubectl port-forward svc/nginx-demo-svc 8081:80 -n demo
```
Open **http://localhost:8081** in your browser. You should see "Hello from a Kubernetes ConfigMap!".

(Press `Ctrl+C` to stop port-forwarding.)

### 7. Check the env var inside a Pod
```bash
kubectl exec -n demo deploy/nginx-demo -- printenv WELCOME_MESSAGE
```
Expected: `Hello from ConfigMap!`

### 8. Change the config and see the result
```bash
kubectl edit configmap nginx-config -n demo       # change the <h1> text and/or WELCOME_MESSAGE
kubectl rollout restart deployment/nginx-demo -n demo
```
Because we used `subPath`, a restart is required to see the change. Refresh the browser to confirm.

### 9. Clean up
```bash
kubectl delete namespace demo
```

---

## Important commands cheat sheet

| Task | Command | When to use |
|---|---|---|
| Create from literals | `kubectl create cm NAME --from-literal=K=V` | A few simple keys |
| Create from file | `kubectl create cm NAME --from-file=FILE` | Existing config file |
| Create from env file | `kubectl create cm NAME --from-env-file=FILE.env` | `KEY=VALUE` file -> many keys |
| Create from YAML | `kubectl apply -f cm.yaml` | Real projects / Git |
| Generate YAML only | `kubectl create cm NAME --from-literal=K=V --dry-run=client -o yaml` | Build a manifest quickly |
| List | `kubectl get cm` | See all ConfigMaps |
| Describe | `kubectl describe cm NAME` | Quick readable check |
| View YAML | `kubectl get cm NAME -o yaml` | See exact stored data |
| Edit live | `kubectl edit cm NAME` | Quick change |
| Patch a key | `kubectl patch cm NAME --type merge -p '{"data":{"K":"V"}}'` | Script-friendly change |
| Delete | `kubectl delete cm NAME` | Remove it |
| See env in Pod | `kubectl exec POD -- env` | Verify env injection |
| See mounted files | `kubectl exec POD -- ls /path` | Verify volume mount |
| Restart Deployment | `kubectl rollout restart deploy NAME` | Pick up new env values |
| Use other namespace | `-n NAMESPACE` | Any command |

---

## Common errors and fixes

| Problem | Likely cause | Fix |
|---|---|---|
| Pod stuck in `CreateContainerConfigError` | Referenced ConfigMap or key doesn't exist | `kubectl describe pod POD`, then create the ConfigMap / fix the key name |
| `configmap "x" not found` | Wrong name or wrong namespace | Check `kubectl get cm -n NAMESPACE` |
| Pod mounts but file is missing | Key name in ConfigMap differs from expected | `kubectl describe cm NAME` and check keys |
| Changed ConfigMap but app shows old value | Used as env var or `subPath` | Restart: `kubectl rollout restart deploy NAME` |
| `cannot unmarshal number into Go struct field ... of type string` | Unquoted number/boolean in `data` | Quote values: `"8080"`, `"true"` |
| Cannot edit ConfigMap | It is `immutable: true` | Delete and recreate (use a new name/version) |
| Existing folder content disappeared after mounting | Volume mount hides the original folder | Use `subPath` to mount a single file |

---

## Quick revision

- **ConfigMap = non-secret settings as key-value pairs**, separate from the image.
- Create with **literal**, **file**, or **YAML** (YAML is best for real work).
- Use in Pods as **env vars** (`envFrom` / `configMapKeyRef`) or **files** (`volumes` + `volumeMounts`).
- **Env vars don't auto-update**; **volume files do** (about 1 min, except `subPath`).
- **Passwords go in Secrets**, not ConfigMaps.
- Limit: **1 MiB**.
- Use `immutable: true` for safe, stable production configs.

