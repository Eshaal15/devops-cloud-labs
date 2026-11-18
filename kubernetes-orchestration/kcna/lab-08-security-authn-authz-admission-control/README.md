# Lab 8: Implementing Security with Authentication, Authorization, and Admission Control

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Learn how Kubernetes decides who can do what, and how it limits what can be created:
- **Authentication**: proving who you are (service accounts and tokens)
- **Authorization**: what you are allowed to do (RBAC roles and bindings)
- **Admission control**: rules applied before a resource is created (resource quotas)
- **Network policies**: which pods may talk to each other

## Environment
Al Nafi cloud lab machine (Ubuntu) with a single-node Kubernetes cluster and `kubectl`. All work was done in a dedicated `security-lab` namespace.

## Task 1: Explore the existing security setup
I looked at the current context, the cluster, the default service accounts, and the built-in roles and bindings:

```bash
kubectl config current-context
kubectl get serviceaccounts --all-namespaces
kubectl describe clusterrole view
kubectl get rolebindings --all-namespaces
kubectl get clusterrolebindings
```

## Task 2: Service accounts and RBAC
I created a namespace and three service accounts, each with a different level of access:

| Service account | Access | How |
|-----------------|--------|-----|
| `developer-sa` | Create, update, and delete pods, services, configmaps, secrets, and deployments in the namespace | Custom `Role` + `RoleBinding` |
| `viewer-sa` | Read-only (get and list) on pods, services, configmaps, and deployments | Custom `Role` + `RoleBinding` |
| `admin-sa` | Everything, cluster-wide | `ClusterRoleBinding` to `cluster-admin` |

```bash
kubectl create namespace security-lab
kubectl create serviceaccount developer-sa -n security-lab
kubectl apply -f developer-role.yaml -f developer-rolebinding.yaml
kubectl apply -f viewer-role.yaml -f viewer-rolebinding.yaml
kubectl apply -f admin-clusterrolebinding.yaml
```

Result in the namespace:

```
NAME                          AGE
serviceaccount/admin-sa       21h
serviceaccount/default        22h
serviceaccount/developer-sa   21h
serviceaccount/viewer-sa      21h

NAME                  ROLE                  
developer-binding     Role/developer-role   
viewer-binding        Role/viewer-role      
```

## Task 3: Test access control
I deployed an nginx test app and generated short-lived tokens with `kubectl create token`, then ran commands as each service account by passing `--token`:

```bash
DEVELOPER_TOKEN=$(kubectl create token developer-sa -n security-lab)
kubectl --token=$DEVELOPER_TOKEN create configmap test-config --from-literal=key1=value1 -n security-lab
kubectl --token=$DEVELOPER_TOKEN scale deployment test-app --replicas=3 -n security-lab
kubectl --token=$VIEWER_TOKEN create configmap viewer-test --from-literal=key1=value1 -n security-lab
kubectl --token=$ADMIN_TOKEN get nodes
```

Results:
- **Developer:** could list pods, create a configmap, and scale the deployment to 3 replicas.
- **Viewer:** could list pods and deployments, but creating a configmap and deleting a pod were denied with a Forbidden error.
- **Admin:** could run cluster-wide commands such as listing nodes and create resources in another namespace.

## Task 4: Admission control with a resource quota
I applied a `ResourceQuota` that caps CPU, memory, pods, services, and configmaps in the namespace. Then I deployed an app (`resource-heavy-app`) with 5 replicas, each requesting 1 CPU and 1 GiB of memory.

```bash
kubectl apply -f resource-quota.yaml
kubectl apply -f resource-heavy-deployment.yaml
kubectl describe resourcequota security-lab-quota -n security-lab
```

Result: only **one** of the five replicas was created. The quota allows 2 CPUs of requests, and most of that was already used, so the other replicas were rejected.

```
Resource         Used    Hard
--------         ----    ----
configmaps       5       10
limits.cpu       3700m   4
limits.memory    2560Mi  4Gi
pods             5       10
requests.cpu     1850m   2
requests.memory  1280Mi  2Gi
services         0       5
```

The numbers add up: 3 `test-app` pods (250m each) + 1 heavy pod (1 CPU) + the test client (100m) = 1850m of CPU requests. A second heavy pod would need another 1 CPU, which goes over the 2 CPU limit.

## Task 5: Network policies and final audit
I applied a default-deny ingress policy and a second policy that lets only pods labelled `access: allowed` reach the test app on port 80. Then I tested from two client pods:

```bash
kubectl exec test-client -n security-lab -- wget -qO- --timeout=5 http://$TEST_APP_IP
kubectl exec unauthorized-client -n security-lab -- timeout 10 wget -qO- http://$TEST_APP_IP
```

- The pod with the `access: allowed` label (`test-client`) reached the app.
- The unlabelled pod (`unauthorized-client`) was expected to be blocked.

```
NAME                      POD-SELECTOR
allow-test-app-ingress    app=test-app
deny-all-ingress          <none>
```

I finished with an audit of every service account, role, binding, quota, and policy created in the namespace.

## Troubleshooting
- My first `test-client` manifest failed because the `image` key was not indented under the container, so I rewrote it. YAML indentation matters, so I now check files with `cat` before applying.
- Pasting a multi-line YAML file on one line gave a broken manifest. Writing it with a proper `cat > file <<EOF` block fixed it.

## What I learned
- A service account is an identity for apps, and RBAC decides what that identity can do. Giving each account only the permissions it needs is the principle of least privilege.
- A `Role` works inside one namespace, while a `ClusterRole` works cluster-wide. Binding `cluster-admin` is powerful and should be rare.
- A request can pass authentication and still be denied at the authorization step.
- Resource quotas are enforced by admission control. The API server rejects a request that breaks a rule before the resource exists.
- Network policies only work if the cluster's network plugin supports them. A default-deny policy plus specific allow rules (matched by labels) is a common pattern.
- Tokens are credentials. I used short-lived tokens, and I never commit them or kubeconfig files to GitHub.

## Cleanup
```bash
kubectl delete namespace security-lab
kubectl delete clusterrolebinding admin-binding
kubectl config set-context --current --namespace=default
```
