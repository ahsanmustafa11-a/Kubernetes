# Helm for Kubernetes — Practical Learning Guide

A beginner-friendly, hands-on guide to Helm, covering chart fundamentals, templates, values, releases, repositories, upgrades, rollbacks, storage, security, CI/CD integration, troubleshooting, and practical exercises. The examples use an NGINX practice chart and progress toward converting the DevBoard application into a Helm chart.

---

## 1. What is Helm?

Helm is a package manager for Kubernetes. It helps you install, configure, upgrade, roll back, and manage Kubernetes applications.

Think of installing software on Windows:

- You download or install a software package.
- You configure its settings.
- You install it.
- You update it when a new version is available.
- You uninstall it when you no longer need it.

Helm provides a similar workflow for Kubernetes applications.

## Without Helm vs. With Helm

Without Helm

Manage individual YAML files manually.

Edit image versions, replicas, ports and configuration in multiple places.

Keep track of upgrades and rollback steps yourself.

With Helm

Group Kubernetes resources into a reusable package called a chart.

Configure values in one place.

Use commands to install, upgrade and roll back releases.

For example, suppose your DevBoard application has:

- Frontend Deployment and Service
- Backend Deployment and Service
- PostgreSQL StatefulSet and headless Service
- ConfigMap and Secret
- Persistent storage

Without Helm, you manage several YAML files separately. With Helm, you can organize these resources into one chart and install the application with a single command.

```
helm install devboard ./devboard-chart \
  --namespace devboard-ns \
  --create-namespace
```

This command installs a Helm release named `devboard` using the chart in `./devboard-chart`.

Important: Helm does not replace Kubernetes. Kubernetes still runs your containers, schedules Pods, manages Services, and maintains storage. Helm helps you package and manage the Kubernetes resource definitions.

## 2. What problems does Helm solve?

| Problem                             | How Helm helps                                              |
| ----------------------------------- | ----------------------------------------------------------- |
| Too many YAML files                 | Organizes them into a chart                                 |
| Repeated configuration              | Uses reusable templates and values                          |
| Different environments              | Uses different values files for dev, staging and production |
| Application updates                 | Uses `helm upgrade`                                         |
| Failed application upgrade          | Can attempt a rollback to an earlier release revision       |
| Installing third-party applications | Installs published charts, such as monitoring tools         |
| Reusing application definitions     | Packages charts for sharing and reuse                       |

Helm is commonly used by DevOps engineers, platform engineers, and Kubernetes administrators to deploy applications consistently.

## 3. Important Helm terminology

Learn these terms first because they appear in almost every Helm command and interview question.

1\. Chart

A package containing templates, default settings, metadata, and possibly dependencies needed to create Kubernetes resources.

Example: `devboard-chart/`

2\. Values

Settings that customize a chart without requiring you to rewrite its templates.

Example: `replicaCount: 3`

3\. Template

A file that contains Kubernetes YAML plus placeholders that Helm fills in using values.

Example: `templates/deployment.yaml`

4\. Release

An installed instance of a chart in a Kubernetes cluster. You can install the same chart multiple times with different release names and settings.

Example: `devboard-dev`, `devboard-prod`

5\. Revision

A numbered version of a release's recorded Helm history, usually incremented by successful upgrades or rollbacks.

Example: revision 1 → revision 2 → revision 3

### How these concepts work together

Chart

Templates + default values + metadata

Helm renders the templates

Combines templates with selected values

Kubernetes manifests

Deployment, Service, ConfigMap, etc.

Release

A named installation managed by Helm

One distinction matters: a chart is the package; a release is an installation of that package. A single chart can have many releases.

## 4. Install Helm on your computer

Helm is a command-line tool. It normally runs on your computer or inside a CI/CD runner and communicates with your Kubernetes API server using your existing Kubernetes access configuration. You do not need to deploy a separate Helm server into your cluster for standard modern Helm usage.

Since you're learning with a local `kind` cluster, you can install Helm on the same Linux machine where you run `kubectl`.

### Option A — Linux

On a Linux system that has `curl` installed, you can use the official Helm installation script:

