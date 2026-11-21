# Lab 9 · Package it with Helm

GitHub user: elvenhope · Fallback image used: no · Fallback Mission 8 files used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste your password. -->

## 1 · The chart

```text
$ helm lint charts/course-app
==> Linting charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
  replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/elvenhope/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- 🔮 Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted only the api's `replicas:` line goes to 2, the db stays 1 and the images don't change, and without `--set` values.yaml decides (3) because the template only has `{{ .Values.api.replicas }}` in it and --set just overrides that one value. What happened: that's what happened, api `replicas: 2`, db `replicas: 1`. i forgot there are two postgres images in there tho (the wait-for-db init container uses postgres too), so 3 image lines not 2. without --set it goes back to `replicas: 3`
- In my own words, why the Namespace and the Secret are **not** in the chart: helm only manages stuff it made itself and won't take over something that already exists. the namespace has to exist before the install (helm install -n course puts everything into it), and the secret has the real password which can't go in git, so both get made with kubectl first. also if the secret was in the chart, helm uninstall would delete it too

## 2 · Install, upgrade, rollback

```text
$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Sat Nov 21 20:46:51 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Sat Nov 21 20:48:04 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS         RESTARTS   AGE
api-559d6d7788-gfl4w   0/1     ErrImagePull   0          35s
api-79d676fcbc-q4p66   1/1     Running        0          103s
api-79d676fcbc-vhw28   1/1     Running        0          103s
api-79d676fcbc-xbz2t   1/1     Running        0          103s

$ helm history course-app -n course            (before the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Sat Nov 21 20:46:51 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Sat Nov 21 20:48:04 2026	deployed  	course-app-0.1.0	0.2.0      	Upgrade complete

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ helm history course-app -n course            (after the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Sat Nov 21 20:46:51 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Sat Nov 21 20:48:04 2026	superseded	course-app-0.1.0	0.2.0      	Upgrade complete
3       	Sat Nov 21 20:48:40 2026	deployed  	course-app-0.1.0	0.2.0      	Rollback to 1
```

- 🔮 Before the upgrade with a tag that doesn't exist: will `helm upgrade` say it failed? Will `GET /api/todos` still answer? I predicted helm says it failed and the api still answers because i thought helm would notice the image can't be pulled, and the rolling update keeps the old pods until new ones are ready. What happened: half right. the api answered fine (3 old pods still `Running`, the new one `ErrImagePull`), but helm said `STATUS: deployed` / `Upgrade complete`. helm only checks that k8s accepted the change, not that the pods actually came up. you'd need `--wait` for that
- 🔮 Before the rollback: which revision number will `helm rollback course-app 1` create? I predicted 3 because a rollback is just another change, it doesn't delete revision 2. What happened: revision 3 with `Rollback to 1`, 1 and 2 are both `superseded`. `helm get values` after that shows `USER-SUPPLIED VALUES: null` so the bad tag is really gone
- 🔮 Before `helm uninstall` and a new `helm install`: will your todo still be there? I predicted no because the pvc db-data is one of the chart's templates so uninstall deletes it with everything else. What happened: gone, `[]`. `get pvc,secret` right after the uninstall showed db-data `Terminating` while db-secret was still there (helm didn't make it)

```text
$ curl -s localhost:8080/api/todos             (after uninstall and install)
[]
```

before the uninstall the same curl gave `[{"id":1,"title":"packaged by helm",...}]`, also during the broken upgrade and after the rollback.

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: something in the templates that stops helm from rendering, since helm renders everything before it talks to the cluster. the files changed tab had a new namespace.yaml too so i figured that one would fail after

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | the upgrade stopped before anything reached the cluster | `helm upgrade course-app charts/course-app -n course` → `Error: UPGRADE FAILED: course-app/templates/api-deployment.yaml:23:28 executing "course-app/templates/api-deployment.yaml" at <.Values.api.images.repository>: nil pointer evaluating interface {}.repository`. values.yaml has `image`, not `images` | `images` back to `image` in api-deployment.yaml (c3a7547 `fix(chart): read the image from api.image again`) |
| 2 | after fix 1 it rendered but the upgrade still failed | `helm upgrade course-app charts/course-app -n course` → `Error: UPGRADE FAILED: unable to continue with update: Namespace "course" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata`. the namespace was made with kubectl so helm won't own it | `git rm` templates/namespace.yaml (0379401 `fix(chart): drop the namespace from the chart`) |
| 3 | upgrade said `deployed` and lint passed, but only 1 api pod | `kubectl -n course get deployment api` → `api    1/1     1            1`. then `helm template course-app charts/course-app -n course \| grep replicas:` → `replicas: ` with nothing after it. values.yaml said `replicaCount: 3` but the template asks for `api.replicas`, so it's empty and k8s uses 1 | `replicaCount` back to `replicas` in values.yaml (58528a1 `fix(chart): name the value api.replicas again`) |

after the 3 fixes: `helm upgrade` → revision 3 deployed, `kubectl -n course get deployment api` → `3/3`, `curl -s localhost:8080/api/todos` → `[]` through the port-forward. `git diff main -- charts/` was empty so the chart is exactly back to what main had.
