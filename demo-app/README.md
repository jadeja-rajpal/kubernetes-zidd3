# Helm Basics: demo-app

A small chart to learn Helm. It deploys two microservices (**backend** and **frontend**), each with a Deployment, Service, ConfigMap and Secret. Every value comes from `values.yaml`, so the same chart can be deployed to **dev** and **prod** (different namespaces) just by changing the values file.

---

## 1. What is Helm?

Helm is the **package manager for Kubernetes**, like `apt` for Ubuntu or `npm` for Node.

Without Helm you copy-paste the same YAML for dev, staging and prod and edit it by hand. With Helm you write the YAML **once** as a template and keep the changing parts in a values file.

| Term | Meaning |
|---|---|
| **Chart** | A folder with templates and default values (this `demo-app/` folder) |
| **Release** | One installed copy of a chart in a cluster (e.g. `demo-dev`) |
| **Values** | The settings that fill in the templates (`values.yaml`) |
| **Template** | A Kubernetes YAML file with `{{ ... }}` placeholders |
| **Revision** | Each install or upgrade of a release creates a new revision. This is what rollback uses. |

---

## 2. Folder structure

```
demo-app/
├── Chart.yaml            # chart name and version
├── values.yaml           # DEV values (default)
├── values-prod.yaml      # PROD values (override file)
├── README.md
├── COMMANDS.md
└── templates/
    ├── backend-configmap.yaml
    ├── backend-deployment.yaml
    ├── backend-secret.yaml
    ├── backend-service.yaml
    ├── frontend-configmap.yaml
    ├── frontend-deployment.yaml
    ├── frontend-secret.yaml
    └── frontend-service.yaml
```

---

## 3. How values flow into templates

`values.yaml`:

```yaml
backend:
  replicas: 2
```

`templates/backend-deployment.yaml`:

```yaml
spec:
  replicas: {{ .Values.backend.replicas }}
```

Result after rendering:

```yaml
spec:
  replicas: 2
```

Only three bits of syntax are used in this chart:

| Syntax | Meaning |
|---|---|
| `{{ .Values.backend.replicas }}` | Read this value from the values file |
| `\| quote` | Wrap the value in double quotes (needed for strings in ConfigMaps and Secrets) |
| `{{ ... }}` | Anything inside the braces is Helm. Everything outside is plain Kubernetes YAML. |

---

## 4. Dev vs Prod values

`values.yaml` is the default. `values-prod.yaml` has the same keys with prod values. Helm merges the file passed with `-f` on top of `values.yaml`, and the `-f` file wins.

| Key | Dev (`values.yaml`) | Prod (`values-prod.yaml`) |
|---|---|---|
| `backend.replicas` | 2 | 4 |
| `backend.logLevel` | info | warn |
| `backend.appEnv` | dev | prod |
| `backend.dbPassword` | changeme | prod-secret-123 |
| `frontend.replicas` | 1 | 3 |
| `frontend.apiKey` | dummy-key | prod-api-key |

Everything else (image, tag, ports, service type) is the same in both.

---

## 5. Why different namespaces?

The templates use fixed names (`backend`, `frontend`). Two releases in the **same** namespace would try to create the same Deployment and Service names and clash. In **different** namespaces there is no problem:

```
namespace: dev      namespace: prod
 ├── backend (2)     ├── backend (4)
 └── frontend (1)    └── frontend (3)
```

This is why we install dev into `dev` and prod into `prod`.

---

## 6. Step-by-step: deploy to two namespaces

Run these from inside the `demo-app/` folder (the chart path is `.`).

### Step 1: Check the chart

```bash
helm lint .
```

Expected: `1 chart(s) linted, 0 chart(s) failed`.

### Step 2: Preview the YAML (no cluster needed)

```bash
helm template demo-dev . 
helm template demo-prod . -f values-prod.yaml
```

Compare the two outputs. Look for `replicas:`, `LOG_LEVEL` and `APP_ENV`.

### Step 3: Dry run against the cluster

```bash
helm install demo-dev . -n dev --create-namespace --dry-run --debug
```

### Step 4: Install DEV into the `dev` namespace

```bash
helm install demo-dev . -n dev --create-namespace
```

### Step 5: Install PROD into the `prod` namespace

```bash
helm install demo-prod . -n prod --create-namespace -f values-prod.yaml
```

`-n` picks the namespace. `--create-namespace` creates it if it does not exist.

### Step 6: Verify both

```bash
helm list -A
kubectl get deploy,svc,cm,secret -n dev
kubectl get deploy,svc,cm,secret -n prod
```

