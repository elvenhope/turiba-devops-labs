# Lab 10 · Build it from code with OpenTofu

GitHub user: elvenhope · Fallback chart used: no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores). Paste command output inside the ```text blocks. Never paste the password, the state file or the kubeconfig. -->

## 1 · The cluster and the app from code

```text
$ tofu plan                                    (Part 1: the end of the first plan)
  # kind_cluster.course will be created
  + resource "kind_cluster" "course" {
      + client_certificate     = (known after apply)
      + client_key             = (known after apply)
      + cluster_ca_certificate = (known after apply)
      + completed              = (known after apply)
      + context_name           = (known after apply)
      + endpoint               = (known after apply)
      + id                     = (known after apply)
      + kubeconfig             = (known after apply)
      + kubeconfig_path        = (known after apply)
      + name                   = "course"
      + node_image             = (known after apply)
      + wait_for_ready         = true
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + kube_context = (known after apply)

$ time tofu apply -auto-approve                (Part 2: from zero, the last lines and the time)
random_password.db: Creation complete after 0s [id=none]
kind_cluster.course: Creation complete after 28s [id=course-]
kubernetes_namespace_v1.course: Creation complete after 0s [id=course]
kubernetes_secret_v1.db: Creation complete after 0s [id=course/db-secret]
helm_release.course_app: Creation complete after 20s [id=course-app]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

kube_context = "kind-course"
real 48.94
user 4.90
sys 2.69

$ kubectl -n course get pods
NAME                   READY   STATUS    RESTARTS   AGE
api-79d676fcbc-b9zhw   1/1     Running   0          20s
api-79d676fcbc-djs25   1/1     Running   0          20s
api-79d676fcbc-fm9br   1/1     Running   0          20s
db-864878fdcf-td8rq    1/1     Running   0          20s

$ curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"built from code"}' localhost:8080/api/todos
{"id":1,"title":"built from code","done":false,"created_at":"2026-11-21T18:58:53.730Z"}

$ tofu plan                                    (right after that apply)
No changes. Your infrastructure matches the configuration.

$ grep -c '"result"' terraform.tfstate
2

