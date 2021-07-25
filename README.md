<p align="center">
<img width="65%" src="https://cloud.google.com/kubernetes-engine/kubernetes-comic/assets/panel-5.png">
<br>
(from https://cloud.google.com/kubernetes-engine/kubernetes-comic)
<h1 align="center">⛵ AthenaKube ⛵</h1>

Repository containing Kubernetes related files, used in our Kubernetes cluster. Useful for local development, to learn how we run our stuff, ~~and to avoid me losing those files~~. :^)

## Deployments & Stuff

We don't run everything within Kubernetes... yet. So here's a list of what of our services we run inside Kubernetes. All configuration files are present in this repository, in their respective folders.

#### Loritta
* Gabriela's Image Server
* Showtime

## Setting up Kubernetes

Setting up Kubernetes manually is pretty hard, so that's why we use [k3s](https://k3s.io) for our cluster!

Installing k3s' is pretty straightforward if you want to use for development, just one single command...
```bash
curl -sfL https://get.k3s.io | sh -
```
...and Kubernetes/k3s is installed in your computer, nice!

However we need to change some options to fit our needs, because...
*  We want to run a high availiability node (two master nodes for the control plane, tasks are not executed there)
* Using an external datastore (PostgreSQL) 
* The Kubernetes nodes to communicate between them with our VXLAN connection (using Proxmox's Experimental SDN Support)
* Load balancing (just a nginx server that points to our both master nodes)

Similar to what [Techno Tim shows in his k3s tutorial](https://youtu.be/UoOcLXfa8EU).

### Master Node
```bash
curl -sfL https://get.k3s.io | sh -s - server \
    --node-external-ip="172.29.0.1" `# VXLAN IP, this can be removed if ran locally` \
    --token="athena" `# The cluster token` \
    --datastore-endpoint="postgres://username:password@172.29.2.1:5432" `# PostgreSQL connection string` \
    --tls-san="172.16.0.2" `# Load balancer IP, this can be removed if not using a Load Balancer` \
    --disable="traefik" `# We don't use traefik, we proxy our stuff from a nginx server outside of the Kubernetes cluster` \
    --flannel-iface="ens19" `# If you have multiple network interfaces please ensure that --flannel-iface points to the interface where nodes have shared networking. See issue #1968` \
    --node-taint CriticalAddonsOnly=true:NoExecute `# Used for master clusters, does not allow jobs to be scheduled on this node`
```

### Worker Nodes
```bash
curl -sfL https://get.k3s.io | sh -s - agent \
    --server="https://172.16.0.2:6443" `# The load balancer IP, if not using a load balancer, use the IP of any of the master clusters` \
    --node-external-ip="172.29.1.2" `# VXLAN IP, this can be removed if ran locally` \
    --token="athena" `# The cluster token` \
    --flannel-iface="ens19" `# If you have multiple network interfaces please ensure that --flannel-iface points to the interface where nodes have shared networking. See issue #1968` \
```

If everything goes well, all nodes should be present in the `kubectl get nodes` command :)

```bash
root@k3s-athena1:/home/mrpowergamerbr/AthenaKube# kubectl get nodes
NAME          STATUS   ROLES                  AGE     VERSION
k3s-worker2   Ready    <none>                 4h      v1.21.3+k3s1
k3s-worker3   Ready    <none>                 3h53m   v1.21.3+k3s1
k3s-worker1   Ready    <none>                 6h8m    v1.21.3+k3s1
k3s-athena1   Ready    control-plane,master   6h38m   v1.21.3+k3s1
k3s-athena2   Ready    control-plane,master   6h35m   v1.21.3+k3s1
```
