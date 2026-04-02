# Homelab GitOps repository

Bootstrap-ready definitions of what's deployed in my home cluster of 1 control plane node and 2 worker nodes, running on a repurposed workstation behind my fridge. This is my playground to try out different components and patterns before applying them in production clusters.

## Infrastructure stack

| Component                   | Version |
| --------------------------- | ------- |
| Proxmox Virtual Environment | 9.1.4   |
| OPNsense                    | 26.1.4  |
| Talos Linux                 | 1.12.5  |
| talosctl                    | 1.12.5  |
| kubectl                     | 1.35.3  |
| Kustomize                   | 5.7.1   |
| Kubernetes                  | 1.35.2  |
| Helm                        | 4.1.3   |

The rest of the versions are pinned in the respective Helm subcharts or kustomizations.

## Architecture

```mermaid
architecture-beta
    group pve(server)[pve]
    group vps(server)[vps]
    group newtnet(cloud)[newt] in pve
    group cluster(cloud)[cluster] in pve

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
    service pangolin(cloud)[pangolin] in vps
    service newt(cloud)[newt] in newtnet

    router:R -- L:dmzbus1
    dmzbus1:R -- L:dmzbus2
    dmzbus2:B -- L:clusterbus
    control1:B -- T:clusterbus
    worker1:T -- B:clusterbus
    worker2:L -- R:clusterbus
    nfs:B -- T:dmzbus2
    pg:B -- T:dmzbus1
    git:T -- B:dmzbus1
    pangolin:B -- T:newt
    newt:R -- L:router
```

## Bootstrap a cluster

Create nodes as desired and install the Talos image on each as per the docs. Assign the nodes static IPs on the router and add them as BGP neighbours.

I like to keep things reproducible. Therefore, I try to keep all versions of everything installed in the cluster in git by either making Kustomizations with remote resources or setting up Helm subcharts. I also plan on using GitOps patterns later to manage the apps, so having dangling Helm releases in the cluster would only confuse things, hence the choice for no `helm install` in this project.

Hosting the homelab repo itself in the cluster is out of scope for practical purposes. Having it on a separate machine makes deploying new clusters much easier, and [Forgejo isn't HA-ready anyway](https://code.forgejo.org/forgejo-helm/forgejo-helm/src/branch/main/docs/ha-setup.md), so for now it can stay on a docker host.

### Load balancing

Install MetalLB with

```
kubectl apply -k metallb/installation
```

Allow it a little bit of time to settle in, then apply the BGP setup with

```
kubectl apply -k metallb/configuration
```

The cluster is now ready to assign external IPs to load balancer services.

#### Reverse proxy

I have Pangolin installed on an external VPS, where my public DNS records are also pointed. The wireguard tunnel terminates behind the OPNsense router, so additionally port forwarding needs to be set up for the tunnel to reach the load balancer(s).

### Ingress

Install Traefik with

```
helm dep update traefik
helm template traefik traefik -n traefik | kubectl apply -f -
```

Now we can add service and ingress files to apps to expose them in a more controlled manner.

### Storage

Install the NFS CSI driver with a default storage class configured, pointing to the NFS server:

```
helm dep update csi-driver-nfs
helm template csi-driver-nfs csi-driver-nfs -n kube-system | kubectl apply -f -
```

Now there is a distributed storage backend in the cluster.

### Test everything with:

```
helm template helloworld helloworld | kubectl apply -f -
kubectl logs -n helloworld pod/test-pod
curl 10.2.2.10
```

### ArgoCD

ArgoCD's' CRDs exceed the size limit for `kubectl apply`, so `--server-side` is needed. `--force-conflicts` is needed for reliable upgrades, so I'll include it already.

```
helm dep update argocd
helm template argocd argocd -n argocd | kubectl apply --server-side --force-conflicts -f -
helm template root-app | kubectl apply -f -
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

## TODO

### Storage

For now, since everything happens on the same machine, I'll use a single storage VM serving NFS. In the future, it might make sense to add a cloud native storage solution like Longhorn or Piraeus.

### Database

In a similar fashion, to get things moving, I'll make use of a non-HA, centralised database server. I'll use PostgreSQL since it's a very common requirement of various applications, and paves way for a possible future CloudNativePG deployment.
