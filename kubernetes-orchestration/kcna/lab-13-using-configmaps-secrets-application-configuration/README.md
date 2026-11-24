# Lab 13: Using ConfigMaps and Secrets for Application Configuration

Part of my Kubernetes and Cloud Native Associate (KCNA) practice labs in the Al Razzaq program at Al Nafi.

## Goal

Keep application configuration out of the container image and feed it to Pods in different ways:

- Create ConfigMaps for non-sensitive settings (from literals and from a file)
- Create Secrets for sensitive data (from literals and from files)
- Pass both to Pods as environment variables
- Mount both into Pods as files
- Use them in a Deployment, then update them and see what changes in running Pods

## Environment

Al Nafi cloud lab machine (Ubuntu) with a single-node Minikube cluster and kubectl. All resources were created in the `multi-tier-app` namespace, next to my earlier frontend, backend and database deployments. The test Pods use `nginx:1.21`.

## Task 1: ConfigMaps as environment variables

I created a ConfigMap from four literal values, then a second one from a properties file.

```bash
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=mysql-service \
  --from-literal=DATABASE_PORT=3306 \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info
kubectl describe configmap app-config

kubectl create configmap app-properties --from-file=app.properties
kubectl get configmap app-properties -o yaml
```

`app-config` ended up with 4 keys, one per literal. `app-properties` has a single key called `app.properties`, and its value is the whole file. So `--from-literal` gives one key per value, while `--from-file` gives one key per file.

Then I created a Pod (`app-pod-env`) that loads every key from `app-config` with `envFrom`, and also maps `APP_ENV` to a new variable called `CUSTOM_MESSAGE` with `configMapKeyRef`.

```bash
kubectl apply -f pod-with-configmap.yaml
kubectl exec app-pod-env -- env | grep -E "(DATABASE|APP|LOG)"
```

```
DATABASE_HOST=mysql-service
DATABASE_PORT=3306
LOG_LEVEL=info
APP_ENV=production
```

The grep also printed a lot of `DATABASE_SERVICE_*` variables (for example `DATABASE_SERVICE_PORT=tcp://10.106.82.111:3306`). Those did not come from my ConfigMap. Kubernetes injects environment variables for every Service that already exists in the namespace, and my earlier database service matched the pattern. `CUSTOM_MESSAGE` did not show up because its name does not match the grep pattern.

## Task 2: Secrets

I created a Secret with a username, a password and a root password.

```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123 \
  --from-literal=root-password=rootsecret456
kubectl describe secret db-credentials
kubectl get secret db-credentials -o yaml
```

`describe` only shows the size of each value (`password: 14 bytes`), so it does not leak anything. `-o yaml` shows the values base64 encoded, for example `password: c3VwZXJzZWNyZXQxMjM=`. Anyone who can read the Secret can decode that with `base64 -d`, so base64 is encoding, not encryption.

I also made a Secret from files and deleted the local files straight after:

```bash
echo -n 'admin' > username.txt
echo -n 'supersecret123' > password.txt
kubectl create secret generic file-credentials --from-file=username.txt --from-file=password.txt
rm username.txt password.txt
```

With `--from-file` the key names are the file names (`username.txt`, `password.txt`), not `username` and `password`. I used `echo -n` so no newline got stored inside the values.

The Pod `app-pod-secret` reads each key with `secretKeyRef`:

```bash
kubectl apply -f pod-with-secret.yaml
kubectl exec app-pod-secret -- env | grep DB_
```

```
DB_USERNAME=admin
DB_PASSWORD=supersecret123
DB_ROOT_PASSWORD=rootsecret456
```

The values are decoded back to plain text inside the container, which is what the application needs. It also means anyone with `exec` access to the Pod can read them.

## Task 3: Mounting ConfigMaps and Secrets as files

`app-pod-volumes` mounts three volumes: `app-config` at `/etc/config`, `db-credentials` at `/etc/secrets` (with `defaultMode: 0400`), and `app-properties` at `/etc/properties`.

```bash
kubectl apply -f pod-with-volumes.yaml
kubectl wait --for=condition=Ready pod/app-pod-volumes --timeout=60s
kubectl exec app-pod-volumes -- ls -la /etc/config/
kubectl exec app-pod-volumes -- cat /etc/properties/app.properties
```

Listing `/etc/config` showed that every key is a symlink, for example `APP_ENV -> ..data/APP_ENV`. `..data` is itself a symlink to a timestamped folder. Kubernetes updates a mounted ConfigMap by building a new timestamped folder and switching the `..data` link to it, so a reader never sees a half-written file.

`/etc/properties/app.properties` showed the full six-line properties file exactly as I wrote it.