Expected replicas:

| Namespace | backend | frontend |
|---|---|---|
| dev | 2 | 1 |
| prod | 4 | 3 |

Check the config values landed correctly:

```bash
kubectl get cm backend-config -n dev  -o yaml
kubectl get cm backend-config -n prod -o yaml
```

Check a secret (Kubernetes stores it base64-encoded):

```bash
kubectl get secret backend-secret -n prod -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Check environment variables inside the frontend pod:

```bash
kubectl exec -n prod deploy/frontend -- env | grep -E "BACKEND_URL|API_KEY"
```

### Step 7: Test frontend to backend inside the cluster

```bash
kubectl exec -n dev deploy/frontend -- curl -s http://backend
```

Expected output: `hello from backend`. The short name `backend` resolves because both services are in the same namespace.

---

## 7. Upgrade, history and rollback

```bash
# Change a value from the command line
helm upgrade demo-dev . -n dev --set backend.replicas=3

# Or change values.yaml and upgrade
helm upgrade demo-dev . -n dev

# See revisions
helm history demo-dev -n dev

# Go back to revision 1
helm rollback demo-dev 1 -n dev
```

Rollback is the moment Helm clicks for most people, so try it live.

Precedence, from lowest to highest: `values.yaml`, then `-f file.yaml`, then `--set key=value`.

---

## 8. Inspecting a release

```bash
helm list -n dev                  # releases in a namespace
helm list -A                      # releases in all namespaces
helm status demo-dev -n dev       # status and notes
helm get values demo-dev -n dev   # values used by this release
helm get manifest demo-dev -n dev # the final YAML that was applied
```

---

## 9. Cleanup

```bash
helm uninstall demo-dev  -n dev
helm uninstall demo-prod -n prod

# optional: remove the namespaces
kubectl delete ns dev prod
```

---

## 10. Cheat sheet

| Goal | Command |
|---|---|
| Check chart syntax | `helm lint .` |
| Render YAML locally | `helm template <release> .` |
| Install | `helm install <release> . -n <ns> --create-namespace` |
| Install with values file | `helm install <release> . -n <ns> -f values-prod.yaml` |
| Upgrade | `helm upgrade <release> . -n <ns>` |
| Install or upgrade in one command | `helm upgrade --install <release> . -n <ns> --create-namespace` |
| List releases | `helm list -A` |
| History | `helm history <release> -n <ns>` |
| Rollback | `helm rollback <release> <revision> -n <ns>` |
| Uninstall | `helm uninstall <release> -n <ns>` |
| Show values used | `helm get values <release> -n <ns>` |
| Show final YAML | `helm get manifest <release> -n <ns>` |

---

## 11. Common errors

| Error | Cause and fix |
|---|---|
| `Error: INSTALLATION FAILED: cannot re-use a name that is still in use` | A release with this name already exists in the namespace. Use `helm upgrade` or pick a new name. |
| `namespaces "prod" not found` | Add `--create-namespace`. |
| `Error: path "./demo-app" not found` | You are already inside `demo-app/`. Use `.` as the chart path. |
| `nil pointer evaluating interface {}.xyz` | A key in the template is missing from the values file. Check spelling and indentation. |
| `YAML parse error` | Wrong indentation in a template or values file. Run `helm template .` to see where. |
| Pods stuck in `ImagePullBackOff` | Wrong `image` or `tag` in values. |

---

## 12. Exercises for students

1. Change `frontend.replicas` to 5 in `values.yaml`, run `helm template .` and find the change.
2. Add a new ConfigMap key `TIMEOUT` (add it in `values.yaml` and in `backend-configmap.yaml`), then upgrade.
3. Upgrade prod with `--set backend.logLevel=debug`. Run `helm get values demo-prod -n prod` and see what changed.
4. Break the chart on purpose (delete a `}`), then run `helm lint .`.
5. Roll back to the previous revision and confirm the replicas went back.

---

## 13. Important note on secrets

The passwords here are in plain text in the values files **for learning only**. In real projects:

- Never commit real secrets to Git.
- Use External Secrets Operator, Sealed Secrets, or inject from CI with `--set`.
- Remember that Kubernetes Secrets are only base64-encoded, not encrypted.

---

## 14. What to learn next

1. `_helpers.tpl` for reusable names and labels
2. `{{ .Release.Name }}` so the same chart can be installed twice in one namespace
3. `{{ if }}` and `{{ range }}` for optional resources and loops
4. `helm package` and pushing a chart to a registry
5. `helm dependency` for sub-charts
6. Helmfile or ArgoCD to manage many releases with GitOps
