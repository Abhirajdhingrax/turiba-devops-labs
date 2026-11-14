# Lab 8 · Run the whole app

GitHub user: Abhirajdhingrax · Fallback image used: yes · Fallback Mission 7 files used: no

## 1 · Database with storage

- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted yes because the data folder of PostgreSQL is on the PVC db-data, and a new Pod of the Deployment mounts the same claim. What happened: the new db Pod had a new name and a few seconds' age, and GET /api/todos still listed my todo.
- 🔮 Right after the first apply, I predicted the PVC would be Pending, because kind's standard class binds only when a Pod uses the claim (WaitForFirstConsumer). It was Pending at that moment, and Bound once the db Pod had started.

```text
$ kubectl -n course get pvc,pv
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-dd1ddf06-010f-4f69-a552-b5ebefab0d34   1Gi        RWO            standard       <unset>                 96s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-dd1ddf06-010f-4f69-a552-b5ebefab0d34   1Gi        RWO            Delete           Bound    course/db-data   standard       <unset>                          80s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8080/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-14T18:33:40.962Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-qcqlm   1/1     Running   0          96s

$ kubectl -n course delete pod db-84f4dd66bb-qcqlm
pod "db-84f4dd66bb-qcqlm" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-wpqp8   1/1     Running   0          8s

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-14T18:33:40.962Z"}]
```

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
Found 3 pods, using pod/api-75c8684f64-kf8q8
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
Found 3 pods, using pod/api-75c8684f64-kf8q8
course-api 0.1.0 starting (Node.js v24.22.0)
API listening on port 5000
Database ready
Database connection lost: terminating connection due to administrator command
```

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted not in plain text because a Secret stores its values base64-encoded. What I saw, in words (don't paste the value): the output has a data section with a password key, and its value is a base64 string, not the readable password. Base64 is only an encoding, so anyone who may read the Secret can decode it; it is not encryption.

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted they get traffic, because the readiness probe still asks /healthz and passes, but they are restarted again and again, because the liveness probe gets 404 three times in a row and the kubelet kills the container. What happened: the output below shows the new Pods READY 1/1 while the RESTARTS column grows, and the Events show Liveness probe failed with statuscode 404 and the container being restarted.

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                   READY   STATUS    RESTARTS      AGE
api-84d76fb6c8-8wpq2   1/1     Running   1 (36s ago)   76s
api-84d76fb6c8-jt9r4   1/1     Running   1 (38s ago)   78s
api-84d76fb6c8-mbm92   1/1     Running   2 (10s ago)   80s

$ kubectl -n course describe pod api-84d76fb6c8-8wpq2         (only the Events)
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  77s                default-scheduler  Successfully assigned course/api-84d76fb6c8-8wpq2 to kind-control-plane
  Normal   Pulled     76s                kubelet            spec.initContainers{wait-for-db}: Container image "postgres:17-alpine" already present on machine and can be accessed by the pod
  Normal   Created    76s                kubelet            spec.initContainers{wait-for-db}: Container created
  Normal   Started    76s                kubelet            spec.initContainers{wait-for-db}: Container started
  Normal   Pulled     37s (x2 over 76s)  kubelet            spec.containers{api}: Container image "ghcr.io/martinsturiba/turiba-devops-labs/api:0.2.0" already present on machine and can be accessed by the pod
  Normal   Created    37s (x2 over 76s)  kubelet            spec.containers{api}: Container created
  Normal   Killing    37s                kubelet            spec.containers{api}: Container api failed liveness probe, will be restarted
  Normal   Started    36s (x2 over 76s)  kubelet            spec.containers{api}: Container started
  Warning  Unhealthy  36s (x2 over 76s)  kubelet            spec.containers{api}: Readiness probe failed: Get "http://10.244.0.12:5000/healthz": dial tcp 10.244.0.12:5000: connect: connection refused
  Warning  Unhealthy  7s (x5 over 57s)   kubelet            spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

- How I made Git and the cluster agree again afterwards: git restore k8s/ threw away the uncommitted /nope change, then kubectl apply -f k8s/ rolled the API back to the file in Git, and git diff was empty afterwards.

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: the new api Pods would get stuck before the app starts, because the init container is the first thing that runs and it depends on DB_HOST from the ConfigMap. The old Pods from main kept serving during the broken rollout.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | The new api Pod stayed in Init:0/1: DB_HOST had become database, and no Service has that name | kubectl -n course logs <pod> -c wait-for-db: database:5432 - no response | 54a4c17 fix(k8s): point DB_HOST back at the db Service |
| 2 | After fix 1 and a rollout restart, the new Pod showed CreateContainerConfigError: the secretKeyRef key had become pasword | kubectl -n course describe pod <pod>: Warning  Failed     3s (x4 over 40s)  kubelet            spec.containers{api}: Error: couldn't find key pasword in Secret course/db-secret | 85c9b35 fix(k8s): read DB_PASSWORD from the Secret key password |
| 3 | After fix 2, the new Pod was 0/1 Running and never became ready: the readiness probe asked port 5001, where nothing listens | kubectl -n course describe pod <pod>: Warning  Unhealthy  2s (x10 over 39s)  kubelet            spec.containers{api}: Readiness probe failed: Get "http://10.244.0.18:5001/healthz": dial tcp 10.244.0.18:5001: connect: connection refused | 11805bd fix(k8s): readiness probe asks port 5000 where the API listens |

After the three fixes, every Pod was 1/1 and GET /api/todos answered through the port-forward:

```text
NAME                   READY   STATUS      RESTARTS   AGE
api-57679498f6-mvp9x   0/1     Completed   0          2m24s
api-94c8844-rs2xs      1/1     Running     0          6s
api-94c8844-t7lxt      1/1     Running     0          4s
api-94c8844-tbzq6      1/1     Running     0          2s
db-864878fdcf-zbm86    1/1     Running     0          5m11s

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"after the break-fix","done":false,"created_at":"2026-11-14T18:40:27.695Z"}]
```