```
curl -fsSL -o get_helm.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh
./get_helm.sh
```

This script targets Helm 3. For the latest release and installation options, check the [official Helm installation guide](https://helm.sh/docs/intro/install/?utm_source=chatgpt.com).

### Verify the installation

```
helm version
helm help
kubectl cluster-info
kubectl get nodes
```

If Helm is installed but cannot access Kubernetes, verify your `kubectl` context:

```
kubectl config current-context
kubectl config get-contexts
```

Helm generally uses the same Kubernetes context and credentials as `kubectl`. Check the active context before making changes to any cluster.

## 5. The most important Helm command categories

You don't need to memorize every command on day one. Learn these groups in order.

| Category       | Main commands                                           | Purpose                         |
| -------------- | ------------------------------------------------------- | ------------------------------- |
| Information    | `helm version`, `helm env`                              | Check Helm and its environment  |
| Chart creation | `helm create`                                           | Generate a starter chart        |
| Repository     | `helm repo add`, `helm repo update`, `helm search repo` | Find published charts           |
| Installation   | `helm install`, `helm upgrade --install`                | Deploy an application           |
| Inspection     | `helm list`, `helm status`, `helm get values`           | Understand a release            |
| Validation     | `helm lint`, `helm template`                            | Check a chart before deployment |
| History        | `helm history`, `helm rollback`                         | Review revisions and recover    |
| Removal        | `helm uninstall`                                        | Remove a release                |

## 6. Chart structure — understand the files

Create your first chart:

```
helm create mychart
```

Helm generates a directory with files similar to these:

mychart/

`Chart.yaml`

Chart information

`values.yaml`

Default settings

`templates/`

`deployment.yaml`

`service.yaml`

`serviceaccount.yaml`

`ingress.yaml`

`hpa.yaml`

`_helpers.tpl`

`NOTES.txt`

`charts/`

Dependency charts

`.helmignore`

Files to exclude from packaging

The exact files can vary by Helm version and chart template.

### What does each file do?

| File or directory     | Plain-English meaning                                           |
| --------------------- | --------------------------------------------------------------- |
| `Chart.yaml`          | Defines the chart name, version and description                 |
| `values.yaml`         | Contains default configuration values                           |
| `templates/`          | Contains Kubernetes YAML templates                              |
| `_helpers.tpl`        | Stores reusable template snippets                               |
| `charts/`             | Contains packaged dependencies                                  |
| `templates/NOTES.txt` | Displays useful post-installation instructions                  |
| `.helmignore`         | Prevents selected files from being included in a packaged chart |

A chart can contain templates for any Kubernetes resources you need, including StatefulSets, PVCs, RBAC resources, ConfigMaps, and Services.

## 7. Your first practical project: Create a Helm chart for NGINX

We'll start with a simple NGINX Deployment and Service. This is easier to understand than converting a complete three-tier application immediately.

### Step 1: Create the chart

```
helm create nginx-lab
cd nginx-lab
```

First, inspect the generated files:

```
find . -maxdepth 2 -type f | sort
```

If you are using Windows PowerShell, use:

```
Get-ChildItem -Recurse -File
```

### Step 2: Understand `Chart.yaml`

Open the file:

```
cat Chart.yaml
```

A simplified example is:

```
apiVersion: v2
name: nginx-lab
description: My first Kubernetes Helm chart
type: application
version: 0.1.0
appVersion: "1.28.0"
```

Meaning:

- `apiVersion: v2` — the chart format used by Helm 3 and also supported by Helm 4.
- `name` — the name of the chart.
- `description` — what the chart does.
- `type: application` — this chart deploys an application.
- `version` — the version of the chart package.
- `appVersion` — the application version the chart is intended to deploy; this is informational unless your templates use it.

Important difference: `version` and `appVersion` are not the same thing.

For example, you could update the chart from `0.1.0` to `0.2.0` because you changed the Kubernetes configuration while keeping the NGINX image version unchanged.

### Step 3: Understand `values.yaml`

Open the generated values:

```
cat values.yaml
```

It contains many settings. For our first example, simplify it to:

```
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.28.0"

service:
  type: ClusterIP
  port: 80
```

These values tell Helm:

- Create two replicas.
- Use the `nginx` image repository.
- Use the specified image tag.
- Reuse a locally cached image if available; otherwise, pull it.
- Expose the application through a ClusterIP Service on port 80.

Use a valid image tag available in your chosen registry when running the lab. You can use another supported NGINX tag if necessary.

### Step 4: Understand a Helm template

In your chart's `templates/deployment.yaml`, you can write a Kubernetes Deployment with placeholders.

For a simple learning example, replace the file contents with:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-nginx
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-nginx
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
```

Read the placeholders carefully:

| Template expression        | Meaning                                       |
| -------------------------- | --------------------------------------------- |
| `.Release.Name`            | The name you assign when installing the chart |
| `.Values.replicaCount`     | The replica count from `values.yaml`          |
| `.Values.image.repository` | The image repository                          |
| `.Values.image.tag`        | The image tag                                 |
| `.Values.image.pullPolicy` | The image pull policy                         |

For example, if you install with the release name `web-test`, Helm renders `.Release.Name` as `web-test`.

The Deployment name becomes `web-test-nginx`.

This is how a single chart can produce different resource names for different releases.

### Step 5: Create a Service template

Replace `templates/service.yaml` with:

```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-nginx
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}-nginx
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: http
```

Notice that the Service selector matches the Pod label in the Deployment template.

`targetPort: http` refers to the named container port in the Pod template. This is the same Kubernetes Service concept you've already learned; Helm only generates the YAML.

### Step 6: Validate the chart

Run:

```
helm lint .
```

This checks the chart for common problems. It does not guarantee that the application will run successfully.

Next, render the templates without installing them:

```
helm template web-test .
```

Look at the output. You should see ordinary Kubernetes YAML with the Helm placeholders replaced.

For example, the Deployment will contain:

```
metadata:
  name: web-test-nginx
