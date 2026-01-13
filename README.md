# argocd-apps

This repository contains Argo CD Application definitions and Helm value overrides
used to deploy applications to Kubernetes using a GitOps workflow.

Each application is defined as an Argo CD Application that points back to this
repository and deploys a Helm chart with environment specific values.

---

## Repository structure

```text
argocd-apps/
└── env/
    └── <cluster>/
        └── <app>/
            ├── Chart.yaml
            ├── app.yaml
            └── values.yaml
```

Each application directory represents a single Argo CD Application deployed to
a specific cluster.

---

## Requirements

- Kubernetes
- Argo CD
- Argo CD Image Updater (optional, for automatic image updates)

---

## Example

This example deploys a sample application using the reusable microservice
Helm chart from the titus-charts repository.

The titus-charts/microservice chart includes optional support for Argo CD Image
Updater. When enabled, it can detect newly published container images and commit
updated image tags back to this repository.

### Chart.yaml

Declares the Helm dependency:

```yaml
apiVersion: v2
name: hello
version: 1.0.0
dependencies:
  - name: microservice
    version: "0.*"
    repository: "https://soft-titus.github.io/titus-charts"
```

---

### Argo CD Application

The app.yaml file defines how Argo CD deploys and syncs the application:

- Fully automated sync
- Namespace auto-creation
- Helm values sourced from this repo

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:soft-titus/argocd-apps.git
    path: "env/my-cluster/hello"
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: hello
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

### Helm Values

The values.yaml file configures values required by the Helm chart
(in this case, the microservice dependency):

```yaml
microservice:
  ...
  image:
    repository: ghcr.io/soft-titus/hello
    tag: "1.0.0"
    argocdImageUpdater:
      enabled: true
      applicationNamespace: "argocd"
      applicationNamePattern: "hello"
      images:
      - alias: app
        imageName: ghcr.io/soft-titus/hello
        commonUpdateSettings:
          updateStrategy: semver
          allowTags: "regexp:^1\\.[0-9]+\\.[0-9]+$"
        manifestTargets:
          helm:
            name: microservice.image.repository
            tag: microservice.image.tag
      writeBackConfig:
        method: git
        gitConfig:
          writeBackTarget: "helmvalues:/env/my-cluster/hello/values.yaml"
  ...
```

When a new image tag is detected, Argo CD Image Updater commits the updated value
to env/my-cluster/hello/values.yaml under microservice.image.tag.
