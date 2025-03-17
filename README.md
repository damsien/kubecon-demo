## Procedure

All the procedure has to be done in the `demo` directory.

### 1. Create the managament cluster

```sh
kind create cluster --name management-cluster --config kind-workload-cluster-config.yaml
```

### 2. Install cert-manager

```sh
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true
```

### 3. Install & configure ArgoCD

```sh
helm repo add argo https://argoproj.github.io/argo-helm
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd --set crds.install=true
```

Get the argo-cd's secret
```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Serve the dashboard
```sh
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Change the default reconciliation timer
```sh
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"timeout.reconciliation":"5s"}}'
kubectl rollout restart -n argocd statefulset argocd-application-controller
```

Create the `Application`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capi-demo
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  source:
    directory:
      jsonnet: {}
      recurse: true
    path: .
    repoURL: <REPO_URL>
    targetRevision: main
  syncPolicy:
    automated:
      allowEmpty: true
      prune: true
      selfHeal: true
    syncOptions:
    - PrunePropagationPolicy=background
```

### 4. Install & configure Syngit

```sh
helm repo add syngit https://syngit-org.github.io/syngit --force-update
helm install syngit syngit/syngit -n syngit \
  --create-namespace \
  --set providers.gitlab.enabled="true" \
  --set providers.github.enabled="true" \
  --set controller.replicas="1"
```

Create the `Secret` and the `RemoteUser`.

Create the `RemoteSyncer`

```yaml
apiVersion: syngit.io/v1beta3
kind: RemoteSyncer
metadata:
  name: capi-cluster-create-update-delete
  annotations:
    syngit.io/remotetarget.pattern.one-or-many-branches: main
spec:
  remoteRepository: <REPO_URL>
  defaultBranch: main
  strategy: CommitApply
  targetStrategy: OneTarget
  bypassInterceptionSubjects:
  - kind: ServiceAccount
    name: system:serviceaccount:argocd:argocd-application-controller
    namespace: argocd
  - kind: ServiceAccount
    name: system:serviceaccount:argocd:argocd-server
    namespace: argocd
  - kind: ServiceAccount
    name: system:serviceaccount:capi-system:capi-manager
    namespace: capi-system
  defaultUnauthorizedUserMode: Block
  rootPath: capi
  excludedFields:
    - spec.controlPlaneEndpoint.host
    - spec.controlPlaneEndpoint.port
  scopedResources:
    rules:
    - apiGroups: ["cluster.x-k8s.io"]
      apiVersions: ["v1beta1"]
      resources: ["clusters"]
      operations: ["CREATE", "UPDATE","DELETE"]
```

### 5. Install CAPI

```sh
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker
```

### 6. Create the ClusterResourceSet for Cilium

```sh
kubectl create configmap cilium-crs-cm --from-file=cilium-1.17.1.yaml
kubectl apply -f cilium-crs.yaml
```

### 7. Generate docker cluster manifests

```sh
clusterctl generate cluster my-cluster --flavor development \
  --kubernetes-version=v1.32.0 \
  --control-plane-machine-count=1 \
  --worker-machine-count=3 \
  > capi-docker-cluster.yaml
```

### 8. Create the cluster

```sh
kubectl apply -f capi-docker-cluster.yaml
```

OR

```sh
kubectl apply -f capi-docker-cluster-infra.yaml
kubeclt apply -f cluster-only.yaml
```