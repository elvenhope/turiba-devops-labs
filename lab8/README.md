# Lab 8 · Run the whole app

GitHub user: elvenhope · Fallback image used: no · Fallback Mission 7 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password, not even the base64 form. -->

## 1 · Database with storage

- 🔮 PVC right after the apply, before the db pod: I guessed __Pending__ because __kind only makes the volume once a pod wants it (WaitForFirstConsumer)__. What happened: __Pending at first, then Bound about 10s later when the db pod started. describe pvc showed WaitForFirstConsumer then ProvisioningSucceeded__
- 🔮 Before deleting the database Pod: will your todo still be there afterwards? I predicted __yes__ because __the data lives on the pvc db-data, not in the pod, and the new pod mounts the same pvc__. What happened: __it was there, but the very first GET right after gave {"error":"internal error"}. the api still had its old connection to the dead pod and the log says "Database connection lost". the second GET listed the todo again__

```text
$ kubectl -n course get pvc,pv
NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-887eba2e-739a-4144-bc26-e0c64a8557ae   1Gi        RWO            standard       <unset>                 2m7s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-887eba2e-739a-4144-bc26-e0c64a8557ae   1Gi        RWO            Delete           Bound    course/db-data   standard       <unset>                          2m3s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"survive a restart"}' localhost:8080/api/todos
{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-14T18:19:43.030Z"}

$ kubectl -n course get pods -l app=db        (before the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-r7pr5   1/1     Running   0          2m6s

$ kubectl -n course delete pod db-84f4dd66bb-r7pr5
pod "db-84f4dd66bb-r7pr5" deleted from course namespace

$ kubectl -n course get pods -l app=db        (after the delete)
NAME                  READY   STATUS    RESTARTS   AGE
db-84f4dd66bb-ljhzv   1/1     Running   0          1s

$ curl -s localhost:8080/api/todos
{"error":"internal error"}

$ curl -s localhost:8080/api/todos
[{"id":1,"title":"survive a restart","done":false,"created_at":"2026-11-14T18:19:43.030Z"}]
```

new pod name (ljhzv instead of r7pr5), same todo with the same id and created_at, so it really came from the disk and not from a new insert.

## 2 · Connect the API

```text
$ kubectl -n course logs deployment/api -c wait-for-db
Found 3 pods, using pod/api-7b78b774dc-72d9x
db:5432 - accepting connections

$ kubectl -n course logs deployment/api -c api
Found 3 pods, using pod/api-7b78b774dc-72d9x
course-api 0.1.0 starting (Node.js v24.21.0)
API listening on port 5000
Database ready
Database connection lost: terminating connection due to administrator command
GET /api/todos failed: Connection terminated due to connection timeout
```

the last two lines are from when i deleted the db pod in section 1, that's the 500 i got.

- 🔮 Before looking: will `kubectl -n course get secret db-secret -o yaml` show your password? I predicted __not in plain text, but kind of yes__ because __secrets are only base64, not encrypted__. What I saw, in words (don't paste the value): __a data: block with one key, password, and a base64 string next to it, no plain text. but anyone who can run this can just base64 -d it, so it's hidden not secret. the protection is that it's not in git and who has access to the cluster__

## 3 · Health and resources

- 🔮 On my working copy only, I pointed the **liveness** probe of the API at `/nope` and applied it. Will those Pods get traffic? Will they be restarted? I predicted __they get traffic but get restarted over and over__ because __readiness still asks /healthz so they count as ready, but liveness gets 404 and after 3 fails kubelet kills the container__. What happened: __mostly right. they were 1/1 and got restarted, RESTARTS was 1 for each after a minute (liveness waits 10s, then 3 fails 10s apart, so about every 40s). what i didn't think about: right after every restart readiness fails for a moment (connection refused, the api isn't up yet) so for a few seconds they're out of the service too__

```text
$ kubectl -n course get pods -l app=api       (about a minute later)
NAME                  READY   STATUS    RESTARTS      AGE
api-57bb5dfd9-hgxsr   1/1     Running   1 (26s ago)   66s
api-57bb5dfd9-rcqsv   1/1     Running   1 (30s ago)   71s
api-57bb5dfd9-x25bx   1/1     Running   1 (28s ago)   68s

$ kubectl -n course describe pod api-57bb5dfd9-hgxsr         (only the Events)
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  66s                default-scheduler  Successfully assigned course/api-57bb5dfd9-hgxsr to kind-control-plane
  Normal   Pulled     66s                kubelet            spec.initContainers{wait-for-db}: Container image "postgres:17-alpine" already present on machine and can be accessed by the pod
  Normal   Created    66s                kubelet            spec.initContainers{wait-for-db}: Container created
  Normal   Started    66s                kubelet            spec.initContainers{wait-for-db}: Container started
  Normal   Pulled     26s (x2 over 65s)  kubelet            spec.containers{api}: Container image "ghcr.io/elvenhope/turiba-devops-labs/api:a9546981f7d1077fa313d4c66777e880c832eca9" already present on machine and can be accessed by the pod
  Normal   Created    26s (x2 over 65s)  kubelet            spec.containers{api}: Container created
  Normal   Started    26s (x2 over 65s)  kubelet            spec.containers{api}: Container started
  Warning  Unhealthy  26s (x2 over 65s)  kubelet            spec.containers{api}: Readiness probe failed: Get "http://10.244.0.12:5000/healthz": dial tcp 10.244.0.12:5000: connect: connection refused
  Normal   Killing    26s                kubelet            spec.containers{api}: Container api failed liveness probe, will be restarted
  Warning  Unhealthy  6s (x5 over 46s)   kubelet            spec.containers{api}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

- How I made Git and the cluster agree again afterwards: __`git restore k8s/ && kubectl apply -f k8s/`. that rolled the api back to the main version (new pods with 0 restarts) and `git diff` was empty after__

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: __something with the config, i guessed the new api pods would get stuck and the old ones from main keep running. didn't know which fault would show first__. it was the DB_HOST one, cause the init container runs before anything else in the pod. and yes the 3 old pods kept serving the whole time.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | new api pod stuck in `Init:0/1`, `DB_HOST` in app-config.yaml was changed to `database` and there's no service with that name | `kubectl -n course logs api-7d7df94dd5-dgq65 -c wait-for-db` → `database:5432 - no response` | set it back to `DB_HOST: db`, applied, then `kubectl -n course rollout restart deployment/api` so the pods read the configmap again. `bbcf86c fix(k8s): point DB_HOST back at the db service` |
| 2 | after that the new pod went `CreateContainerConfigError`, the secretKeyRef key was `pasword` | `kubectl -n course describe pod api-59f4df7679-jzzx6` → `Error: couldn't find key pasword in Secret course/db-secret` | key back to `password`. `896628c fix(k8s): use the password key of db-secret` |
| 3 | then the pod was `0/1 Running` and never got ready, the readiness probe port was `5001` | `kubectl -n course describe pod api-9f9cbcd56-9s96s` → `Readiness probe failed: Get "http://10.244.0.18:5001/healthz": dial tcp 10.244.0.18:5001: connect: connection refused` | port back to `5000`. `0ab6790 fix(k8s): send the readiness probe to port 5000` |

after the third fix all 3 new api pods were 1/1, `curl -s localhost:8080/api/todos` answered through the port-forward and `git diff main -- k8s/` was empty. merged as PR #23.
