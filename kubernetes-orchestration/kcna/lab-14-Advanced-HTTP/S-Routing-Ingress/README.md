# Lab 14: Advanced HTTP/S Routing with Ingress

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Route external HTTP/HTTPS traffic to multiple applications through a single entry point:
- Deploy two applications behind ClusterIP services
- Configure the NGINX Ingress Controller for path-based routing
- Secure the Ingress with a self-signed TLS certificate and force HTTPS
- Verify routing, check controller logs, and run a basic load test

## Environment
Ubuntu machine with a single-node Minikube cluster (Docker driver) and `kubectl`. The Ingress controller is the NGINX Ingress addon (v1.15.1). Both apps are nginx 1.21 serving a custom page from a ConfigMap.

## Task 1: Environment setup
I checked the cluster and Minikube status, then enabled the ingress addon and waited for the controller pod to become ready. Minikube also warned that Docker was at 90% disk usage, which is worth cleaning up with `docker system prune` before deploying more.

```bash
kubectl cluster-info
kubectl get nodes
minikube status
minikube addons enable ingress
kubectl get pods -n ingress-nginx
kubectl wait --namespace ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=120s
```

## Task 2: Two applications
I created a `web-apps` namespace and deployed App1 and App2. Each has 2 replicas, an HTML page in a ConfigMap mounted at `/usr/share/nginx/html`, and a ClusterIP service.

```bash
kubectl create namespace web-apps
kubectl apply -f app1-deployment.yaml
kubectl apply -f app2-deployment.yaml
kubectl get deployments,pods,svc -n web-apps
kubectl wait --for=condition=ready pod -l app=app1 -n web-apps --timeout=60s
```

## Task 3: Path-based routing
I wrote an Ingress for host `myapps.local` that sends `/app1` to app1-service, `/app2` to app2-service, and `/` to app1-service. I added the Minikube IP to `/etc/hosts` and tested with and without the `Host` header.

```bash
kubectl apply -f ingress-basic.yaml
kubectl describe ingress web-apps-ingress -n web-apps
echo "$(minikube ip) myapps.local" | sudo tee -a /etc/hosts
curl -H "Host: myapps.local" http://$(minikube ip)/app1
curl http://myapps.local/app2
```

## Task 4: TLS
I generated a self-signed certificate with `openssl`, stored it as a TLS secret, and updated the Ingress to terminate TLS and force HTTPS. After that, plain HTTP returned a `308 Permanent Redirect` to HTTPS.

```bash
openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out tls.csr -subj "/CN=myapps.local/O=myapps.local"
openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt
kubectl create secret tls myapps-tls-secret --cert=tls.crt --key=tls.key -n web-apps
kubectl apply -f ingress-tls.yaml
curl -k https://myapps.local/app1
curl -v http://myapps.local/app1
```

## Task 5: Advanced routing and logs
I applied a second Ingress with an extra `api.myapps.local` host and read the controller logs to follow requests and see why some configs were rejected.

```bash
kubectl apply -f ingress-advanced.yaml
echo "$(minikube ip) api.myapps.local" | sudo tee -a /etc/hosts
curl -k -I https://myapps.local/app1
INGRESS_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n ingress-nginx $INGRESS_POD --tail=50
```

## Task 6: Verification and load test
I wrote `test-ingress.sh` to check the redirect, both paths, the API host, the certificate subject, and the custom headers. Then I load tested both paths with ApacheBench (100 requests, concurrency 10, over HTTPS). Neither run had failed requests: about 3300 req/s for `/app1` and 2500 req/s for `/app2`, with a 99th percentile of 25 ms or less.

```bash
./test-ingress.sh
kubectl get ingress --all-namespaces
kubectl get endpoints -n web-apps
ab -n 100 -c 10 -k https://myapps.local/app1
ab -n 100 -c 10 -k https://myapps.local/app2
```

## Issues I ran into
- **YAML errors:** an indentation mistake, a selector that didn't match the template labels, and a `backbend` typo (should be `backend`) were all caught by `kubectl apply`. The error messages pointed straight to the problem.
- **`configuration-snippet` was rejected:** the admission webhook blocks snippet annotations because snippet directives are disabled on this controller. So the custom `X-Served-By` and `X-App-Version` headers never appeared.
- **Duplicate host and path:** the second Ingress was rejected because `myapps.local` and `/app1` were already defined in the first. I deleted the old Ingress and applied the new one.
- **`api.myapps.local` returned 404:** my Ingress had the host in the TLS section but no rule for it, so requests hit the default backend. The fix is to add a rule for `api.myapps.local` that routes to app2-service.
- **Certificate warning in logs:** the controller warned that the certificate relies on the legacy Common Name field. A certificate with a Subject Alternative Name (SAN) for `myapps.local` avoids this.

## What I learned
- An Ingress is only rules. The Ingress controller reads them and configures NGINX, and it sends traffic to Services, not directly to Pods.
- `pathType: Prefix` with `/` as the last path works as a catch-all for anything the other paths don't match.
- TLS termination happens at the Ingress. The secret holds the certificate and key, and the backend apps still talk plain HTTP inside the cluster.
- `force-ssl-redirect` turns HTTP requests into a 308 redirect to HTTPS.
- Two Ingresses can't claim the same host and path, and the admission webhook enforces that.
- The controller logs show both sides of a problem: which backend served each request, and why a config was rejected or a certificate failed to load.
- Self-signed certificates need `curl -k`, and modern certificates should use SANs instead of only a CN.

## Cleanup
```bash
kubectl delete ingress --all -n web-apps
kubectl delete -f app1-deployment.yaml
kubectl delete -f app2-deployment.yaml
kubectl delete secret myapps-tls-secret -n web-apps
kubectl delete namespace web-apps
sudo sed -i '/myapps.local/d' /etc/hosts
rm -f tls.key tls.csr tls.crt test-ingress.sh
```
