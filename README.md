# Homelab GitOps repository

Bootstrap-ready definitions of what's running in my home cluster of 1 control plane node and 2 worker nodes.

## Infrastructure stack

### Proxmox 9.1.4

Hosts the router and cluster nodes as VMs. Runs on a repurposed workstation behind my fridge.

### OPNsense 26.1.4

Router/firewall to segment the network, provide a WireGuard VPN tunnel for management, and manage BGP peerings with the worker nodes for load balancing.

### Talos Linux 1.12.5

Kubernetes-focused OS installed on the cluster nodes.

### MetalLB 0.15.3

Network load balancer that will automatically assign external IPs for LoadBalancer resources.

### Traefik Proxy 3.6.11

Installed as ingress controller to route incoming http(s) traffic to services and manage the TLS certificates.

### Pangolin

Identity-aware reverse proxy installed on a VPS outside of my home infrastructure to publish the services.

## Bootstrap a cluster

Create nodes as desired and install the Talos image on each as per the docs. Assign the nodes static IPs on the router and add them as BGP neighbours.

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
