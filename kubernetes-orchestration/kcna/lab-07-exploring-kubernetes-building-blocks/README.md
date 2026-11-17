\# Lab 7: Exploring Kubernetes Building Blocks



Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.



\## Goal

Learn the three core Kubernetes building blocks and how they differ:

\- \*\*Pod\*\*: the smallest deployable unit

\- \*\*Deployment\*\*: keeps a set number of Pods running and makes scaling easy

\- \*\*ConfigMap\*\*: keeps configuration outside the container image



\## Environment

\- Al Nafi cloud lab machine (Ubuntu) with a single-node Minikube cluster

\- Tools: `kubectl`, `nano`



\## Task 1: Deploy a Pod from a YAML manifest

I wrote a manifest for an nginx Pod with labels, a container port, and resource requests and limits. Requests are the minimum a container is guaranteed, and limits are the maximum it can use.



```bash

kubectl apply -f simple-pod.yaml

kubectl get pods

kubectl describe pod nginx-pod

kubectl logs nginx-pod

kubectl get pod nginx-pod -o wide

kubectl exec -it nginx-pod -- curl localhost:80

```



Result: the Pod reached `Running`, and curl inside the Pod returned the default nginx page.



```

NAME                                    READY   STATUS    RESTARTS   AGE

app-pod                                 1/1     Running   0          34m

nginx-deployment-7dcfcf54bd-4lw5z       1/1     Running   0          65m

nginx-deployment-7dcfcf54bd-l254q       1/1     Running   0          50m

nginx-deployment-7dcfcf54bd-z5ghm       1/1     Running   0          65m

nginx-pod                               1/1     Running   0          72m

```



\## Task 2: Run and scale with a Deployment

I created a Deployment with 2 replicas, then scaled it two ways:



```bash

kubectl apply -f nginx-deployment.yaml

kubectl get deployments

kubectl get pods -l app=nginx



\# imperative scaling

kubectl scale deployment nginx-deployment --replicas=4



\# declarative scaling: change replicas in the YAML, then

kubectl apply -f nginx-deployment.yaml

```



Result: the Deployment scaled from 2 to 4 replicas, then to 3 after the YAML change. The label selector `app=nginx` also matched `nginx-pod`, which carries the same label.



```

NAME                                READY   STATUS    RESTARTS   AGE

nginx-deployment-7dcfcf54bd-4lw5z   1/1     Running   0          65m

nginx-deployment-7dcfcf54bd-l254q   1/1     Running   0          50m

nginx-deployment-7dcfcf54bd-z5ghm   1/1     Running   0          65m

nginx-pod                           1/1     Running   0          72m

```



\## Task 3: Configuration with a ConfigMap

I created a ConfigMap with app settings (database host, port, name, log level, max connections, app version) and a Pod that loads all of them as environment variables using `envFrom`. I also passed in the Pod's own name with `fieldRef`.



```bash

kubectl apply -f app-config.yaml

kubectl describe configmap app-config

kubectl apply -f pod-with-config.yaml

kubectl logs app-pod

kubectl exec -it app-pod -- sh   # then: env | sort

```



I then changed values in the ConfigMap with `kubectl patch` (log level to DEBUG, app version to 1.2.4), deleted the Pod, and recreated it. The new Pod picked up the updated values:



```

Starting application...

HOSTNAME=app-pod

database\_port=3306

database\_name=myapp

app\_version=1.2.4

POD\_NAME=app-pod

log\_level=DEBUG

max\_connections=100

database\_host=mysql.example.com

```



\## What I learned

\- A bare Pod isn't replaced if it dies, but a Deployment keeps the desired number of replicas running.

\- `kubectl scale` is quick, but editing the YAML and running `kubectl apply` keeps the file as the single source of truth.

\- A label selector matches every Pod with that label, even ones the Deployment doesn't manage. That's why `nginx-pod` showed up next to the Deployment's Pods.

\- Changing a ConfigMap does not update environment variables in a running Pod. The Pod has to be recreated to pick up new values.

\- Resource requests and limits tell the scheduler how much a Pod needs and stop one container from using all the resources on a node.



\## Troubleshooting notes

\- If a Pod stays in `Pending`, check `kubectl describe pod <name>` for events such as image pull or resource problems.

\- If a ConfigMap doesn't load, check the ConfigMap name in the Pod spec and the YAML indentation.



\## Cleanup

```bash

kubectl delete pod nginx-pod app-pod

kubectl delete deployment nginx-deployment

kubectl delete configmap app-config

