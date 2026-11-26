# Lab 15: Implementing Autoscaling in Kubernetes

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Make an application adjust to load automatically:
- Deploy a sample app and check that metrics are available
- Configure a Horizontal Pod Autoscaler (HPA) that scales on CPU
- Generate traffic against the app
- Install and configure a Vertical Pod Autoscaler (VPA)
- Write HPA manifests for multiple metrics and custom metrics

## Environment
Al Nafi cloud lab machine (Ubuntu) with a single-node Minikube cluster (Kubernetes v1.37.0) and `kubectl`. The app is the `hpa-example` PHP/Apache image. `metrics-server` was already installed in the cluster.

## Task 1: Horizontal Pod Autoscaler
I deployed `php-apache` with 1 replica, a CPU request of 200m and a limit of 500m, plus a ClusterIP service. I checked metrics before creating the HPA.

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl top pods
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl apply -f hpa-config.yaml
kubectl get hpa
kubectl describe hpa php-apache
```

```
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   cpu: 0%/50%   1         10        1
```

I first created the HPA with `kubectl autoscale`, then applied a YAML version that adds a `behavior` section. When idle, the pod used 1m CPU (0% of its request). `describe` showed `ScalingLimited: True / TooFewReplicas`, meaning the HPA wanted fewer than the minimum of 1 replica, so it stayed at 1.

The `behavior` rules I set:
- **Scale up:** add up to 100% or 4 pods every 15 seconds, taking whichever policy allows more (`selectPolicy: Max`), with no stabilisation delay.
- **Scale down:** remove at most 10% every 60 seconds, after a 300-second stabilisation window.

## Task 2: Load test
I started a `busybox` pod and ran `while true; do wget -q -O- http://php-apache; done`. The service answered `OK!` on every request except once, when `wget` reported `Connection refused`. I stopped the loop with Ctrl+C and the pod was deleted (`--rm`).

I did not record the HPA watch output in this session, so I am not claiming specific scale-up or scale-down numbers for this task.

## Task 3: Vertical Pod Autoscaler
The lab says to run `./hack/vpa-install.sh`, but that script does not exist in the current repository, so I looked in `hack/` and found `vpa-up.sh`. It installed the three VPA components (recommender, updater, admission controller) and created the certificate secret for the webhook.

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-up.sh
kubectl get pods -n kube-system | grep vpa
kubectl delete hpa php-apache
kubectl apply -f vpa-config.yaml
kubectl get vpa
```

I deleted the HPA first, because an HPA and a VPA should not both act on the CPU of the same Deployment. The VPA produced a recommendation:

```
NAME             MODE   CPU    MEM     PROVIDED
php-apache-vpa   Auto   100m   250Mi   True

Lower Bound:      cpu 100m   memory 250Mi
Target:           cpu 100m   memory 250Mi
Uncapped Target:  cpu 25m    memory 250Mi
Upper Bound:      cpu 807m   memory 500Mi
```

- The **uncapped target** (25m CPU) is what the recommender wanted. The **target** is 100m because my `minAllowed` raised it.
- The 250Mi memory recommendation is much more than the pod really used (9 to 12Mi). It looks like a recommender minimum rather than measured usage.
- After I started a 3-replica load generator, the upper bound for CPU dropped from 807m to 356m, but the target stayed the same. The pod's `kubectl top` reading stayed at 1m CPU, and the pod was never restarted or resized (still 200m CPU and 64Mi memory requested, 0 restarts).
- The VPA updater probably left the pod alone because there was only 1 replica, though I did not confirm that.

I then deleted that VPA and created a second one with `updateMode: "Off"`, which only gives recommendations and never changes pods. It had no recommendation yet when I checked, because it had just been created.

## Task 4: Advanced HPA manifests
I wrote `multi-metric-hpa.yaml` (CPU 60% and memory 70%, 2 to 15 replicas, slow scale-down) and `custom-metric-hpa.yaml` (CPU plus an `http_requests_per_second` pods metric). I did not apply either one. The custom-metric version would need a custom metrics API adapter, which this cluster does not have.

## Manifests
The `manifests/` folder holds the YAML files I wrote for this lab.

## Troubleshooting
- **Service rejected:** `unknown field "spec.ports[0].ports"`. I had typed `ports` inside a port entry. Fixing the typo let the service be created.
- **HPA rejected:** `unknown field "spec.behaviour"`. The Kubernetes field is spelled `behavior`, not `behaviour`.
- **`kubectl autoscale` failed twice:** once because I typed `--cpu-percent` twice, and once with `failed to create autoscaling/v2 HPA`, which was probably because the HPA already existed. The flag also prints a deprecation warning (`--cpu` is the newer form).
- **`kubectl apply` warned** that the HPA was missing the `last-applied-configuration` annotation, because I had first created it with `kubectl autoscale`. The annotation was patched automatically.
- **`vpa-install.sh: No such file or directory`.** The script was removed from the repository, so I used `vpa-up.sh`.
- **Load generator failed three times.** The first error was `no matches for kind "Deployment" in version "app/v1"` (should be `apps/v1`). The second was `unknown field "spec.selector.matchLabeles"` plus a misplaced `spec` (a typo and wrong indentation). Rewriting the file with `cat` fixed it.
- **Second VPA rejected:** `unknown field "spec.resourcePolicy.maxAllowed"`. `minAllowed` and `maxAllowed` belong inside each item under `containerPolicies`, not directly under `resourcePolicy`.
- **VPA warning:** `"Auto" mode is deprecated`. The newer modes are `Off`, `Initial`, `Recreate`, `InPlace` and `InPlaceOrRecreate`.
- **Re-applying the latest metrics-server manifest** was unnecessary, because the lab only asks for it when metrics-server is missing. It created a second metrics-server pod that stayed `0/1` Ready while the old one kept serving, and metrics kept working.
- Pasting a command with a leading `$` gave `command not found`.

## What I learned
- An HPA scales on a percentage of the pod's CPU request, so requests must be set. With a 200m request, a 50% target means about 100m per pod.
- The `behavior` section controls how fast the HPA reacts: scale up quickly, scale down slowly.
- HPA changes the number of pods, while VPA changes the size of each pod. They should not both control CPU on the same Deployment.
- A VPA in `Off` mode is a safe way to get right-sizing advice without anything restarting.
- VPA recommendations are clamped by `minAllowed` and `maxAllowed`, so compare the target with the uncapped target.
- Kubernetes rejects misspelled fields (strict decoding), which turned out to be the most common error in this lab.
- Setup scripts and defaults change between versions, so check the repository instead of trusting the lab text.
