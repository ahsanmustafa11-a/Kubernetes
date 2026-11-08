# Kustomize for Kubernetes — Beginner to Advanced Notes

Kustomize is the next tool worth learning alongside Helm. It solves a similar problem, but it takes a different approach.

The easiest way to remember the difference is:

- Helm: Use templates and values to generate Kubernetes YAML.
- Kustomize: Start with normal Kubernetes YAML and customize it for each environment.

Kustomize is built into `kubectl` through commands such as `kubectl apply -k` and `kubectl kustomize`. You usually don't need to install a separate executable to start learning it.

## 1. What is Kustomize?

Suppose your application has these files:

```
deployment.yaml
service.yaml
configmap.yaml
```

Your development environment needs one replica, but production needs five. You also need different image tags and namespaces.

Without a configuration tool, you might copy all the YAML files into separate folders and edit each copy. That creates duplication and makes maintenance harder.

Kustomize solves this with bases and overlays.

Base

Your standard Kubernetes YAML files

Deployment + Service + ConfigMap

Dev overlay

1 replica, dev image tag, dev namespace

Prod overlay

5 replicas, approved image tag, prod namespace

Both environments reuse the same base files without requiring separate copies of every manifest.

### Key terminology

| Term                 | definition                                                                               |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Kustomize            | A tool for customizing Kubernetes manifests                                              |
| `kustomization.yaml` | The file that tells Kustomize which resources and changes to combine                     |
| Base                 | The common, reusable Kubernetes configuration                                            |
| Overlay              | Environment-specific changes layered on top of a base                                    |
| Patch                | A change applied to a particular resource or field                                       |
| Generator            | A feature that creates resources such as ConfigMaps and Secrets from specified inputs    |
| Transformer          | A mechanism that modifies fields across resources, such as names, labels, and namespaces |

A useful distinction: Kustomize renders the final YAML, but it does not create Helm-style release history or manage chart revisions. Kubernetes receives the rendered resources when you apply them.

## 2. Install or verify Kustomize

On your existing Linux machine, run:

```
kubectl version --client
kubectl kustomize --help
```

The Kustomize version embedded in `kubectl` can vary by `kubectl` release.

You can also install the standalone Kustomize executable if you need a version independent of `kubectl`. For now, use the built-in commands to keep your learning environment simple.

## 3. Your first Kustomize project

We'll deploy NGINX just as we did with Helm, so you can compare the two tools directly.

Create this directory structure:

```
kustomize-lab/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

The `base` directory contains ordinary Kubernetes YAML. The two overlays customize it for development and production.

### Step 1: Create the Deployment

File: `base/deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.28.0
          ports:
            - name: http
              containerPort: 80
```

This is a normal Kubernetes Deployment, not a template. The `replicas` value is explicitly set to two.

### Step 2: Create the Service

File: `base/service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: http
```

The Service selects Pods using the `app: nginx` label.

### Step 3: Create the base kustomization file

File: `base/kustomization.yaml`

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

This tells Kustomize to include both files when it builds the base.

### Step 4: Build the base

From the `kustomize-lab` directory:

```
kubectl kustomize base/
```

Kustomize combines the resources and prints the resulting Kubernetes YAML. It does not deploy anything yet.

This is similar to `helm template`, which renders a Helm chart without installing it.

## 4. Bases and overlays — the most important Kustomize concept

A base contains the common configuration. An overlay customizes that configuration for a particular environment.

Imagine your NGINX Deployment starts with two replicas. Development needs one, while production needs five. You can keep the Deployment in the base and patch only the replica count in each overlay.

### Step 1: Create the development overlay

File: `overlays/dev/kustomization.yaml`

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - ../../base

replicas:
  - name: nginx
    count: 1

images:
  - name: nginx
    newTag: "1.28.0"
```

What does each part mean?

- `namespace: dev` — sets the namespace on namespaced resources.
- `resources` — includes the base directory.
- `replicas` — changes the Deployment's replica count to one.
- `images` — lets you change the image tag without editing the original Deployment.

The example keeps the same image tag in both environments for now. Later, you can give production a different approved tag.

### Step 2: Create the production overlay

File: `overlays/prod/kustomization.yaml`

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

replicas:
  - name: nginx
    count: 5

images:
  - name: nginx
    newTag: "1.28.0"