spec:
  replicas: 2
```

This is one of the most important Helm skills: understand the generated Kubernetes YAML before deploying it.

### Step 7: Install the chart

```
helm install web-test . \
  --namespace helm-lab \
  --create-namespace
```

Check the results:

```
helm list -n helm-lab
helm status web-test -n helm-lab
kubectl get deploy,svc,pods -n helm-lab
```

You should see the release and its Kubernetes resources. Wait for the Pods to become ready before testing the application.

### Step 8: Access NGINX locally

Use port-forwarding:

```
kubectl port-forward \
  -n helm-lab \
  svc/web-test-nginx 8085:80
```

Keep the command running, then open `http://localhost:8085` in your browser.

The port mapping is:

Your computer

## localhost:8085

Kubernetes Service

## Port 80

NGINX Pod

## Container port 80

This port-forward is for local testing; it does not change your Service into a NodePort.

### Step 9: Clean up the lab

When you're finished:

```
helm uninstall web-test -n helm-lab
kubectl delete namespace helm-lab
```

The second command removes the namespace and any remaining namespaced resources in it. Only do this for the dedicated practice namespace, not a namespace containing applications you need to keep.

## 8. Helm values — the most important concept

You will use values almost every time you work with Helm.

Imagine you have one application, but you want different configurations for different environments.

| Setting           | Development     | Production         |
| ----------------- | --------------- | ------------------ |
| Frontend replicas | 1               | 5                  |
| Backend replicas  | 1               | 4                  |
| Log level         | debug           | info               |
| Resource requests | Low             | Sized for workload |
| Ingress hostname  | dev.example.com | app.example.com    |

You should not need to maintain two completely separate copies of your Deployment YAML. Instead, use one chart and change the values.

### 8.1 Override a value using `--set`

Your `values.yaml` contains:

```
replicaCount: 2
```

Install the chart with three replicas instead:

```
helm install web-test ./nginx-lab \
  --set replicaCount=3
```

Helm uses three replicas for this installation, overriding the default.

To change an existing release:

```
helm upgrade web-test ./nginx-lab \
  --set replicaCount=4
```

### 8.2 Override values using a separate file

Create `values-dev.yaml`:

