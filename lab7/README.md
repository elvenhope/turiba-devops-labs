# Lab 7 · Run it on a cluster

GitHub user: elvenhope · Fallback image used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. -->

## 1 · Deploy

- 🔮 Before my first `apply`: how many Pods, and what their names start with? I predicted __3 pods starting with api-__ because __replicas is 3 and the deployment is called api__. What happened: __3 pods, but the names are api- then the replicaset hash then a random bit, like api-6dbf48d96f-qgcl4. i didn't think about the replicaset part__

```text
$ kubectl -n course get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   3/3     3            3           12s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/api-6dbf48d96f   3         3         3       12s

NAME                       READY   STATUS    RESTARTS   AGE
pod/api-6dbf48d96f-qgcl4   1/1     Running   0          12s
pod/api-6dbf48d96f-qttbn   1/1     Running   0          12s
pod/api-6dbf48d96f-tkwcr   1/1     Running   0          12s

$ curl -s localhost:8080/
{"name":"course-api","version":"0.1.0"}
```

## 2 · Heal and scale

- 🔮 After deleting one Pod, two seconds later I expected to see __3 pods again, one of them new__ because __the deployment wants 3 so it makes a new one__. What happened: __yes, api-6dbf48d96f-crb9t showed up with age 3s and the other two kept their old age__

```text
$ kubectl -n course get pods        (after the delete)
NAME                   READY   STATUS    RESTARTS   AGE
api-6dbf48d96f-crb9t   1/1     Running   0          3s
api-6dbf48d96f-qttbn   1/1     Running   0          24s
api-6dbf48d96f-tkwcr   1/1     Running   0          24s

$ kubectl -n course get pods        (with replicas: 5)
NAME                   READY   STATUS    RESTARTS   AGE
api-6dbf48d96f-75bxm   1/1     Running   0          0s
api-6dbf48d96f-bfzlz   1/1     Running   0          0s
api-6dbf48d96f-crb9t   1/1     Running   0          3s
api-6dbf48d96f-qttbn   1/1     Running   0          24s
api-6dbf48d96f-tkwcr   1/1     Running   0          24s
```

## 3 · Roll and roll back

- 3a: I rolled from the tag __0.2.0__ to the tag __a9546981f7d1077fa313d4c66777e880c832eca9__.

```text
$ kubectl -n course rollout status deployment/api
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "api" rollout to finish: 1 old replicas are pending termination...
deployment "api" successfully rolled out
```

after it the new replicaset api-85d4599d6f had 3 pods and the old api-6dbf48d96f went to 0.

- 3b: 🔮 With the tag 9.9.9, I predicted that __3__ old Pods keep running, because __the new pod can't pull the image so it never gets ready, so it never kills the old ones__. What happened: __3 old ones kept running and one new pod was stuck in ErrImagePull. rollout status timed out after 30s__

```text
$ kubectl -n course get pods        (during the stall)
NAME                   READY   STATUS         RESTARTS   AGE
api-85d4599d6f-jddbp   1/1     Running        0          43s
api-85d4599d6f-wzx58   1/1     Running        0          46s
api-85d4599d6f-z9q9n   1/1     Running        0          42s
api-d67564d76-b2hnl    0/1     ErrImagePull   0          30s

$ kubectl -n course rollout undo deployment/api
Warning: resource deployments/api was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/api rolled back
```

- What the `rollout undo` warning means, in my own words, and what I did so that Git and the cluster agree again: __undo only changes the cluster. the cluster still remembers 9.9.9 as the last thing i applied and my file still said 9.9.9, so the next apply would break it again. i did git restore k8s/ and kubectl apply -f k8s/ so the file and the cluster are the same again, and git diff was empty__

## 4 · What broke and how I found it

- 🔮 Before applying the teammate's branch, I expected this to go wrong first: __the image, the tag says 0.2.O with a letter O so it won't pull__

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | a new pod stuck in ImagePullBackOff, old 3 kept running | `kubectl -n course describe pod api-84bb7b79cb-dllqh` → `Failed to pull image "ghcr.io/elvenhope/turiba-devops-labs/api:0.2.O": ... not found` | tag back to a9546981f7d1077fa313d4c66777e880c832eca9, `fix(k8s): point the image back at a tag that exists` |
| 2 | service had no pods behind it | `kubectl -n course get endpointslices -l kubernetes.io/service-name=api` → `api-6pdwh   IPv4   <unset>   <unset>`. selector was `app.kubernetes.io/name: api` but pods are `app=api` | selector back to `app: api`, `fix(k8s): match the service selector to the pod labels` |
| 3 | curl gave empty reply (exit 52) | `kubectl -n course port-forward service/api 8080:80` → `failed to connect to localhost:8080 inside namespace ... connection refused`, targetPort was 8080 but the api listens on 5000 | targetPort back to 5000, `fix(k8s): send service traffic to port 5000` |
