# Homelab GitOps repository

Bootstrap-ready definitions of what's deployed in my home cluster of 1 control plane node and 2 worker nodes, running on a repurposed workstation behind my fridge.

## Infrastructure stack

| Thing                       | Version |
| --------------------------- | ------- |
| Proxmox Virtual Environment | 9.1.4   |
| OPNsense                    | 26.1.4  |
| Talos Linux                 | 1.12.5  |
| talosctl                    | 1.12.5  |
| MetalLB                     | 0.15.3  |
| Traefik Proxy               | 3.6.11  |
| kubectl                     | 1.35.3  |
| Kustomize                   | 5.7.1   |
| Kubernetes                  | 1.35.2  |
| Helm                        | 4.1.3   |

## Bootstrap a cluster

Create nodes as desired and install the Talos image on each as per the docs. Assign the nodes static IPs on the router and add them as BGP neighbours.

I like to keep things reproducible. Therefore, I try to keep all versions of everything installed in the cluster in git by either making Kustomizations with remote resources or setting up Helm subcharts.

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

### Ingress

Install Traefik with

```
helm template traefik -n traefik | kubectl apply -f -
```

Now we can add service and ingress files to apps to expose them in a more controlled manner.

### Storage (next: Longhorn/Piraeus? CloudNativePG?)

To get persistent storage in a cluster, it needs to have some backend - the initial node-attached disks are to be considered ephemeral. For now, since everything happens on the same machine, let's use a single storage VM serving NFS.

Install the NFS CSI driver with a default storage class configured, pointing to the NFS server:

```
helm template csi-driver-nfs -n kube-system | kubectl apply -f -
```

Now there is a distributed storage backend in the cluster. Test everything with:

```
helm template helloworld | kubectl apply -f -
kubectl logs -n helloworld pod/test-pod
curl 10.2.2.10/helloworld
```

#### Database

In a similar fashion, to get things moving, let's make use of a non-HA, centralised database server. I'll use PostgreSQL since it's a very common requirement of various applications.
```
