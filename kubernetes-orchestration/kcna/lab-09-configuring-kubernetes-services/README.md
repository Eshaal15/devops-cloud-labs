# Lab 9: Configuring and Using Kubernetes Services

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Learn how Kubernetes exposes applications, and how the different Service types differ:
- **ClusterIP**: reachable only inside the cluster (the default)
- **NodePort**: reachable from outside through a fixed port on each node
- **LoadBalancer**: reachable through an external load balancer, which normally needs a cloud provider
- **Service discovery**: finding services by name through cluster DNS

## Environment
Al Nafi cloud lab machine (Ubuntu) with a Kubernetes cluster, `kubectl`, and Docker. I used an nginx deployment with 3 replicas as the test application.

## Task 1: Deployment and ClusterIP service
I deployed nginx (3 replicas, labelled `app: nginx`) and exposed it with a ClusterIP service that selects the pods by label. I then tested it from a temporary busybox pod, using the service IP, the short service name, and the full DNS name.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods -l app=nginx
kubectl apply -f nginx-clusterip-service.yaml
kubectl describe service nginx-clusterip-service
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh
# inside the pod:
wget -qO- http://nginx-clusterip-service
wget -qO- http://nginx-clusterip-service.default.svc.cluster.local
```

All three ways of reaching the service returned the nginx welcome page.

## Task 2: NodePort
I created a NodePort service on a fixed port (30080), tested it with `curl` against the node, and created a second NodePort service without choosing a port to see that Kubernetes assigns one automatically from its default range (30000 to 32767).

```bash
kubectl apply -f nginx-nodeport-service.yaml
kubectl get nodes -o wide
curl http://localhost:30080
kubectl apply -f nginx-nodeport-auto.yaml
kubectl get service nginx-nodeport-auto
```

## Task 3: LoadBalancer and MetalLB
I created a LoadBalancer service with a cloud-specific annotation. On a lab cluster with no cloud provider, there is no load balancer to give the service an external IP, so I installed MetalLB to simulate one and gave it an address pool.

```bash
kubectl apply -f nginx-loadbalancer-service.yaml
kubectl get service nginx-loadbalancer-service
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
kubectl wait --namespace metallb-system --for=condition=ready pod --selector=app=metallb --timeout=90s
kubectl apply -f metallb-config.yaml
kubectl get service nginx-loadbalancer-service
```

The address pool in the config (192.168.1.240 to 192.168.1.250) is only an example. For an address to be reachable from outside, the pool has to match the real network of the nodes.

## Task 4: DNS and endpoints
I resolved the services by name from a test pod with `nslookup`, then looked at the endpoints behind each service and checked they match the pod IPs. I scaled the deployment from 3 to 5 replicas and watched the endpoints list grow, then scaled it back.

```bash
kubectl run dns-test --image=busybox --rm -it --restart=Never -- sh
# inside the pod:
nslookup nginx-clusterip-service
kubectl get endpoints
kubectl describe endpoints nginx-clusterip-service
kubectl get pods -l app=nginx -o wide
kubectl scale deployment nginx-app --replicas=5
kubectl get endpoints nginx-clusterip-service
kubectl scale deployment nginx-app --replicas=3
```

## Troubleshooting
- I made a typo in a scale command (`deploymnet`), which kubectl rejected, and fixed it by retyping the command correctly.

## What I learned
- A Service gives a stable address and DNS name to a changing set of pods, and a label selector decides which pods it sends traffic to.
- ClusterIP is internal only, NodePort adds a port on every node, and LoadBalancer adds an external load balancer on top of that.
- A LoadBalancer needs infrastructure behind it. Without a cloud provider, it waits for an external IP, which is why tools like MetalLB exist for local clusters.
- Endpoints are the list of pod IPs behind a service, and they update automatically when the deployment scales.
- Services can be reached by short name inside the same namespace and by the full `service.namespace.svc.cluster.local` name from anywhere in the cluster.

## Cleanup
```bash
kubectl delete service nginx-clusterip-service nginx-nodeport-service nginx-nodeport-auto nginx-loadbalancer-service
kubectl delete deployment nginx-app
```
