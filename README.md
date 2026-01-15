## Installation

Go to the `demo/` subfolder.

### 1. Create the managament cluster

```sh
kind create cluster --name demo-cluster
```

### 2. Install cert-manager

```sh
helm repo add jetstack https://charts.jetstack.io --force-update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
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
kubectl port-forward service/argocd-server -n argocd 8081:443
```

And connect your repo

### 5. Install headlamp

```sh
# first add our custom repo to your local helm repositories
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/

# now you should be able to install headlamp via helm
helm install headlamp headlamp/headlamp --namespace kube-system
```

```sh
kubectl create token -n kube-system headlamp
kubectl port-forward -n kube-system service/headlamp 8080:80
```

Go to `http://127.0.0.1:8080`

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

## Multi-users - Procedure

All the procedure has to be done in the `demo/multi-users` directory.

Fill the `Secret` and the mail in `user-a.yaml` & `user-b.yaml`.

```sh
kubectl apply -f rbac.yaml
kubectl apply -f user-a.yaml --as user-a
kubectl apply -f user-b.yaml --as user-b
kubectl apply -f remotesyncer.yaml
```

`kubectl create deploy test --image=nginx --as user-a`