Then I used both in a Deployment (`web-app-deployment`, 2 replicas): `envFrom` for the whole ConfigMap, `secretKeyRef` for `DB_USERNAME` and `DB_PASSWORD`, and both volumes mounted under `/usr/share/nginx/html`.

```bash
kubectl apply -f deployment-with-config.yaml
kubectl get deployments
kubectl get pods -l app=web-app
POD_NAME=$(kubectl get pods -l app=web-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD_NAME -- env | grep -E "(DATABASE|APP|LOG|DB_)"
```

```
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
web-app-deployment    2/2     2            2           24s
```

Both replicas were Running, and the environment check showed `APP_ENV`, `DATABASE_HOST`, `DATABASE_PORT`, `LOG_LEVEL`, `DB_USERNAME` and `DB_PASSWORD` set correctly.

## Task 4: Updating a ConfigMap and a Secret

I patched `app-config` to change `LOG_LEVEL` and add a new key:

```bash
kubectl patch configmap app-config --patch '{"data": {"LOG_LEVEL": "debug","NEW_FEATURE":"enabled"}}'
kubectl describe configmap app-config
kubectl exec app-pod-volumes -- cat /etc/config/LOG_LEVEL
```

`describe` showed `LOG_LEVEL: debug` and the new `NEW_FEATURE` key (5 keys now). The mounted file `/etc/config/LOG_LEVEL` returned `debug` without restarting the Pod, so volume mounts pick up ConfigMap changes by themselves after the kubelet syncs. Environment variables do not work this way: they are set when the container starts, so `app-pod-env` and the Deployment pods would need a restart to see `debug`.

`cat /etc/config/NEW_FEATURE` printed nothing the first time I ran it, so I need to run it again after waiting a bit longer for the sync.

For the Secret, I added a new key with `patch`. Secret values have to be base64 encoded in the patch, so I encoded it inline:

```bash
kubectl patch secret db-credentials --patch '{"data":{"api-key":"'$(echo -n 'new-api-key-123' | base64)'"}}'
kubectl describe secret db-credentials
```

`describe` now lists `api-key: 15 bytes` next to the other three keys.

## Task 5: Overview and cleanup

Final state of the namespace after the lab:

- ConfigMaps: `app-config` (5 keys), `app-properties` (1 key), plus my earlier `backend-config`, `database-config`, `frontend-config`, `frontend-html` and `nginx-config`
- Secrets: `db-credentials`, `file-credentials`, plus the older `database-secret`
- Pods: `app-pod-env`, `app-pod-secret`, `app-pod-volumes` and the two `web-app-deployment` pods, all Running
- Deployments: `web-app-deployment` 2/2 next to the backend, database and frontend ones

I left these running for now. The cleanup commands from the lab only delete my new resources (the three Pods, `web-app-deployment`, `app-config`, `app-properties`, `db-credentials`, `file-credentials`) and not the earlier multi-tier app.

## Troubleshooting

- **Typos in field names.** Four applies failed with `strict decoding error: unknown field ...`: `secretFrom` (should be `secretKeyRef`), `restartPoilcy` (three times, should be `restartPolicy`) and `secertKeyRef` (should be `secretKeyRef`). The API server rejects fields it does not know, and the error names the exact path, such as `spec.containers[0].env[2].valueFrom.secretFrom`, so it is easy to find the line in the YAML.
- **Garbled kubectl command.** While typing the `file-credentials` command a stray `;` split it into broken pieces. I ran it again from a clean line and it worked.
- **Broken `$(...)` command.** I missed the closing `'` and `)` in the `jsonpath` expression, so the shell waited for more input. I pressed Ctrl+C and retyped it properly.
- **Wrong Pod name.** `kubectl exec app-pod-volume` failed with `NotFound` because the Pod is called `app-pod-volumes`.
- **Stray `;ls`.** A leading `;` caused `syntax error near unexpected token`.

## What I learned

- A ConfigMap holds non-sensitive settings and a Secret holds sensitive ones. Both can reach a Pod as environment variables or as mounted files.
- Secrets are only base64 encoded by default, not encrypted. Access should be limited with RBAC, and `exec` plus `env` shows the plain values.
- `envFrom` loads all keys at once, while `configMapKeyRef` and `secretKeyRef` pick one key and let me rename it.
- `--from-literal` makes one key per value, and `--from-file` makes one key per file, named after the file.
- Mounted ConfigMaps and Secrets update on their own (through the `..data` symlink swap), but environment variables only change when the Pod restarts.
- `defaultMode: 0400` makes the mounted Secret files readable only by their owner.
- Kubernetes injects environment variables for existing Services into new Pods, which explains the extra `DATABASE_SERVICE_*` entries.
- `kubectl apply` validates the manifest strictly, so field-name typos fail immediately instead of being ignored.