```

Now the same base can be reused in both environments.

### Step 3: Render the development configuration

```
kubectl kustomize overlays/dev/
```

Inspect the output. You should see:

- Deployment replicas set to `1`.
- Namespace set to `dev`.
- The image tag set to `1.28.0`.
- The Service included.

### Step 4: Render the production configuration

```
kubectl kustomize overlays/prod/
```

This output should have five replicas and the `production` namespace.

The key lesson: Kustomize doesn't need you to write a second Deployment file just to change the replica count.

The exact base-and-overlay pattern is documented in the official [Kustomize guide](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/?utm_source=chatgpt.com).

## 5. Deploy your Kustomize overlays

First, check that your intended Kubernetes context is selected:

```
kubectl config current-context
kubectl get nodes
```

Then apply the development environment:

```
kubectl apply -k overlays/dev/
```

Check its resources:

```
kubectl get deploy,svc,pods -n dev
```

Apply the production environment separately:

```
kubectl apply -k overlays/prod/
```

Check production:

```
kubectl get deploy,svc,pods -n production
```

Kustomize does not create the namespaces automatically just because the overlay specifies `namespace: dev` or `namespace: production`. Create them first:

```
kubectl create namespace dev
kubectl create namespace production
```

If a namespace already exists, don't create it again unnecessarily.

### Compare changes before applying

Use:

```
kubectl diff -k overlays/dev/
```

This previews the differences between the resources in the rendered configuration and the live cluster. It is a good habit before important changes.

### Clean up the practice resources

```
kubectl delete -k overlays/dev/
kubectl delete -k overlays/prod/
```

This deletes the resources rendered from those overlays, not necessarily the namespaces themselves. Check the target context and rendered output before deleting anything.

## 6. Kustomize patches

Patches are useful when you need to change specific fields in a resource without rewriting the entire resource.

For example, you might want to change a Deployment's replicas, resource limits, or container arguments.

### Example: Change replicas using a patch

Create `overlays/prod/replicas-patch.yaml`:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 5
```

Then update `overlays/prod/kustomization.yaml`:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

patches:
  - path: replicas-patch.yaml
```

Now build the production overlay:

```
kubectl kustomize overlays/prod/
```

The Deployment will have five replicas.

You might wonder why we used `replicas` in the earlier example and a patch here. Both are valid approaches:

- Use the `replicas` field for simple replica-count changes.
- Use `patches` for more general changes involving one or more resource fields.

Modern Kustomize supports the `patches` field for strategic merge and JSON patch operations.

### Other common customization options

| Field                | Purpose                                     |
| -------------------- | ------------------------------------------- |
| `namePrefix`         | Adds a prefix to resource names             |
| `nameSuffix`         | Adds a suffix to resource names             |
| `namespace`          | Sets the namespace for namespaced resources |
| `labels`             | Adds or modifies labels                     |
| `commonAnnotations`  | Adds annotations to resources               |
| `images`             | Changes image names, tags, or digests       |
| `configMapGenerator` | Generates ConfigMaps                        |
| `secretGenerator`    | Generates Secrets                           |
| `patches`            | Modifies selected resources                 |

Be careful with labels and selectors: changing a Pod's labels without updating the corresponding selectors can break a Service or workload controller.

## 7. ConfigMap and Secret generation

Kustomize can generate ConfigMaps and Secrets from files or literals.

### Generate a ConfigMap

Suppose you have `app.properties`:

```
APP_MODE=production
LOG_LEVEL=info
```

Add this to a `kustomization.yaml`:

```
configMapGenerator:
  - name: app-config
    files:
      - app.properties
```

When you build the configuration, Kustomize generates a ConfigMap and normally adds a content hash suffix to its name. References to generated ConfigMaps in supported resource fields are updated accordingly, which helps trigger workload rollouts when configuration changes.

### Generate a Secret

An illustrative example using a local environment file:

```
secretGenerator:
  - name: app-secret
    envs:
      - app-secret.env
```

The `app-secret.env` file might contain:

```
API_TOKEN=example-token
```

This is only a syntax example. Don't commit real credentials to Git, and don't assume that a generated Secret is encrypted simply because it is a Kubernetes Secret. Use appropriate secret-management and access-control practices.

## 8. Kustomize with your DevBoard project

You already have Kubernetes resources for the frontend, backend, and PostgreSQL. Kustomize is a good fit if you want to preserve those ordinary YAML files and apply different configurations to development and production.

A possible directory structure is:

```
devboard/
├── base/
│   ├── namespace.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── configmap.yaml
│   ├── postgres-service.yaml
│   ├── postgres-statefulset.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

This is a proposed structure, not a replacement for your current files. You can keep additional resources, such as HPA, RBAC, and PVC configuration, wherever they belong in your chosen design.

A base `kustomization.yaml` could list the common resources:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - frontend-deployment.yaml
  - frontend-service.yaml
  - backend-deployment.yaml
  - backend-service.yaml
  - configmap.yaml
  - postgres-service.yaml
  - postgres-statefulset.yaml
