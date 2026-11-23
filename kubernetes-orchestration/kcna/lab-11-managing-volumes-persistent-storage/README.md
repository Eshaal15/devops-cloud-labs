# Lab 11: Managing Kubernetes Volumes and Persistent Storage

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal
Learn how Kubernetes keeps data alive beyond a pod's lifetime:
- **PersistentVolume (PV):** a piece of storage in the cluster
- **PersistentVolumeClaim (PVC):** a request for storage by a user or application
- **Binding:** how a claim finds a matching volume
- **Persistence:** proving data survives pod deletion, recreation, and scaling

## Environment
Al Nafi cloud lab machine (Ubuntu) with a single-node Minikube cluster and `kubectl`. All work was done in a dedicated `storage-lab` namespace. The cluster's default storage class is `standard` (the Minikube hostpath provisioner), so this lab uses a custom `manual` class with volumes I created myself.

## Task 1: Cluster check and namespace
I checked the cluster, nodes, and storage classes, then created the `storage-lab` namespace and made it the default for my session.

```bash
kubectl cluster-info
kubectl get nodes
kubectl get storageclass
kubectl create namespace storage-lab
kubectl config set-context --current --namespace=storage-lab
```

## Task 2: PersistentVolume and PersistentVolumeClaim
I created a 1 Gi PV on the node's local disk (`hostPath`) with the `manual` storage class and a `Retain` reclaim policy, then a PVC requesting 500 Mi with the same class and access mode.

```bash
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml
kubectl get pv,pvc
kubectl describe pvc lab-pvc
```

The claim bound to the volume. The claim asked for 500 Mi but the volume is 1 Gi, and a PV can only be bound to one claim, so the claim holds the whole 1 Gi volume.

## Task 3: An application that writes to the volume
I deployed a busybox container that appends a timestamp to `/data/timestamps.log` every 30 seconds, with `/data` mounted from the PVC and with CPU and memory requests and limits. I checked that the log was growing.

```bash
kubectl apply -f storage-app-deployment.yaml
kubectl wait --for=condition=Ready pod -l app=storage-app --timeout=60s
POD_NAME=$(kubectl get pods -l app=storage-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- cat /data/timestamps.log
```

## Task 4: Proving the data persists
I counted the log lines, deleted the whole Deployment (which deletes the pod), recreated it, and checked the new pod still had the old lines. I then wrote a test file, scaled the Deployment to zero and back to one, and confirmed the file survived that too.

```bash
kubectl delete deployment storage-app
kubectl apply -f storage-app-deployment.yaml
kubectl exec $NEW_POD_NAME -- head -5 /data/timestamps.log
kubectl scale deployment storage-app --replicas=0
kubectl scale deployment storage-app --replicas=1
kubectl exec $FINAL_POD_NAME -- cat /data/test-file.txt
```

The data lives in the volume, not in the container, so it survives anything that replaces the pod.

## Task 5: Monitoring storage
I checked disk usage inside the pod with `df -h /data`, looked at PV and PVC details and events, and wrote a small script that prints the PVs, PVCs, usage inside the pod, and recent storage events.

```bash
kubectl exec $FINAL_POD_NAME -- df -h /data
kubectl get pv,pvc
kubectl describe pv lab-pv
./monitor-storage.sh
```

## Task 6: Multiple volumes
I created a second 2 Gi PV and a second PVC (requesting 1 Gi), then deployed an app that mounts two claims at once (`/data1` and `/data2`) and writes a log to each.

```bash
kubectl apply -f additional-pv.yaml -f additional-pvc.yaml
kubectl apply -f multi-volume-app.yaml
kubectl exec $MULTI_POD -- ls -la /data1 /data2
```

The first claim was already mounted by the first app and was mounted again here. That works because `ReadWriteOnce` limits a volume to one **node**, not one pod, and this is a single-node cluster.

## Task 7: Troubleshooting a claim that can't bind
I created a claim requesting 10 Gi, which is larger than any available volume. It stayed `Pending`, and `kubectl describe` and the events explained why: no volume matched the request.

```bash
kubectl apply -f problematic-pvc.yaml
kubectl get pvc large-pvc
kubectl describe pvc large-pvc
```

## What I learned
- A PV is the storage, a PVC is the request, and the cluster binds them by matching storage class, access mode, and size.
- Binding is one-to-one, so a small claim can hold a much larger volume.
- Data in a PV outlives pods, Deployments, and scaling to zero, because the volume is separate from the container.
- `Retain` keeps the data when the claim or volume object is deleted, so the files stay on the node's disk until an administrator removes them.
- `ReadWriteOnce` means one node, not one pod.
- A `Pending` claim is diagnosed with `kubectl describe pvc` and the events, which show whether the problem is the storage class, access mode, or size.

## Cleanup
```bash
./cleanup-storage.sh
```