```
replicaCount: 1

image:
  repository: nginx
  tag: "1.28.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

Create `values-prod.yaml`:

```
replicaCount: 4

image:
  repository: nginx
  tag: "1.28.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

Deploy using the development configuration:

```
helm upgrade --install web-dev ./nginx-lab \
  -n development \
  --create-namespace \
  -f values-dev.yaml
```

Deploy using the production configuration:

```
helm upgrade --install web-prod ./nginx-lab \
  -n production \
  --create-namespace \
  -f values-prod.yaml
```

Both releases use the same chart, but their configurations differ.

### 8.3 Which value wins?

Suppose your chart defaults to two replicas, but you provide a file with three replicas and a command-line override of four.

```
helm upgrade --install web-test ./nginx-lab \
  -f values-dev.yaml \
  --set replicaCount=4
```

For the same value, the precedence is generally:

1. Chart's `values.yaml` defaults
2. User-supplied values files, in the order supplied
3. Command-line `--set` overrides

The last applicable override wins. If you provide multiple `-f` files, later files override earlier files for overlapping values.

Useful command: See the values configured on an installed release.

```
helm get values web-test -n helm-lab
```

Show computed values, including chart defaults:

```
helm get values web-test -n helm-lab --all
```

Be careful when sharing output because values can contain sensitive information.

## 9. Helm templating — from basic to advanced

Helm templates use Go template syntax with functions and other template features. The expressions inside `{{ }}` are processed before Kubernetes receives the manifests.

### 9.1 Variables and expressions

A regular Kubernetes manifest contains a fixed value:

```
replicas: 3
```

A Helm template can contain:

```
replicas: {{ .Values.replicaCount }}
```

And `values.yaml` supplies:

```
replicaCount: 3
```

After rendering, Kubernetes receives:

```
replicas: 3
```

Helm does not send the curly braces to Kubernetes.

### 9.2 Conditional templates: `if`

Sometimes you want a resource to exist only when a feature is enabled.

In `values.yaml`:

```
ingress:
  enabled: false
```

In a template:

```
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}-ingress
spec:
  ingressClassName: nginx
{{- end }}
```

When `ingress.enabled` is `false`, Helm skips that resource.

When it is `true`, Helm includes it in the rendered output.

This is useful for optional Ingress, HPA, NetworkPolicy, and other resources. The Ingress example is only a partial manifest; a working Ingress also needs appropriate rules, a backend, and a compatible controller.

### 9.3 Loops: `range`

Suppose you want to create multiple environment variables.

In `values.yaml`:

```
extraEnv:
  - name: APP_MODE
    value: production
  - name: LOG_LEVEL
    value: info
```

In a Deployment template:

```
env:
  {{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
  {{- end }}
```

Helm repeats the YAML block for each item in `extraEnv`.

The `quote` function adds quotes to the rendered string. This helps preserve string values, especially when they resemble numbers or special YAML values.

### 9.4 Reusable templates: `_helpers.tpl`

When multiple resources need consistent names and labels, avoid copying the same logic everywhere.

In `templates/_helpers.tpl`:

```
{{- define "nginx-lab.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
```

Then in a Deployment:

```
metadata:
  name: {{ include "nginx-lab.fullname" . }}
```

And in a Service:

```
metadata:
  name: {{ include "nginx-lab.fullname" . }}
```

Now both resources use the same naming logic.

- `define` creates a reusable template.
- `include` renders that template and returns its output.
- `.` passes the current template context.
- `printf` formats a string.
- `trunc 63` limits the name length.
- `trimSuffix "-"` removes a trailing hyphen.

### 9.5 Common template functions

