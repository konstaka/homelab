# Homelab GitOps repository

Bootstrap-ready definitions of what's running in my home cluster of 1 control plane node and 2 worker nodes.

## Infrastructure stack

### Proxmox

Hosts the router and cluster nodes as VMs. Runs on a repurposed workstation behind my fridge.

### OPNsense

Router/firewall to segment the network, provide a WireGuard VPN tunnel for management, and manage BGP peerings with the worker nodes for load balancing.

### MetalLB

Network load balancer that will automatically assign external IPs for LoadBalancer resources.

### Talos Linux

Kubernetes-focused OS installed on the cluster nodes.

### Pangolin

Identity-aware reverse proxy installed on a VPS outside of my home infrastructure to publish the services.

## Bootstrap a cluster

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
