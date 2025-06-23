## Installation

Go to the `demo/` subfolder.

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

### 3. Install syngit

```sh
helm repo add syngit https://syngit-org.github.io/syngit --force-update
helm install syngit syngit/syngit -n syngit \
  --create-namespace \
  --set providers.gitlab.enabled="true" \
  --set providers.github.enabled="true" \
  --set controller.replicas="1"
```

### 4. Install CAPI & configure CAPI

```sh
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker
kubectl create configmap cilium-crs-cm --from-file=demo/deep-dive/cilium-1.17.1.yaml
sleep 15
```

```sh
kubectl apply -f demo/deep-dive/cilium-crs.yaml
kubectl apply -f demo/deep-dive/capi-docker-cluster-infra.yaml
```

### 5. Install & configure ArgoCD

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

And connect your repo

## Basic - Procedure

All of the basic procedure is located under the `demo/basic/` folder.

### 1. Create the `RemoteUser`

Fill the `Secret` and the mail.

`kubectl apply -f user.yaml`

### 2. Create the syngit resources

```sh
kubectl apply -f remotetarget.yaml
kubectl apply -f remoteuserbinding.yaml
kubectl apply -f remotesyncer.yaml
```

### 3. Create the deployment

`kubectl create deploy test --image=nginx`

## Deep dive - Procedure

All the procedure has to be done in the `demo/deep-dive/` directory.

### 3. Configure Syngit

Create the `Secret` and the `RemoteUser` for **your** user (based on `syngit-configuration/user.yaml`).

Create the `RemoteSyncer` (based on `syngit-configuration/remotesyncer.yaml`).

```sh
kubectl apply -f syngit-configuration/rbac.yaml
kubectl apply -f syngit-configuration/user-a.yaml --as user-a
kubectl apply -f syngit-configuration/user-b.yaml --as user-b
kubectl apply -f syngit-configuration/remotesyncer.yaml
```

### 4. Create the cluster

```sh
kubectl apply -f cluster-only.yaml --as user-a
```

```sh
kubectl apply -f cluster-only.yaml --as user-b
```