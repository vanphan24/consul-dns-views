# consul-dns-views

This demo will the latest Consul DNS Views feature for Kubernetes Consul 1.20. This feature allows developers use Consul service discovery and migrate from a single-tenant to multi-tenant deployment without changing or updating their existing application code.  
This demo will deploy two Consul clusters on Kubernetes, a server cluster in the default parttion and a client cluster in a partition called 'ap1'. We will use Azure Kuberntes Service (AKS) although any Kubernetes cluster should work.
The Consul client cluster will include a Consul DNS Proxy that will proxy all Consul DNS queries and return the appropriate response. 


![alt text](https://github.com/vanphan24/cluster-peering-demo/blob/main/images/Screen%20Shot%202022-08-18%20at%2010.40.40%20AM.png "Cluster Peering Demo")

# Pre-reqs

1. You have two Kubernetes clusters available. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.
2. Add or update your hashicorp helm repo:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
or
```
helm repo update hashicorp
```
  
3. Clone git repo
```
git clone https://github.com/vanphan24/cluster-peering-demo.git
```
  
# Deploy Consul on each Kubernetes cluster.
