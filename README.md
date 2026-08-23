<div align="center">

<img src="https://s3.login.no/beehive/img/logo/logo-white-small.svg" alt="Login logo" width="80" height="80" />

<h1>Apiary</h1>

<p><strong>Kubernetes cluster for Login.</strong><br />An apiary is where beehives are kept.</p>

<p>
  <img src="https://img.shields.io/badge/Kubernetes-fd8738?style=flat-square&logo=kubernetes&logoColor=white" alt="Kubernetes" />
  <img src="https://img.shields.io/badge/k3s-fd8738?style=flat-square&logo=k3s&logoColor=white" alt="k3s" />
  <img src="https://img.shields.io/badge/Argo_CD-fd8738?style=flat-square&logo=argo&logoColor=white" alt="Argo CD" />
</p>

</div>

---

> **Work in progress.** apiary is being built and services are still migrating onto it. Nothing here serves production traffic yet. Expect things to move.

Workloads running on **apiary**, the Login Kubernetes cluster. Argo CD watches this repository and applies whatever is committed here.

The cluster itself is not defined here. It is a k3s server declared in the `nixos` repository as `hosts/apiary-1`.

## Adding a Service

1. Create a folder under `services/`. The folder name becomes the application and namespace name.
2. Add manifests and a `kustomization.yaml` inside it.
3. Push to main.

The `services` ApplicationSet generates an Argo CD application per folder. No application manifest is needed. Deleting the folder removes the service.

Argo CD polls every three minutes. To apply immediately:

```bash
kubectl -n argocd annotate application root argocd.argoproj.io/refresh=hard --overwrite
```

## Cluster Access

The node sits on SERVERUNSECURE and is not routable from outside the network. Tunnel through the jump host:

```bash
ssh -f -N -L 6443:10.30.0.188:6443 onprem
export KUBECONFIG=$PWD/kubeconfig.tunnel
kubectl get nodes
```

`kubeconfig` targets the node directly, `kubeconfig.tunnel` targets localhost. Both are gitignored.

## Argo CD

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Reachable at `https://localhost:8080`. Credentials are in the TekKom 1Password vault.

Argo CD manages itself from `infra/argocd`. Upgrading is a version bump in `kustomization.yaml`. Reinstalling requires server side apply, as the `applicationsets` CRD exceeds the annotation size limit used by client side apply:

```bash
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.1/manifests/install.yaml
```

## Scaling

`clusterInit` is enabled, so etcd accepts additional servers. Add a host under `hosts/` in the `nixos` repository with `serverAddr` and `tokenFile` set. Go from one server to three, never two, as a two member etcd requires both nodes to be available.

## Project Structure

- `bootstrap/` - Root application, applied once per cluster
- `apps/` - Argo CD applications and application sets
- `infra/` - Cluster components
- `services/` - One folder per workload

## Cluster

| | |
|---|---|
| Node | `apiary-1`, VM 8000 on `pve2` |
| Address | `10.30.0.188`, bridge `SU` |
| Resources | 12 vCPU, 32 GB RAM, 100 GB disk |
| Distribution | k3s on NixOS |
| Storage | `local-path`, default StorageClass |
| Ingress | None in-cluster. The existing reverse proxy is still the edge |
