# Lab 10: Deploying a Stand-Alone Application in Kubernetes

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Deploy an application from a manifest to a running, monitored service:
- Write a Deployment manifest with several replicas
- Expose it outside the cluster with a NodePort service
- Read logs and check resource usage
- Customise the app with a ConfigMap

## Environment
Al Nafi cloud lab machine (Ubuntu) with a single-node Minikube cluster and `kubectl`. The application is nginx 1.25.3.

## Task 1: Deployment from a manifest
I wrote a Deployment with 3 replicas, resource requests and limits, and a label selector. The Deployment already existed from an earlier session, so `kubectl apply` reported it as `unchanged`, which showed that applying a manifest only changes what differs.

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get deployments
kubectl get pods -l app=nginx-standalone
kubectl get replicasets -l app=nginx-standalone
```

```
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
nginx-standalone-app   3/3     3            3           28d

NAME                              DESIRED   CURRENT   READY   AGE
nginx-standalone-app-5ff48d954c   3         3         3       28d
```

## Task 2: NodePort service
I exposed the Deployment with a NodePort service on port 30080 and tested it from the machine using the Minikube IP.

```bash
kubectl apply -f nginx-service.yaml
kubectl describe service nginx-standalone-service
curl http://$(minikube ip):30080
kubectl get endpoints nginx-standalone-service
kubectl get pods -l app=nginx-standalone -o wide
```

```
NAME                       TYPE       CLUSTER-IP      PORT(S)
nginx-standalone-service   NodePort   10.109.46.186   80:30080/TCP
```

`curl` returned the default "Welcome to nginx!" page. The service endpoints (`10.244.0.6`, `10.244.0.8`, `10.244.0.9`, all on port 80) matched the three pod IPs exactly, which confirms the label selector found the right pods.

## Task 3: Logs, metrics and a custom page
**Logs.** `kubectl logs -l app=nginx-standalone` showed the logs of all three pods together: nginx start-up messages and access log lines. I generated traffic with `curl` and then used `--since=10m` to show only the recent requests. Opening the page in a browser also logged a `favicon.ico` 404 error, because the default nginx page has no favicon.

**Resource usage.** Metrics were available in this cluster:

```
NAME                                    CPU(cores)   MEMORY(bytes)
nginx-standalone-app-5ff48d954c-2xlkk   0m           4Mi
nginx-standalone-app-5ff48d954c-q7lxj   1m           6Mi
nginx-standalone-app-5ff48d954c-qtl2c   1m           4Mi

NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
minikube   513m         12%      973Mi           12%
```

The pods use far less than their requests (250m CPU, 64Mi memory), so the limits leave a lot of headroom. `kubectl describe pod` showed the pod runs in the Burstable QoS class, and that the container had restarted 5 times in its 28 days (a previous run had ended with exit code 255). `kubectl rollout status` reported a successful rollout, and the events query for the Deployment returned nothing, meaning no warnings.

**Custom page.** I put a custom HTML page in a ConfigMap, mounted it over `/usr/share/nginx/html`, and applied the updated Deployment. The rollout waited while old replicas terminated, then completed, and `curl` returned my custom page.

```bash
kubectl create configmap nginx-custom-html --from-file=index.html=custom-index.html
kubectl apply -f nginx-deployment-updated.yaml
kubectl rollout status deployment/nginx-standalone-app
curl http://$(minikube ip):30080
```

## Troubleshooting
- `kubectl endpoints ...` failed with "unknown command". The correct form is `kubectl get endpoints <name>`. It also printed a deprecation warning that `Endpoints` is being replaced by `EndpointSlice` in newer Kubernetes versions.
- A misspelled option (`--feild-selector`) was rejected with "unknown flag". Retyping it as `--field-selector` fixed it.
- In the page source, the script line `document.getElementById('hostname') = window.location.hostname;` assigns to the result of a function call, which doesn't work in JavaScript. The fix would be `.textContent = window.location.hostname`, so the "Hostname" field on the page stays on "Loading...".

## What I learned
- A Deployment manages a ReplicaSet, which manages the Pods, and a Service finds those pods by matching labels.
- `kubectl apply` is declarative: it only changes what differs, which is why the unchanged Deployment was left alone.
- Requests tell the scheduler how much a pod needs, and limits cap how much it can use.
- A ConfigMap changes what an app serves without rebuilding its image. Mounting it at `/usr/share/nginx/html` replaces the whole default folder with just my file.
- Changing the pod template triggers a rolling update, so the app stays available while pods are replaced.
- Logs, events, and `kubectl top` show different things: what the app says, what Kubernetes did, and how much it uses.