```

Your development overlay could customize image tags and replica counts, while the production overlay could apply different replica counts, resource requests, and other settings.

For PostgreSQL, don't assume that changing the overlay will safely migrate or reuse an existing PVC. Preserve the current data and test storage changes separately.

# 9. Helm vs. Kustomize — complete comparison

This is the most important section when deciding which tool to learn first.

## Helm

Package, template, configure and manage releases.

## Kustomize

Customize existing Kubernetes YAML using overlays and patches.

Helm documentation: [helm.sh/docs](https://helm.sh/docs/?utm_source=chatgpt.com) · Kustomize documentation: [Kubernetes Kustomize guide](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/?utm_source=chatgpt.com)

| Feature                   | Helm                                                          | Kustomize                                                                          |
| ------------------------- | ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Main purpose              | Package and manage Kubernetes applications                    | Customize Kubernetes manifests                                                     |
| Input                     | Chart templates and values                                    | Existing YAML resources and patches                                                |
| Templates                 | Go templates and template functions                           | No general-purpose templating language                                             |
| Environment configuration | Different values files                                        | Different overlays                                                                 |
| Reuse                     | Charts and subcharts                                          | Bases and overlays                                                                 |
| Third-party applications  | Extensive chart ecosystem                                     | Can customize raw YAML; supports chart rendering through optional Helm integration |
| Install and upgrade       | `helm install`, `helm upgrade`                                | `kubectl apply -k`                                                                 |
| Release history           | Built-in release history and rollback                         | No built-in Helm-style release history                                             |
| Rollback                  | `helm rollback`                                               | Reapply a previous Git revision or known-good manifest set                         |
| Package distribution      | Versioned charts and repositories/registries                  | Usually source-controlled directories and manifests                                |
| Learning curve            | Template syntax and chart structure to learn                  | Often simpler when you already understand Kubernetes YAML                          |
| Best fit                  | Reusable applications, packaged software, complex deployments | Existing YAML, environment-specific changes, GitOps workflows                      |

Both tools can be used in real production environments. Neither one automatically makes a deployment secure, highly available, or reliable. Those properties depend on your Kubernetes configuration and operational practices.

## 10. Same task, two different approaches

Suppose you need to deploy the same application with one replica in development and five in production.

### With Helm

`values-dev.yaml`:

```
replicaCount: 1
```

`values-prod.yaml`:

```
replicaCount: 5
```

Deploy:

```
helm upgrade --install myapp ./my-chart \
  -n dev \
  -f values-dev.yaml
```

```
helm upgrade --install myapp ./my-chart \
  -n production \
  -f values-prod.yaml
```

Helm uses template expressions such as `{{ .Values.replicaCount }}` to generate the appropriate manifests.

### With Kustomize

Keep the original Deployment in `base/deployment.yaml`:

```
spec:
  replicas: 2
```

Then configure each overlay:

```
# overlays/dev/kustomization.yaml
resources:
  - ../../base

replicas:
  - name: nginx
    count: 1
```

```
# overlays/prod/kustomization.yaml
resources:
  - ../../base

replicas:
  - name: nginx
    count: 5
```

Deploy:

```
kubectl apply -k overlays/dev/
kubectl apply -k overlays/prod/
```

Notice the difference: Helm changes the values used to render templates, while Kustomize transforms ordinary YAML into environment-specific YAML.

## 11. Can you use Helm and Kustomize together?

Yes. For example, Helm can generate the base application resources, and Kustomize can apply additional patches to the rendered manifests. Kustomize also has optional Helm chart inflation support, which must be explicitly enabled in supported workflows.

A simpler approach in CI/CD is to render a Helm chart first and then apply a Kustomize transformation to the rendered YAML. This requires careful handling of generated files, resource names, and ownership.

Don't combine both tools just because you can. Use the combination only when each tool solves a specific problem.

## 12. Which should you learn first?

For your DevOps and Kubernetes learning path, I recommend:

1. Learn Kustomize fundamentals first. You already have raw YAML files, so practice bases, overlays, image changes, and patches without learning another template language.
2. Continue with Helm. Learn chart structure, values, templates, upgrades, rollback, dependencies, and packaging.
3. Convert the same small application using both tools. Compare the folder structure, rendered YAML, and deployment workflow.
4. Integrate one tool into CI/CD. Add validation and deployment to Jenkins or GitHub Actions, then learn GitOps workflows later.

A practical rule of thumb:

- Choose Kustomize when you want to maintain normal Kubernetes YAML with a small number of environment-specific changes.
- Choose Helm when you want reusable, configurable application packages, especially when installing and distributing complex applications.
- Learn both because real-world Kubernetes teams use different approaches depending on the application and deployment workflow.

## 13. Your final practice assignment

Use your NGINX lab first, then move to DevBoard.

## Kustomize + Helm lab

0 of 8

Create a Kustomize base containing an NGINX Deployment and Service.

Create dev and production overlays with different replica counts.

Change an image tag through the Kustomize images field.

Use kubectl kustomize to inspect the generated YAML.

Use kubectl diff -k before applying a change.

Deploy both overlays into separate namespaces.

Create a Helm chart for the same NGINX application.

Deploy both approaches in separate test namespaces and compare them.

Reset practice checklist

Official references to keep bookmarked:

- [Kustomize: Kubernetes official guide](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/?utm_source=chatgpt.com)
- [Kustomize reference documentation](https://kubectl.docs.kubernetes.io/references/kustomize/?utm_source=chatgpt.com)
- [Helm official documentation](https://helm.sh/docs/?utm_source=chatgpt.com)
- [Helm chart template guide](https://helm.sh/docs/chart_template_guide/?utm_source=chatgpt.com)