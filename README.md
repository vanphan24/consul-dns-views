# consul-dns-views

This demo will the latest Consul DNS Views feature for Kubernetes Consul 1.20. This feature allows developers use Consul service discovery and migrate from a single-tenant to multi-tenant deployment without changing or updating their existing application code.  
This demo will deploy two Consul clusters on Kubernetes, a server cluster in the default parttion and a client cluster in a partition called 'ap1'. We will use Azure Kubernetes Service (AKS) although any Kubernetes cluster should work.
The Consul client cluster will include a Consul DNS Proxy that will proxy all Consul DNS queries and return the appropriate response. 
![alt text](https://github.com/vanphan24/consul-dns-views/blob/main/images/dns-views1.png)

![alt text](https://github.com/vanphan24/consul-dns-views/blob/main/images/dns-views-2.png)

The steps in this repo closely follow steps for setting up Admin Partitions in the Consul docs: https://developer.hashicorp.com/consul/docs/enterprise/admin-partitions#deploying-consul-with-admin-partitions-on-kubernetes

# Pre-reqs

1. You have two Kubernetes clusters available, one for Consul server and one for Consul client workload. In this demo example, we will use Azure Kubernetes Service (AKS) but it can be applied to other K8s clusters.
2. Add or update your hashicorp helm repo:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```
or
```
helm repo update hashicorp
```
3. Consul enterprise license. Admin partition is an enterprise feature.
  
4. Clone git repo
```
git clone https://github.com/vanphan24/consul-dns-views.git
```
5. Go into clone directory.
```
cd consul-dns-views
```
  
# Deploy Consul on each Kubernetes cluster.

1. Set environment variables
```
HELM_RELEASE_SERVER=server
HELM_RELEASE_CLIENT=client

export SERVER_CONTEXT=<K8s context for server>
export CLIENT_CONTEXT=<K8s context for client workload>
```

2. Add license to Kubernetes secrets on server cluster.
```
kubectl create namespace consul --context ${SERVER_CONTEXT)
kubectl create secret --context ${SERVER_CONTEXT} --namespace consul generic license --from-file=key=./path/to/license.hclic
```

3. Add license to Kubernetes secrets on client cluster.
```
kubectl create namespace consul --context ${CLIENT_CONTEXT}
kubectl create secret --context ${CLIENT_CONTEXT} --namespace consul generic license --from-file=key=./path/to/license.hclic
```

# Install the Consul server cluster

1. Set context and install Consul server cluster.

```
kubectl config use-context ${SERVER_CONTEXT}
helm install $HELM_RELEASE_SERVER hashicorp/consul -n consul --values values-server.yaml
```
Example:
```
% kubectl get pod -n consul
NAME                                                  READY   STATUS    RESTARTS      AGE
server-consul-connect-injector-5887fc6cd8-g79rp       1/1     Running   0             15m
server-consul-mesh-gateway-7f55ddf45f-vdwsn           1/1     Running   0             15m
server-consul-server-0                                1/1     Running   0             15m
server-consul-sync-catalog-856dfbcdf7-hpxdq           1/1     Running   1 (14m ago)   15m
server-consul-webhook-cert-manager-5476fd95d9-svwxp   1/1     Running   0             15m
```


2. Retreive Consul server's `consul-expose-servers` external IP address. Save this address. It will be used to configure the Consul client cluster.
```
kubectl get services --selector="app=consul,component=server" --namespace consul --output jsonpath="{range .items[*]}{@.status.loadBalancer.ingress[*].ip}{end}" --context ${SERVER_CONTEXT}
```

3. Get the Kubernetes API Server's URL for the second Kubernetes cluster. Save this URL. It will be used to configure the Consul client cluster.
```
kubectl config view --output "jsonpath={.clusters[?(@.name=='${CLIENT_CONTEXT}')].cluster.server}"
```
4.  Copy the 3 secrets (consul-ca-cert, consul-cs-kay, and consul-partition-acl-token) from the first Kubernetes cluster to the second Kubernetes cluster.
```
kubectl get secret ${HELM_RELEASE_SERVER}-consul-ca-cert --context ${SERVER_CONTEXT} -n consul --output yaml | kubectl apply -n consul --context ${CLIENT_CONTEXT} --filename -
kubectl get secret ${HELM_RELEASE_SERVER}-consul-ca-key --context ${SERVER_CONTEXT} -n consul --output yaml | kubectl apply -n consul --context ${CLIENT_CONTEXT} --filename -
kubectl get secret ${HELM_RELEASE_SERVER}-consul-partitions-acl-token --context ${SERVER_CONTEXT} -n consul --output yaml | kubectl apply -n consul --context ${CLIENT_CONTEXT} --filename -
```

5. Edit the 'values-client.yaml' file with the `consul-expose-servers` external IP address and the Kubernetes API Server's URL steps 2 and 3.

```
.
.
.
externalServers:
  enabled: true
  hosts: [48.216.183.98]   # Update this IP address with your consul-expose-servers
  tlsServerName: server.dc1.consul
  k8sAuthMethodHost: https://dc4-dns-sgf78yij.hcp.eastus.azmk8s.io:443  # Update this URL with the second Kubernetes cluster's API Server's URL 
.
.
.

```

5. Deploy Consul client cluster onto second Kubernetes cluster.
   Note, this is using the Consul-k8s chart that is cloned: `./consul-k8s/charts/consul/`. You can also just refer to the released chart `hashicorp/consul` instead when Consul GAs.  
```
git clone https://github.com/hashicorp/consul-k8s.git
kubectl config use-context ${CLIENT_CONTEXT}
helm install ${HELM_RELEASE_CLIENT}  ./consul-k8s/charts/consul/ -n consul --values values-client.yaml
```
Example:
```
% kubectl get pod -n consul
NAME                                                  READY   STATUS    RESTARTS   AGE
client-consul-connect-injector-f9c6f8784-ll84q        1/1     Running   0          5m30s
client-consul-dns-proxy-64555964cd-t56wh              1/1     Running   0          5m30s
client-consul-dns-proxy-64555964cd-xmzx4              1/1     Running   0          5m30s
client-consul-mesh-gateway-56855869c6-cxpvn           1/1     Running   0          5m30s
client-consul-sync-catalog-898c94994-xxfkv            1/1     Running   0          5m30s
client-consul-webhook-cert-manager-75546cff95-kxtnf   1/1     Running   0          5m30s
```

6. Deploy sample Fake Service application
```
kubectl apply -f ./apps --context ${CLIENT_CONTEXT}
```

7.  If you want to view the Consul UI, retreieve the Consul bootstrap token on the Consul server. And retrieve the consul server External-IP address. 
```
kubectl get secret --namespace consul --context ${SERVER_CONTEXT} --template "{{ .data.token | base64decode }}" ${HELM_RELEASE_SERVER}-consul-bootstrap-acl-token
```
```
kubectl get services ${HELM_RELEASE_SERVER}-consul-ui -n consul --context ${SERVER_CONTEXT}
```
Then open browser using the servers consul UI External-IP. Log on with the bootstrap token. Navigate to the ap1 parttion and notice the Fake Service application that includes the frontend and api service.

8. Retreieve the Consul DNS proxy's IP address. Save this IP address because it will be used in the next step to configure the configmap.

```
kubectl get svc client-consul-dns-proxy -n consul -o jsonpath='{.spec.clusterIP}' --context ${CLIENT_CONTEXT}
```

9. Edit the `coredns-custom-configmap.yaml` configmap file with the external IP address from previous step.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  custom.server: |
    consul:53 {
      errors
      cache 30
      forward . 10.0.135.245 # Update this IP address with your Consul DNS proxy's external IP address.
      reload
    }
```

10. Apply new coredns config map file.

```
kubectl apply -f coredns-custom.yaml --context ${CLIENT_CONTEXT}
```

# Test out Consul DNS resolution

## Option A

1. Log into the frontend service's container and run `nslookup` of api service to confirm Consul DNS resolution.

```
kubectl exec -it <frontend-pod-name> --context ${CLIENT_CONTEXT} -- sh
```
```
nslookup api.service.consul
```
Example:
```
# nslookup api.service.consul
Server:		10.0.0.10
Address:	10.0.0.10:53


Name:	api.service.consul
Address: 10.224.0.12
```
 
The resolved IP address should match the `api` pod's IP address


## Option B

1. Check Frontend service to see if it can resolve the API service's IP address.

```
kubectl  port-forward svc/frontend 9090:9090 --context ${CLIENT_CONTEXT}
```

2. Open browser to `http://localhost:9090/ui`

3. Confirm boxes are not red, and DNS name is resolved.

![alt text](https://github.com/vanphan24/consul-dns-views/blob/main/images/dns-views-fake-service.png)