$ tofu state show kubernetes_secret_v1.db | grep data
    binary_data_wo                 = (write-only attribute)
    data                           = (sensitive value)
    data_wo                        = (write-only attribute)
    metadata {
```

- 🔮 Before the first plan (Part 1): how many resources will it add, and which values will be `known after apply`? I predicted 1 to add (only the kind cluster, the other providers aren't used yet) and the endpoint, the certs and the context name as known after apply because those only exist once kind actually made the cluster. What happened: 1 to add, right. more stuff was known after apply than i thought tho: also `id`, `kubeconfig`, `kubeconfig_path`, `node_image` and `completed`, basically everything except `name` and `wait_for_ready` which i wrote myself
- 🔮 Before the second plan (Part 2): what will `tofu plan` say right after `tofu apply`? I predicted no changes because nothing changed between the apply and the plan, the code, state and cluster should all agree. What happened: `No changes. Your infrastructure matches the configuration.`
- 🔮 Where is the database password now? I predicted in the k8s secret and in terraform.tfstate in plain text because tofu has to remember what random_password made, sensitive only hides it in the output. What happened: `grep -c` found 2 `"result"` lines in the state (one is the password), `tofu state show` only says `(sensitive value)`. so the plan hides it but the state file doesn't, that's why it's gitignored

## 2 · Change, drift, rebuild

```text
$ tofu apply -var api_replicas=2               (the plan: the symbol and the value that changes)
  # helm_release.course_app will be updated in-place
  ~ resource "helm_release" "course_app" {
        ...
        name                       = "course-app"
      ~ set                        = [
          ~ {
                name  = "api.replicas"
              ~ value = "3" -> "2"
                # (1 unchanged attribute hidden)
            },
        ]
        # (27 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
helm_release.course_app: Modifications complete after 0s [id=course-app]

Apply complete! Resources: 0 added, 1 changed, 0 destroyed.

$ helm history course-app -n course
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Sat Nov 21 20:58:30 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Sat Nov 21 21:00:48 2026	deployed  	course-app-0.1.0	0.2.0      	Upgrade complete

$ kubectl -n course get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
api    2/2     2            2           2m19s

$ tofu plan -var api_replicas=2                (after kubectl -n course delete secret db-secret)
  # kubernetes_secret_v1.db will be created
  + resource "kubernetes_secret_v1" "db" {
      + data                           = (sensitive value)
      + id                             = (known after apply)
      + type                           = "Opaque"
      ...
      + metadata {
          + name             = "db-secret"
          + namespace        = "course"
          ...
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu plan -var api_replicas=2                (after kubectl -n course scale deployment api --replicas=1)
  # kubernetes_secret_v1.db will be created
  + resource "kubernetes_secret_v1" "db" {
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

$ tofu apply                                   (the plan, and how many API Pods run afterwards)
  # helm_release.course_app will be updated in-place
  ~ resource "helm_release" "course_app" {
      ~ set                        = [
          ~ {
                name  = "api.replicas"
              ~ value = "2" -> "3"
            },
        ]
    }

  # kubernetes_secret_v1.db will be created
  + resource "kubernetes_secret_v1" "db" {
      ...
    }

Plan: 1 to add, 1 to change, 0 to destroy.
kubernetes_secret_v1.db: Creation complete after 0s [id=course/db-secret]
helm_release.course_app: Modifications complete after 4s [id=course-app]

Apply complete! Resources: 1 added, 1 changed, 0 destroyed.

$ kubectl -n course get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
api    3/3     3            3           2m30s

$ time tofu destroy -auto-approve / time tofu apply -auto-approve      (the last lines and the times)
helm_release.course_app: Destruction complete after 2s
kubernetes_secret_v1.db: Destruction complete after 0s
random_password.db: Destruction complete after 0s
kubernetes_namespace_v1.course: Destruction complete after 7s
kind_cluster.course: Destruction complete after 0s
Destroy complete! Resources: 5 destroyed.
real 9.65

random_password.db: Creation complete after 0s [id=none]
kind_cluster.course: Creation complete after 27s [id=course-]
kubernetes_namespace_v1.course: Creation complete after 0s [id=course]
kubernetes_secret_v1.db: Creation complete after 0s [id=course/db-secret]
helm_release.course_app: Creation complete after 22s [id=course-app]
Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
real 50.32

$ curl -s localhost:8080/api/todos             (after the rebuild)
[]
```

(the `...` are where i cut the metadata / known after apply lines, the plans are long)

right before the destroy the same curl gave `[{"id":1,"title":"built from code",...}]`.

- 🔮 Step 1, `-var api_replicas=2`: which symbol, and which value changes? I predicted `~` (update in-place) on the helm_release with the replicas going 3 -> 2 because the release itself stays, only one setting changes. What happened: `~ value = "3" -> "2"` inside `set`, `0 to add, 1 to change`. helm history shows revision 2 `Upgrade complete`, so tofu just does a helm upgrade under the hood and helm still keeps its own history
- 🔮 Step 2, the deleted Secret and the scaled Deployment: what will each plan show? I predicted the secret comes back as `+ create`, and the scale shows up as a change on the helm_release because tofu would see the deployment doesn't match. What happened: the secret one was right (`1 to add`). the scale didn't show up at all, the second plan is exactly the same as the first. the deployment isn't a tofu resource, it's inside the helm release and tofu only looks at the release, which didn't change. so drift inside the chart is invisible
- 🔮 Step 3, a plain `tofu apply`: what will the plan contain, and how many API Pods afterwards? I predicted create the secret + update the release 2 -> 3, and 3 pods because the default api_replicas is 3. What happened: `Plan: 1 to add, 1 to change, 0 to destroy.`, `3/3`. the helm upgrade also undid my `kubectl scale` to 1 cause helm sets replicas again
- 🔮 Step 4, after `tofu destroy` and `tofu apply`: is the todo still there? I predicted no because the pvc lives inside the kind cluster and destroy deletes the whole cluster. What happened: `[]`. everything in the code came back in ~50s, the data didn't

## 3 · What broke and how I found it

- 🔮 Before `tofu plan` on the teammate's branch, I expected this to go wrong first: the provider change in versions.tf, since tofu checks the providers / lock file before it even reads the chart or talks to the cluster

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | plan stopped right away about the lock file | `tofu plan` → `Error: Inconsistent dependency lock file` / `provider registry.opentofu.org/hashicorp/kind: required by this configuration but no version is selected`. `tofu init` then: `provider registry registry.opentofu.org does not have a provider named registry.opentofu.org/hashicorp/kind` and `Did you intend to use tehcyx/kind?`. the source in versions.tf was changed to `hashicorp/kind` | `source = "tehcyx/kind"` again (0912a74 `fix(infra): use the tehcyx/kind provider again`) |
| 2 | after fix 1 the plan got further but couldn't find the chart | `tofu plan` → `Error: Error locating chart` / `Unable to locate chart ./../chart/course-app: path "./../chart/course-app" not found`. `chart` instead of `charts` | path back to `${path.module}/../charts/course-app` (e69129a `fix(infra): point the release at charts/course-app again`) |
| 3 | plan worked but wanted to replace the whole release | `tofu plan` → `# helm_release.course_app must be replaced`, `~ namespace = "course" -> "courses" # forces replacement`, `Plan: 1 to add, 0 to change, 1 to destroy.` the namespace was hardcoded to `"courses"` which doesn't even exist. i did NOT apply it | `namespace = kubernetes_namespace_v1.course.metadata[0].name` again (712b77d `fix(infra): put the release back in the course namespace`) |

after the 3 fixes: `tofu plan` → `No changes. Your infrastructure matches the configuration.`, `tofu apply` → `0 added, 0 changed, 0 destroyed`, `kubectl -n course get deployment api` → `3/3`, and `curl -s localhost:8080/api/todos` still gave `[{"id":1,"title":"dont lose me",...}]`, the todo i saved right before running break.sh. so not applying fault 3 actually saved the data.

## 4 · In my own words

- Why `.terraform.lock.hcl` goes into Git and `terraform.tfstate` doesn't: the lock file pins the exact provider versions (and their hashes) so everyone and the grader gets the same providers on `tofu init`, it has nothing secret in it. the state has every value tofu made in plain text, the db password from random_password, the cluster certs and key, so committing it = leaking secrets. it's also specific to my machine's cluster, nobody else needs it
- What `-/+ … # forces replacement` means, and what it would have cost me in Part 4: tofu can't change that attribute in place so it destroys the resource and creates a new one. for a helm_release that's helm uninstall + install, and the pvc is part of the release, so the db and all todos would be gone. and here the new install would also fail cause `courses` doesn't exist, so i'd end up with no app at all until i fix it and apply again
- What I would lose by ending the day with `kind delete cluster` instead of `tofu destroy`: the state would still list the cluster and everything on it, so the next plan/apply fails (`could not locate any control plane nodes` / connection refused to the old cluster) and i'd have to delete the state files by hand. basically tofu loses track of what's real, which is the whole point of using it