| Function   | What it does                                            | Example                                            |
| ---------- | ------------------------------------------------------- | -------------------------------------------------- |
| `quote`    | Quotes a value                                          | `{{ .Values.name \\| quote }}`                     |
| `default`  | Supplies a fallback when a value is empty               | `{{ .Values.name \\| default "nginx" }}`           |
| `required` | Stops rendering if a required value is missing or empty | `{{ required "Set image.tag" .Values.image.tag }}` |
| `include`  | Renders a named helper template                         | `{{ include "nginx-lab.fullname" . }}`             |
| `toYaml`   | Converts an object or map to YAML                       | `{{ toYaml .Values.resources }}`                   |
| `nindent`  | Indents text and adds a newline                         | `{{ toYaml .Values.resources \\| nindent 12 }}`    |
| `range`    | Iterates over a list or map                             | `{{ range .Values.extraEnv }}`                     |
| `if`       | Conditionally includes template content                 | `{{ if .Values.ingress.enabled }}`                 |

Indentation matters. YAML is sensitive to whitespace, so `nindent` and related functions are especially useful when generating nested configuration.

For example:

```
resources:
  {{- toYaml .Values.resources | nindent 2 }}
```

The exact indentation depends on the surrounding YAML. Always verify the rendered result with `helm template`.

## 10. Helm repositories — install applications created by others

A chart repository is a place where charts are published and made available for installation. A repository is different from a chart and a release.

- Repository: where you find packages.
- Chart: the package you select.
- Release: the installation you create.

