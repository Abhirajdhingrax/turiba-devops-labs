# Lab 9 · Package it with Helm

GitHub user: Abhirajdhingrax · Fallback image used: yes · Fallback Mission 8 files used: no

## 1 · The chart

```text
$ helm lint charts/course-app
==> Linting charts/course-app
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed

$ helm template course-app charts/course-app -n course --set api.replicas=2 | grep -E 'replicas:|image:'
replicas: 2
          image: postgres:17-alpine
          image: "ghcr.io/martinsturiba/turiba-devops-labs/api:0.2.0"
  replicas: 1
          image: postgres:17-alpine
```

- 🔮 Before running `helm template … --set api.replicas=2`: which lines of the output change, and which file decides that? I predicted only the replicas line of the API Deployment would change to 2, because the template reads `.Values.api.replicas`, and without `--set` the value comes from values.yaml. What happened: the API Deployment said `replicas: 2`, the database kept `replicas: 1`, and the images did not change. Without `--set` the API said `replicas: 3` from values.yaml.
- In my own words, why the Namespace and the Secret are **not** in the chart: Helm only manages objects it created itself and refuses to take over existing ones. The Namespace has to exist before the release is installed into it, and the password must never be stored in Git, so both are made with kubectl outside the chart.

## 2 · Install, upgrade, rollback

```text
$ helm install course-app charts/course-app -n course
NAME: course-app
LAST DEPLOYED: Wed Nov 18 19:55:15 2026
NAMESPACE: course
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None

$ helm upgrade course-app charts/course-app -n course --set api.image.tag=does-not-exist
Release "course-app" has been upgraded. Happy Helming!
NAME: course-app
LAST DEPLOYED: Wed Nov 18 20:00:10 2026
NAMESPACE: course
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None

$ kubectl -n course get pods -l app=api        (about 30 seconds later)
NAME                   READY   STATUS             RESTARTS   AGE
api-57679498f6-2z4pm   1/1     Running            0          5m34s
api-57679498f6-cclsb   1/1     Running            0          5m34s
api-57679498f6-n4j8v   1/1     Running            0          5m34s
api-745f697dc7-qztbf   0/1     ImagePullBackOff   0          38s

$ helm history course-app -n course            (before the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 19:55:15 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Wed Nov 18 20:00:10 2026	deployed  	course-app-0.1.0	0.2.0      	Upgrade complete

$ helm rollback course-app 1 -n course
Rollback was a success! Happy Helming!

$ helm history course-app -n course            (after the rollback)
REVISION	UPDATED                 	STATUS    	CHART           	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 19:55:15 2026	superseded	course-app-0.1.0	0.2.0      	Install complete
2       	Wed Nov 18 20:00:10 2026	superseded	course-app-0.1.0	0.2.0      	Upgrade complete
3       	Wed Nov 18 20:00:56 2026	deployed  	course-app-0.1.0	0.2.0      	Rollback to 1
```

- 🔮 Before the upgrade with a tag that doesn't exist: will `helm upgrade` say it failed? Will `GET /api/todos` still answer? I predicted it would not say failed, because Helm only reports that the cluster accepted the change, and that the API would still answer because the rolling update keeps the old Pods. What happened: helm upgrade said deployed, the new Pod could not pull its image while the old Pods kept running, and GET /api/todos still answered with my todo.
- 🔮 Before the rollback: which revision number will `helm rollback course-app 1` create? I predicted revision 3, because a rollback does not delete history but adds a new revision with the settings of revision 1. What happened: helm history shows revision 3 with the description Rollback to 1, and revisions 1 and 2 are superseded.
- 🔮 Before `helm uninstall` and a new `helm install`: will your todo still be there? I predicted no, because the PVC db-data is part of the release, so uninstall deletes it together with the data. What happened: the todo list was empty after the new install, and only the Secret survived the uninstall.

```text
$ curl -s localhost:8080/api/todos             (after uninstall and install)
[]
```

## 3 · What broke and how I found it

- 🔮 Before upgrading with the teammate's branch, I expected this to go wrong first: the template error, because Helm renders the templates before it talks to the cluster, so a wrong value name stops it before anything is applied.

| # | What I saw | The command, and the line of output that showed it | My fix (commit) |
|--:|------------|-----------------------------------------------------|-----------------|
| 1 | Every helm upgrade stopped before anything reached the cluster: the template read `.Values.api.images.repository`, but values.yaml has `image`, not `images` | helm upgrade course-app charts/course-app -n course: nil pointer evaluating interface {}.repository | 26c1a58 fix(chart): read the image repository from api.image again |
| 2 | The new templates/namespace.yaml made the upgrade fail, because the Namespace was made with kubectl and Helm never takes over an object it did not make | helm upgrade course-app charts/course-app -n course: Error: UPGRADE FAILED: unable to continue with update: Namespace "course" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm"; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "course-app"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "course" | 7407263 fix(chart): remove the Namespace, it stays outside the chart |
| 3 | helm lint passed and the upgrade said deployed, but the API had only 1 Pod: values.yaml said replicaCount, so the template's api.replicas was empty and Kubernetes used its default of 1 | kubectl -n course get deployment api: api    1/1     1            1           117s; helm template … grep replicas: 104:  replicas: 1 | 44e80bf fix(chart): rename replicaCount back to replicas in values |

After the three fixes:

```text
$ kubectl -n course get deployment api
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
api    3/3     3            3           2m27s

$ curl -s localhost:8080/api/todos
[]
```
