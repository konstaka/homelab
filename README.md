# Homelab GitOps repository

Bootstrap-ready definitions of what's deployed in my homelab of two clusters, running on a repurposed workstation behind my fridge. This is my playground to try out different components and patterns before applying them in production.

## Infrastructure stack

| Component   | Version |
| ----------- | ------- |
| OPNsense    | 26.1.4  |
| Talos Linux | 1.12.5  |
| talosctl    | 1.12.5  |
| kubectl     | 1.35.3  |
| Kustomize   | 5.7.1   |
| Kubernetes  | 1.35.2  |
| Helm        | 4.1.3   |

The rest of the versions are pinned in the respective Helm subcharts or kustomizations.

## Clusters

### Alpha

"DMZ" cluster serving as application platform for publicly exposed services. Currently 1+2 nodes.

### Beta

"Main" cluster serving as a hub with ArgoCD deployed. Currently 3+2 nodes.

## Architecture

```mermaid
architecture-beta
    group pve(server)[pve]
    group cluster(internet)[cluster] in pve

    service router(internet)[router] in pve
    service control1(cloud)[control1] in cluster
    service worker1(cloud)[worker1] in cluster
    service worker2(cloud)[worker2] in cluster
    service nfs(disk)[nfs] in cluster
    service pg(database)[pg] in cluster
    service git(cloud)[git] in cluster
    junction clusterbus
    junction dmzbus1
    junction dmzbus2

    router:R -- L:dmzbus1
    dmzbus1:R -- L:dmzbus2
    dmzbus2:B -- L:clusterbus
    control1:B -- T:clusterbus
    worker1:T -- B:clusterbus
    worker2:L -- R:clusterbus
    nfs:B -- T:dmzbus2
    pg:B -- T:dmzbus1
    git:T -- B:dmzbus1
```

## My setup: Multiple clusters managed via one ArgoCD instance (hub and spoke)

### Prerequisites

- kubectl
- Helm
- Kubernetes cluster(s) with cloud load balancer support. I currently use [Talos Linux](https://docs.siderolabs.com/talos/v1.12/overview/what-is-talos), [OPNsense](https://opnsense.org/), and [MetalLB](https://metallb.io/) on [Proxmox](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview) to achieve this, your setup may be fancier.

#### If starting from scratch: Network setup with MetalLB

For each cluster in the system,

1. Ensure network connectivity from your workstation to the network where the nodes will be created. E.g. set up a VPN tunnel to the router and allow traffic from that tunnel to the cluster network.

2. After installing the Talos image on each node, assign static IPs on the router and add them as BGP neighbours for the MetalLB setup as per the [OPNsense docs](https://docs.opnsense.org/manual/dynamic_routing.html#bgp-section), with help from [this guide](https://github.com/bug1510/metallb-bgp-opnsense-deployment).

3. Bootstrap a Talos cluster as per the [docs](https://docs.siderolabs.com/talos/v1.12/getting-started/getting-started).

4. Install MetalLB to support creating LoadBalancer services in the cluster.

   ```
   kubectl apply -k metallb/installation/overlays/cluster-***
   ```

5. Allow it a little bit of time to settle in, then apply the BGP setup.

   ```
   kubectl wait --for condition=Ready pod -l app=metallb -n metallb-system
   kubectl apply -k metallb/configuration/overlays/cluster-***
   ```

   The cluster is now ready to assign external IPs to LoadBalancer services.

### Notes

I like to keep things reproducible. Therefore, I try to keep all versions of everything installed in the clusters in git by either making Kustomizations with remote resources or setting up Helm subcharts. With ArgoCD in place, having dangling Helm releases in the clusters would only confuse things, hence the choice for no `helm install` in this project and keeping everything except the load balancer setup in an app-of-apps pattern.

Hosting the homelab repo itself in-cluster is out of scope for practical purposes at this time. [Forgejo isn't HA-ready anyway](https://code.forgejo.org/forgejo-helm/forgejo-helm/src/branch/main/docs/ha-setup.md), so for now it can stay on a separate docker host.

### Install ArgoCD and App of Apps on the main cluster (hub)

ArgoCD's' CRDs exceed the size limit for `kubectl apply`, so `--server-side` is needed. `--force-conflicts` is needed for reliable upgrades, so I'll include it already.

```
helm dep update argocd
helm template argocd argocd -n argocd | kubectl apply --server-side --force-conflicts -f -
```

Wait for the pods to spin up and get the admin password:

```
kubectl wait --for condition=Ready pod -l app.kubernetes.io/part-of=argocd -n argocd
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

### Add the dmz cluster (spoke)

Ensure the API server endpoint of the spoke cluster is reachable from the main cluster nodes as well as from your workstation.

```
argocd login 192.168.3.11
argocd cluster add admin@talos-cluster-alpha
```

### Start the GitOps loop

```
helm template root-app | kubectl apply -f -
```

### Test everything with:

```
kubectl logs -n helloworld pod/test-pod
curl 192.168.3.10
curl 10.2.2.10
```