You can find charts from different maintainers on [Artifact Hub](https://artifacthub.io/?utm_source=chatgpt.com). Check the publisher, chart documentation, supported versions, and security posture before installing third-party software.

### 10.1 Add a repository

For example, add the Bitnami chart repository:

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Repository availability and chart publishing arrangements can change; if these commands fail, consult the publisher's current installation instructions.

### 10.2 Search for a chart

```
helm search repo nginx
helm search repo postgresql
helm search repo --versions nginx
```

### 10.3 Inspect chart information

```
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm show readme bitnami/nginx
```

These commands help you understand what a chart supports before installing it.

### 10.4 Download a chart without installing it

```
helm pull bitnami/nginx
```

Extract the downloaded chart archive:

```
tar -xzf nginx-*.tgz
```

You can inspect its templates and default values locally.

### 10.5 Install a third-party chart

The exact settings depend on the chart's current values schema. First inspect the defaults:

```
helm show values bitnami/nginx
```

Then install with settings that match that chart's documented configuration.

```
helm install my-nginx bitnami/nginx \
  --namespace nginx-lab \
  --create-namespace
```

For real environments, don't blindly install a chart with its defaults. Review what resources it creates, what images it uses, what permissions it requests, how it stores data, and how it exposes the application.

## 11. Upgrade, rollback, and release history

This is one of Helm's biggest advantages in production operations.

### 11.1 Install or upgrade with one command

```
helm upgrade --install web-test ./nginx-lab \
  -n helm-lab
```

The command means:

- If the release doesn't exist, install it.
- If the release already exists, upgrade it.

This is very useful in CI/CD pipelines because the same deployment command can be used repeatedly.

### 11.2 Upgrade an image version

Suppose your `values.yaml` contains:

```
image:
  repository: nginx
  tag: "1.28.0"
```

Change the tag to another valid version, then run:

```
helm upgrade web-test ./nginx-lab \
  -n helm-lab \
  -f values.yaml
```

Helm renders the updated manifests and applies the resulting changes to Kubernetes.

### 11.3 Check release history

```
helm history web-test -n helm-lab
```

Example of the kind of history you might see:

| Revision | Change                | Status     |
| -------- | --------------------- | ---------- |
| 1        | Initial installation  | superseded |
| 2        | Updated replica count | superseded |
| 3        | Changed image         | deployed   |

The table is illustrative, not the history of your actual cluster.

### 11.4 Roll back to an earlier revision

```
helm rollback web-test 1 -n helm-lab
```

This asks Helm to restore the configuration associated with revision 1.

A rollback normally creates a new revision; it doesn't erase the later history.

### 11.5 What rollback does not guarantee

Rollback is not a universal undo button. It may restore Kubernetes resource configuration, but it cannot automatically reverse every side effect.

Examples:

- A database migration might have changed the schema.
- A persistent volume may contain newly written data.
- A change to an external service may not be reversible by Helm.
- A failed application might remain unhealthy because the underlying issue persists.

For a production application, test rollback procedures and database recovery separately.

## 12. Namespaces, releases, and multiple environments

A namespace separates namespaced Kubernetes resources logically. A Helm release is a named installation tracked by Helm.

You can deploy one chart multiple times:

```
helm upgrade --install devboard-dev ./devboard-chart \
  -n dev \
  --create-namespace

helm upgrade --install devboard-staging ./devboard-chart \
  -n staging \
  --create-namespace

helm upgrade --install devboard-prod ./devboard-chart \
  -n production \
  --create-namespace
```

These releases can use different values files:

```
helm upgrade --install devboard-prod ./devboard-chart \
  -n production \
  -f values-prod.yaml
```

Remember:

- A release name must be unique among releases in the same namespace.
- Namespaced resources from separate releases must not accidentally collide.
- Cluster-scoped resources such as certain RBAC resources, CRDs, and cluster-wide controllers require extra care; namespaces do not isolate them in the same way.

## 13. Helm dependencies and subcharts

A dependency is another chart that your chart uses.

For example, your application might need a Redis cache or a database chart in addition to its own Deployment.

A simplified `Chart.yaml` example:

```
apiVersion: v2
name: my-application
description: Application with an optional Redis dependency
type: application
version: 0.1.0

dependencies:
  - name: redis
    version: "20.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

This is an illustrative dependency declaration; confirm that the selected chart version exists and supports the expected configuration.

In `values.yaml`:

```
redis:
  enabled: true
```

The `condition` allows you to enable or disable the dependency.

Update dependencies:

```
helm dependency update ./my-application
```

Inspect the downloaded dependencies:

```
ls -l ./my-application/charts
```

A parent chart can configure subcharts through nested values, but the exact keys depend on the dependency chart.

Production note: Packaging a database as a chart doesn't automatically make it highly available or safe. Persistence, backups, recovery, upgrades, and security still require deliberate design.

## 14. Helm with ConfigMaps and Secrets

This is particularly important when converting your existing Kubernetes applications into Helm charts.

### ConfigMap example

In `values.yaml`:

```
app:
  environment: production
  logLevel: info
```

In `templates/configmap.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  APP_ENVIRONMENT: {{ .Values.app.environment | quote }}
  LOG_LEVEL: {{ .Values.app.logLevel | quote }}
```

Now you can change application configuration through values without editing the template.

### Secret example

Kubernetes Secrets store sensitive values, but they are not automatically safe from unauthorized access. Base64 encoding is not encryption.

You can create a Secret template that accepts an externally supplied value, but avoid putting actual passwords or tokens in a committed `values.yaml` file.

For real applications, prefer an approved secret-management solution, such as an external secrets operator, a cloud secrets manager, or a properly secured secret injection mechanism.

For your DevBoard PostgreSQL deployment, keep the username and database configuration separate from the password. Helm is a packaging tool, not a secret vault.

## 15. Helm and persistent storage

Helm can create storage resources such as:

- PersistentVolumeClaims (PVCs)
- StatefulSets
- Headless Services
- StorageClasses
- ConfigMaps used for database initialization

However, you need to be especially careful when upgrading or uninstalling charts that manage databases.

For example, a PostgreSQL StatefulSet might use:

```
volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```

This belongs under the StatefulSet's `spec`.

Helm can manage the StatefulSet definition, but it doesn't automatically migrate existing PVC data into a newly created volume. If you change resource names, storage templates, or chart structure, existing resources might not be reused as you expect.

Before upgrading a database chart:

1. Back up the database.
2. Identify the existing PVC and its data.
3. Check whether the upgrade changes StatefulSet or PVC names.
4. Test the upgrade and recovery process in a separate environment.
5. Verify data after the operation.

Uninstalling a release also does not guarantee that all persistent data will be deleted or retained in every situation. Check the actual resources, storage policies, and chart behavior before performing destructive operations.

## 16. Advanced Helm topics

Once you can create a chart, deploy it, and override its values, learn these topics in order.

A. RBAC and security

Package ServiceAccounts, Roles, and RoleBindings. Understand why applications should receive only the Kubernetes permissions they require.

B. Hooks

Hooks run chart-related actions at lifecycle events, such as before or after an installation or upgrade. They can support database migrations and operational jobs, but need careful failure handling.

C. Chart testing

Use `helm lint`, `helm template`, and appropriate integration tests to catch errors before deployment.

D. Chart versioning

Version charts independently from application images. Keep a clear record of chart changes and supported application versions.

E. CI/CD integration

Validate charts in pull requests, render manifests, and deploy approved chart versions to development, staging, and production.

F. OCI registries and chart distribution

Package charts and publish them to a supported chart repository or OCI-compatible registry so teams can version and distribute them.

### 16.1 Chart packaging

Package your chart:

```
helm package ./nginx-lab
```

This creates a versioned archive such as `nginx-lab-0.1.0.tgz`, assuming the chart version is `0.1.0`.

Inspect it:

```
helm show chart ./nginx-lab-0.1.0.tgz
```

### 16.2 Validate a chart before deploying

Use these commands together:

```
helm lint ./nginx-lab

helm template web-test ./nginx-lab \
  -n helm-lab

helm upgrade --install web-test ./nginx-lab \
  -n helm-lab \
  --dry-run
```

The final command previews an installation or upgrade without applying the normal release changes. Dry-run behavior can vary by operation and Helm version, and some template or API checks may still require cluster access. Never assume a dry run proves that the live application will work.

### 16.3 Use a Helm hook

A simplified hook annotation looks like this:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-migration
  annotations:
    helm.sh/hook: pre-upgrade
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migration
          image: busybox:1.36
          command: ["sh", "-c", "echo Run migration here"]
```

This demonstrates hook syntax, not a production database migration. A real migration job needs the correct application image, database connection, credentials, retry behavior, timeout, and a safe strategy for partial failure. Also consider hook deletion policies and whether rerunning the operation is safe.

### 16.4 Helm tests

A chart can include test resources, commonly in `templates/`, annotated with:

```
helm.sh/hook: test
```

A test might verify that an application responds to an HTTP request.

Run the release's defined tests with:

```
helm test web-test -n helm-lab
```

A successful Helm test is useful evidence, but it does not replace comprehensive integration, security, and performance testing.

## 17. Helm in Jenkins or GitHub Actions

Since you already work with CI/CD pipelines, Helm is a natural next step.

A basic deployment stage could look like this:

```
stage('Validate Helm Chart') {
    steps {
        sh 'helm lint ./charts/devboard'
        sh 'helm template devboard ./charts/devboard -f values-dev.yaml'
    }
}

stage('Deploy to Kubernetes') {
    steps {
        sh '''
          helm upgrade --install devboard ./charts/devboard \
            --namespace devboard-ns \
            --create-namespace \
            --values values-dev.yaml \
            --wait \
            --timeout 5m
        '''
    }
}
```

This example assumes Helm, `kubectl`, and the required Kubernetes credentials are already configured on the Jenkins agent.

`--wait` asks Helm to wait for supported resources to reach their ready state, up to the specified timeout. It does not guarantee that every application-level dependency or business workflow is healthy.

A real pipeline should also:

- Validate and review the rendered manifests.
- Pin chart and image versions.
- Use least-privilege Kubernetes credentials.
- Keep secrets out of logs and source control.
- Deploy to development before promoting to production.
- Verify application health after deployment.
- Define a tested recovery procedure.

## 18. Troubleshooting common Helm errors

| Error or symptom                            | What to check                                                            |
| ------------------------------------------- | ------------------------------------------------------------------------ |
| `helm: command not found`                   | Installation and PATH                                                    |
| Kubernetes connection refused               | Current context, credentials, and API server                             |
| `cannot re-use a name that is still in use` | Existing release in the target namespace                                 |
| `values don't meet the specifications`      | Correct values keys and expected data types                              |
| YAML parse error                            | Indentation and rendered output                                          |
| Resource already exists                     | Existing Kubernetes resource and ownership conflicts                     |
| Upgrade times out                           | Pod events, readiness probes, image pulls, storage, and application logs |
| Release is pending                          | Release status, previous operation, and cluster state                    |
| PVC remains Pending                         | StorageClass, available storage, access modes, and provisioning          |
| Rollback doesn't fix application            | Database changes, external dependencies, and application health          |

Useful diagnostic commands:

```
helm list -A
helm status devboard -n devboard-ns
helm history devboard -n devboard-ns
helm get values devboard -n devboard-ns
helm get manifest devboard -n devboard-ns

kubectl get pods -n devboard-ns
kubectl get events -n devboard-ns --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod-name> -n devboard-ns
kubectl logs <pod-name> -n devboard-ns
```

Replace the example release, namespace, and Pod names with the ones in your environment.

## 19. Helm command cheat sheet

Keep this section handy during practice.

| Task                      | Command                                                  |
| ------------------------- | -------------------------------------------------------- |
| Check version             | `helm version`                                           |
| Create chart              | `helm create mychart`                                    |
| Validate chart            | `helm lint ./mychart`                                    |
| Render templates          | `helm template myapp ./mychart`                          |
| Install                   | `helm install myapp ./mychart -n lab --create-namespace` |
| Install or upgrade        | `helm upgrade --install myapp ./mychart -n lab`          |
| List releases             | `helm list -A`                                           |
| Check status              | `helm status myapp -n lab`                               |
| Show user-supplied values | `helm get values myapp -n lab`                           |
| Show rendered manifests   | `helm get manifest myapp -n lab`                         |
| View history              | `helm history myapp -n lab`                              |
| Roll back                 | `helm rollback myapp 1 -n lab`                           |
| Run chart tests           | `helm test myapp -n lab`                                 |
| Remove release            | `helm uninstall myapp -n lab`                            |
| Search repositories       | `helm search repo nginx`                                 |
| Update repositories       | `helm repo update`                                       |
| Package chart             | `helm package ./mychart`                                 |
| Update dependencies       | `helm dependency update ./mychart`                       |

For the official reference, see the [Helm documentation](https://helm.sh/docs/?utm_source=chatgpt.com) and [Helm command cheat sheet](https://helm.sh/docs/intro/cheatsheet/?utm_source=chatgpt.com).

## 20. Your practical Helm learning roadmap

Don't try to learn every advanced template function at once. Follow this progression and complete the tasks before moving on.

## Helm hands-on checklist

0/19 complete

### Stage 1 — Fundamentals

Explain chart, release, values, and revision in your own words.

Install Helm and confirm it uses your existing kind cluster.

Create nginx-lab and deploy it into a separate namespace.

### Stage 2 — Templates and values

Change the replica count using --set.

Create dev and production values files.

Add an optional Ingress using an if condition.

Use \_helpers.tpl to generate consistent names.

### Stage 3 — Release management

Upgrade the image or replica count.

Inspect release history and values.

Perform a rollback in the practice namespace.

Use helm lint and helm template before every deployment.

### Stage 4 — Real-world chart

Create a chart for your existing DevBoard frontend and backend.

Add Services, ConfigMaps, and configurable image tags.

Convert PostgreSQL to a chart only after planning PVC and data retention.

Deploy to a test namespace and verify application connectivity.

### Stage 5 — Advanced DevOps

Add resource requests, limits, probes, and optional HPA.

Package and version the chart.

Add chart validation and deployment to Jenkins or GitHub Actions.

Practise secret management, rollback, and a documented recovery procedure.

Reset checklist

## 21. Your first assignment

Start with only these tasks:

1. Install Helm and run `helm version`.
2. Create `nginx-lab` using `helm create`.
3. Replace its Deployment and Service templates with the examples in Section 7.
4. Run `helm lint` and `helm template`.
5. Install the chart into a new namespace and confirm that two NGINX Pods are running.
6. Change `replicaCount` from 2 to 4 and use `helm upgrade`.
7. Inspect the release history and practise a rollback.

Once you've completed these steps, you'll understand the core Helm workflow rather than just knowing the commands.

My recommendation: Learn Helm first with a small NGINX chart, then convert your existing DevBoard project into Helm. That will teach you how Helm handles the Kubernetes resources you already know, while giving you practical experience with templates, values, upgrades, and persistent storage.