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

### 3. Install & configure CAPI

```sh
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker
kubectl create configmap cilium-crs-cm --from-file=cilium-1.17.1.yaml
sleep 10
kubectl apply -f cilium-crs.yaml
```

### 4. Install & configure ArgoCD

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

### 5. Install & configure Syngit

```sh
helm repo add syngit https://syngit-org.github.io/syngit --force-update
helm install syngit syngit/syngit -n syngit \
  --create-namespace \
  --set providers.gitlab.enabled="true" \
  --set providers.github.enabled="true" \
  --set controller.replicas="1"
```

Create the `Secret` and the `RemoteUser` for **your** user (based on `syngit-configuration/user.yaml`).

Create the `RemoteSyncer` (based on `syngit-configuration/remotesyncer.yaml`).

### 8. Create the cluster

```sh
kubectl apply -f capi-docker-cluster-infra.yaml
kubectl apply -f cluster-only.yaml
```