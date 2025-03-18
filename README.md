# Various examples for updating image digests

## Install ArgoCD and Image Updater

Install ArgoCD
```bash
kubectl create ns argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Install ArgoCd Image Updater
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

## Directory tree

```bash
├── README.md
├── argo
│   ├── apps                    # ArgoCD app manifests
│   │   └── cert-manager.yaml 
│   └── install
│       ├── argocd-image-updater.yaml   # ArgoCD image updater installation manifests
│       └── kustomization.yaml
├── k8s
    └── cert-manager
        └── custom-values.yaml  # Helm values file
```

## Example Application Resource

Create a secret 
```bash
k create secret generic ky-rafaels-pull-secret \
--from-file=.dockerconfigjson=/Users/kyle.rafaels/.docker/config.json \
--type=kubernetes.io/dockerconfigjson -n argocd
```

Create a registry configMap 
```bash
cat << EOF > registry.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: argocd-image-updater-config
    namespace: argocd
    data:
    registries.conf: |
        registries:
        - name: Chainguard
          prefix: cgr.dev
          api_url: https://cgr.dev
          credentials: pullsecret:argocd/ky-rafaels-pull-secret
          defaultns: library
          default: true
EOF
```

Reference pull secret in annotation 
```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations: 
    argocd-image-updater.argoproj.io/image-list: controller=cgr.dev/ky-rafaels.example.com/cert-manager-controller, webhook=cgr.dev/ky-rafafels.example.com/cert-manager-webhook, cainjector=cgr.dev/ky-rafaels.example.com/cert-manager-cainjector
    argocd-image-updater.argoproj.io/cert-manager.update-strategy: digest
    argocd-image-updater.argoproj.io/controller.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/controller.helm.image-tag: image.tag
    argocd-image-updater.argoproj.io/webhook.helm.image-name: webhook.image.repository
    argocd-image-updater.argoproj.io/webhook.helm.image-tag: webhook.image.tag
    argocd-image-updater.argoproj.io/cainjector.helm.image-name: cainjector.image.repository
    argocd-image-updater.argoproj.io/cainjector.helm.image-tag: cainjector.image.tag
    argocd-image-updater.argoproj.io/git-repository: https://github.com/ky-rafaels/image-updater.git
    argocd-image-updater.argoproj.io/write-back-method: argocd
    argocd-image-updater.argoproj.io/git-branch: main
    # Optionally use git write back strategy
    # argocd-image-updater.argoproj.io/write-back-method: git
    # regexp below specifies version tagging in 0.0.0 format
    # argocd-image-updater.argoproj.io/myalias.allow-tags: regexp:^1\.[0-9]+\.[0-9]+$
spec:
  destination:
    namespace: cert-manager
    server: 'https://kubernetes.default.svc'
  sources:
    - repoURL: https://charts.jetstack.io
      chart: cert-manager 
      targetRevision: v1.17.1
      helm:
        ignoreMissingValueFiles: true
        valueFiles:
          - $values/k8s/cert-manager/custom-values.yaml
    - repoURL: https://github.com/ky-rafaels/image-updater.git   # External repo for values
      targetRevision: main
      ref: values
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
      - ServerSideApply=true
```

## Using cgr private registry

Create a secret 
```bash
k create secret generic ky-rafaels-pull-secret \
--from-file=.dockerconfigjson=/Users/kyle.rafaels/.docker/config.json \
--type=kubernetes.io/dockerconfigjson -n argocd
```
