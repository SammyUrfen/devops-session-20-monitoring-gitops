# Monitoring, Observability & GitOps Homework — Session 20

Bibek Jyoti Charah — 24bcs10112 (GitHub: SammyUrfen)

I ran every command on 7 Oct 2026 on Fedora, against a minikube v1.38.1 cluster (profile `s20`,
Docker driver, 4 CPUs, 4096 MB, Kubernetes v1.35.1) with `kubectl` v1.37.0. All output is pasted
as printed. Long output is cut with `...`. The homework asks for screenshots. I pasted terminal
output instead.

| Homework item | Section |
| :--- | :--- |
| Task 1: metrics, CPU, memory | [1.3](#13--metrics-promql-through-the-prometheus-api), [1.4](#14--cpu-and-memory-per-pod) |
| Task 1: logs | [1.5](#15--logs) |
| Task 1: alerts | [1.6](#16--alerts-a-custom-prometheusrule-that-fires) |
| Task 1: application health | [1.2](#12--demo-app-with-probes-and-a-servicemonitor), [1.3](#13--metrics-promql-through-the-prometheus-api) |
| Task 1: Grafana | [1.7](#17--grafana) |
| Task 2: observability doc | [OBSERVABILITY.md](OBSERVABILITY.md) |
| Task 3: GitOps demo | [Task 3](#task-3--gitops-with-argo-cd) |
| Task 3: GitOps concepts | [3.5](#35--gitops-concepts) |
| Failures and fixes | [Findings](#findings) |

## Repo layout

| Path | What it is |
| :--- | :--- |
| `monitoring/values-slim.yaml` | Helm values for kube-prometheus-stack: small requests/limits, 6 h retention, no persistence |
| `monitoring/demo-app.yaml` | Demo app (`prometheus-example-app`) with probes, a Service and a ServiceMonitor |
| `monitoring/demo-alerts.yaml` | `PrometheusRule` with `DemoAppDown` and `DemoAppHighMemory` |
| `gitops/` | The manifests Argo CD watches: Namespace, Deployment, Service (from the course `08-mini-project`) |
| `argocd/application.yaml` | The Argo CD `Application`. It sits outside `gitops/`, as the course says. |
| `argocd/appproject-default.yaml` | The `default` AppProject (see Findings 2) |
| `OBSERVABILITY.md` | Task 2 |

## Task 1 — Monitoring

### 1.1 — Install kube-prometheus-stack

The first install failed because Grafana was OOMKilled at a 256Mi limit (Findings 1). I raised the
limit to 512Mi and ran `helm upgrade`.

```text
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories
$ helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f monitoring/values-slim.yaml --wait --timeout 10m
Error: INSTALLATION FAILED: resource Deployment/monitoring/kps-grafana not ready. status: Failed, message: Progress deadline exceeded
...
$ helm upgrade kps prometheus-community/kube-prometheus-stack -n monitoring -f monitoring/values-slim.yaml --wait --timeout 8m
Release "kps" has been upgraded. Happy Helming!
NAME: kps
LAST DEPLOYED: Wed Oct  7 21:21:07 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
NOTES:
...
$ helm ls -n monitoring
NAME	NAMESPACE 	REVISION	UPDATED                                	STATUS  	CHART                       	APP VERSION
kps 	monitoring	2       	2026-10-07 21:21:07.869837474 +0530 IST	deployed	kube-prometheus-stack-92.1.0	v0.94.1    
$ kubectl get pods -n monitoring
NAME                                                    READY   STATUS        RESTARTS        AGE
alertmanager-kps-kube-prometheus-stack-alertmanager-0   2/2     Running       0               29m
kps-grafana-59b7cfbcf6-lqtz6                            3/3     Running       0               42s
kps-grafana-7ccc5c5bdc-jbp7b                            3/3     Terminating   8 (5m33s ago)   30m
kps-kube-prometheus-stack-operator-d5468b5f8-wkxpz      1/1     Running       0               30m
kps-kube-state-metrics-6855b75755-5224z                 1/1     Running       0               30m
kps-prometheus-node-exporter-pfzv7                      1/1     Running       0               30m
prometheus-kps-kube-prometheus-stack-prometheus-0       2/2     Running       0               29m
```

I reached the UIs and APIs with port-forwards on my session's port range:

```text
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 42090:9090
kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-alertmanager 42093:9093
kubectl port-forward -n monitoring svc/kps-grafana 42030:80
```

### 1.2 — Demo app with probes and a ServiceMonitor

The demo app serves `/` and `/metrics` on port 8080. The readiness probe removes a pod from the
Service when it fails. The liveness probe restarts the container when it fails.

```text
$ kubectl apply -f monitoring/demo-app.yaml -f monitoring/demo-alerts.yaml
namespace/demo created
deployment.apps/demo-app created
service/demo-app created
servicemonitor.monitoring.coreos.com/demo-app created
prometheusrule.monitoring.coreos.com/demo-app-alerts created
$ kubectl get pods,svc,servicemonitor,prometheusrule -n demo
NAME                            READY   STATUS    RESTARTS   AGE
pod/demo-app-589d74c5c5-rrksj   1/1     Running   0          14s
pod/demo-app-589d74c5c5-tzdc8   1/1     Running   0          14s

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/demo-app   ClusterIP   10.106.212.108   <none>        8080/TCP   14s

NAME                                            AGE
servicemonitor.monitoring.coreos.com/demo-app   14s

NAME                                                   AGE
prometheusrule.monitoring.coreos.com/demo-app-alerts   14s
$ kubectl describe pod -n demo -l app=demo-app | grep -E "Liveness|Readiness"
    Liveness:     http-get http://:http/ delay=0s timeout=1s period=10s successThreshold=1 failureThreshold=3
    Readiness:    http-get http://:http/ delay=0s timeout=1s period=5s successThreshold=1 failureThreshold=3
    Liveness:     http-get http://:http/ delay=0s timeout=1s period=10s successThreshold=1 failureThreshold=3
    Readiness:    http-get http://:http/ delay=0s timeout=1s period=5s successThreshold=1 failureThreshold=3
```

### 1.3 — Metrics: PromQL through the Prometheus API

`up` is 1 when Prometheus scraped the target with success. kube-state-metrics gives the Kubernetes
view of health (available replicas, restarts).

```text
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=up{job="demo-app"}' | jq -r '.data.result[] | "\(.metric.job // "") \(.metric.pod // .metric.instance // "") \(.value[1])"'
demo-app demo-app-589d74c5c5-rrksj 1
demo-app demo-app-589d74c5c5-tzdc8 1
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=count by (job) (up == 1)' | jq -r '.data.result[] | "\(.metric.job // "") \(.metric.pod // .metric.instance // "") \(.value[1])"'
kps-kube-prometheus-stack-alertmanager  2
kps-kube-prometheus-stack-prometheus  2
kubelet  3
kube-state-metrics  1
kps-kube-prometheus-stack-operator  1
coredns  1
apiserver  1
node-exporter  1
kps-grafana  1
demo-app  2
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=kube_deployment_status_replicas_available{namespace="demo"}' | jq -r '.data.result[] | "\(.metric.job // "") \(.metric.pod // .metric.instance // "") \(.value[1])"'
kube-state-metrics kps-kube-state-metrics-6855b75755-5224z 2
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=sum by (pod) (kube_pod_container_status_restarts_total{namespace="monitoring"})' | jq -r '.data.result[] | "\(.metric.job // "") \(.metric.pod // .metric.instance // "") \(.value[1])"'
 prometheus-kps-kube-prometheus-stack-prometheus-0 0
 kps-prometheus-node-exporter-pfzv7 0
 kps-kube-state-metrics-6855b75755-5224z 0
 alertmanager-kps-kube-prometheus-stack-alertmanager-0 0
 kps-kube-prometheus-stack-operator-d5468b5f8-wkxpz 0
 kps-grafana-59b7cfbcf6-lqtz6 0
```

(The second line of the replica query shows the kube-state-metrics pod name because the jq filter
prints `.metric.pod`, and on that series the label holds the scraping pod. The value `2` is the
demo app's available replicas.)

### 1.4 — CPU and memory per pod

My first queries filtered on `container!=""` and returned nothing. On this cluster the cAdvisor
series have no `container` label (Findings 3), so I filter on `pod!=""`.

```text
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=count by (namespace,container) (container_memory_working_set_bytes)' | jq -c '.data.result[].metric'
{"namespace":"kube-system"}
{"namespace":"monitoring"}
{"namespace":"demo"}
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=sum by (pod) (rate(container_cpu_usage_seconds_total{namespace=~"monitoring|demo",pod!=""}[2m]))' | jq -r '.data.result[] | "\(.metric.pod) \(.value[1])"'
kps-prometheus-node-exporter-pfzv7 0.0037156167978123784
kps-kube-state-metrics-6855b75755-5224z 0.0011144158338909156
prometheus-kps-kube-prometheus-stack-prometheus-0 0.027979293146283633
kps-kube-prometheus-stack-operator-d5468b5f8-wkxpz 0.019324344666832544
alertmanager-kps-kube-prometheus-stack-alertmanager-0 0.0011488457516801193
kps-grafana-59b7cfbcf6-lqtz6 0.019897714568821695
demo-app-589d74c5c5-rrksj 0.0001425422151898733
demo-app-589d74c5c5-tzdc8 0.00011943999999999997
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=max by (pod) (container_memory_working_set_bytes{namespace=~"monitoring|demo",pod!=""}) / 1024 / 1024' | jq -r '.data.result[] | "\(.metric.pod) \(.value[1])"'
kps-prometheus-node-exporter-pfzv7 19.7734375
kps-kube-state-metrics-6855b75755-5224z 26.3203125
prometheus-kps-kube-prometheus-stack-prometheus-0 234.5390625
kps-kube-prometheus-stack-operator-d5468b5f8-wkxpz 26.94140625
alertmanager-kps-kube-prometheus-stack-alertmanager-0 45.12890625
kps-grafana-59b7cfbcf6-lqtz6 558.64453125
demo-app-589d74c5c5-rrksj 2.37890625
demo-app-589d74c5c5-tzdc8 3.03125
```

How to read it: CPU is in cores, so Prometheus uses about 0.028 cores (28 millicores). Memory is in
MiB. Grafana is the largest pod at about 559 MiB. That number is the whole pod (Grafana plus two
sidecars). The 512Mi limit applies only to the Grafana container.

### 1.5 — Logs

```text
$ kubectl logs -n demo deploy/demo-app --tail=5
Found 2 pods, using pod/demo-app-589d74c5c5-rrksj
$ kubectl logs -n monitoring prometheus-kps-kube-prometheus-stack-prometheus-0 -c prometheus --tail=3
time=2026-10-07T15:25:30.443Z level=INFO source=manager.go:235 msg="Starting rule manager..." component="rule manager"
time=2026-10-07T15:25:30.532Z level=INFO source=main.go:1721 msg="Loading configuration file" filename=/etc/prometheus/config_out/prometheus.env.yaml
time=2026-10-07T15:25:30.614Z level=INFO source=main.go:1763 msg="Completed loading of configuration file" db_storage=3.163µs remote_storage=2.68µs web_handler=859ns query_engine=1.124µs scrape=77.209µs scrape_sd=52.615µs notify=168.061µs notify_sd=6.17µs rules=71.204637ms tracing=6.059µs filename=/etc/prometheus/config_out/prometheus.env.yaml totalDuration=82.183123ms
```

The demo app prints nothing to stdout, so its log is empty. That is a real lesson: an app that does
not log gives you metrics only. The Prometheus log shows it reloaded config after I added the
`PrometheusRule` (`rules=71ms`). The Grafana crash log in Findings 1 and the Argo CD controller log
in 3.3 are more `kubectl logs` examples.

**Where Loki fits.** `kubectl logs` reads one pod at a time from the node, and the logs go away
with the pod. Loki stores logs centrally and indexes them by labels only (namespace, pod, app), the
same labels Prometheus uses. A DaemonSet agent (Promtail or Grafana Alloy) tails
`/var/log/pods/*` on each node and pushes to Loki. Grafana then shows logs next to metrics.
**I did not install Loki.** The monitoring stack already used about 910 MiB (sum of 1.4) and
Argo CD still had to fit in the same 4 GB node. Loki plus an agent adds about 300 to 500 MiB. I
kept that memory for Task 3.

### 1.6 — Alerts: a custom PrometheusRule that fires

`DemoAppDown` uses `absent(up{job="demo-app"} == 1)`. It fires when no demo-app target is healthy,
including when there are no pods at all (a plain `up == 0` returns nothing in that case). It must
stay true for 30 s before it fires.

```text
$ kubectl apply -f monitoring/demo-alerts.yaml
prometheusrule.monitoring.coreos.com/demo-app-alerts configured
$ curl -s localhost:42090/api/v1/rules | jq -r '.data.groups[] | select(.name=="demo-app") | .rules[] | "\(.name) \(.state) \(.health)"'
DemoAppDown inactive ok
DemoAppHighMemory inactive ok
$ kubectl scale deploy/demo-app -n demo --replicas=0
deployment.apps/demo-app scaled
```

I scaled at 21:24:20 IST. The alert went pending at 21:24:54 and firing at 21:25:24 (15:55:24Z).

```text
$ curl -s localhost:42090/api/v1/alerts | jq -r '.data.alerts[] | select(.labels.alertname|startswith("Demo")) | "\(.labels.alertname) \(.state) since \(.activeAt)"'
DemoAppDown firing since 2026-10-07T15:54:54.220090206Z
$ curl -s localhost:42093/api/v2/alerts | jq '.[] | select(.labels.alertname=="DemoAppDown") | {labels, annotations, status, startsAt}'
{
  "labels": {
    "alertname": "DemoAppDown",
    "prometheus": "monitoring/kps-kube-prometheus-stack-prometheus",
    "severity": "critical"
  },
  "annotations": {
    "summary": "No healthy demo-app target in namespace demo"
  },
  "status": {
    "inhibitedBy": [],
    "mutedBy": [],
    "silencedBy": [],
    "state": "active"
  },
  "startsAt": "2026-10-07T15:55:24.220Z"
}
$ kubectl scale deploy/demo-app -n demo --replicas=2
deployment.apps/demo-app scaled
```

Prometheus evaluates the rule. Alertmanager receives the alert and would route it to Slack, email or
PagerDuty. I did not configure a receiver, so it stops at Alertmanager.

### 1.7 — Grafana

The chart provisions the Prometheus and Alertmanager data sources and 25 dashboards.
`demo-password-not-real` is the fake admin password from the values file.

```text
$ curl -s -u admin:demo-password-not-real localhost:42030/api/health
{
  "database": "ok",
  "version": "13.2.3",
  "commit": "90ffed056f0884267356c12a0eeb72a022af53f1"
}
$ curl -s -u admin:demo-password-not-real 'localhost:42030/api/search?type=dash-db' | jq -r '.[].title' | head
Alertmanager / Overview
CoreDNS
Grafana Overview
Kubernetes / API server
Kubernetes / Compute Resources /  Multi-Cluster
Kubernetes / Compute Resources / Cluster
Kubernetes / Compute Resources / Namespace (Pods)
Kubernetes / Compute Resources / Namespace (Workloads)
Kubernetes / Compute Resources / Node (Pods)
Kubernetes / Compute Resources / Nodes Overview
$ curl -s -u admin:demo-password-not-real 'localhost:42030/api/search?type=dash-db' | jq length
25
$ curl -s -u admin:demo-password-not-real localhost:42030/api/datasources | jq -r '.[] | "\(.name) \(.type) \(.url)"'
Alertmanager alertmanager http://kps-kube-prometheus-stack-alertmanager.monitoring:9093/
Prometheus prometheus http://kps-kube-prometheus-stack-prometheus.monitoring:9090/
```

## Task 2 — Observability

See [OBSERVABILITY.md](OBSERVABILITY.md).

## Task 3 — GitOps with Argo CD

Source repo: https://github.com/SammyUrfen/devops-session-20-monitoring-gitops, path `gitops/`,
branch `main`. The Application has `automated: {prune: true, selfHeal: true}`.

### 3.1 — Install Argo CD (core) and create the Application

I used `core-install.yaml`. It has no API server and no UI, which saves memory. I drive it only with
`kubectl`.

```text
$ kubectl create namespace argocd
namespace/argocd created
$ kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
...
networkpolicy.networking.k8s.io/argocd-applicationset-controller-network-policy serverside-applied
networkpolicy.networking.k8s.io/argocd-redis-network-policy serverside-applied
networkpolicy.networking.k8s.io/argocd-repo-server-network-policy serverside-applied
$ kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          72s
argocd-applicationset-controller-5c66f59556-p2cq4   1/1     Running   0          72s
argocd-redis-7f877d46b7-pbbxw                       1/1     Running   0          72s
argocd-repo-server-7bc46c4ddf-mthsb                 1/1     Running   0          72s
$ kubectl apply -f argocd/application.yaml
application.argoproj.io/session20-mini created
$ kubectl get applications -n argocd
NAME             SYNC STATUS   HEALTH STATUS
session20-mini   Unknown       Unknown
$ kubectl get application session20-mini -n argocd -o jsonpath='{.status.conditions}'
[{"lastTransitionTime":"2026-10-07T15:56:49Z","message":"Application referencing project default which does not exist","type":"InvalidSpecError"}]
$ kubectl apply -f argocd/appproject-default.yaml
appproject.argoproj.io/default created
```

### 3.2 — Initial sync

Right after the project existed, Argo CD synced commit `5e922d7` on its own:

```text
$ kubectl get applications -n argocd
NAME             SYNC STATUS   HEALTH STATUS
session20-mini   OutOfSync     Healthy
$ kubectl get deploy,pods,svc -n session20
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/session20-mini   0/1     1            0           1s

NAME                                  READY   STATUS              RESTARTS   AGE
pod/session20-mini-77c6755446-fr2c6   0/1     ContainerCreating   0          1s

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/session20-mini   ClusterIP   10.109.137.67   <none>        80/TCP    1s
$ kubectl get application session20-mini -n argocd -o jsonpath='{.status.sync.revision}{"\n"}'
5e922d7743be22c352e826c1d10e348da5aa21ef
$ git log --oneline -1
5e922d7 Add monitoring values, demo app and GitOps manifests
```

That capture caught the sync mid-way. A few seconds later:

```text
$ kubectl get applications -n argocd
NAME             SYNC STATUS   HEALTH STATUS
session20-mini   Synced        Healthy
$ kubectl get deploy,pods -n session20
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/session20-mini   1/1     1            1           26s

NAME                                  READY   STATUS    RESTARTS   AGE
pod/session20-mini-77c6755446-fr2c6   1/1     Running   0          26s
```

### 3.3 — A Git change reconciled with no kubectl

I changed only the file in Git and pushed. I did not run `kubectl apply`.

```text
$ git diff
diff --git a/gitops/deployment.yaml b/gitops/deployment.yaml
index 57f0bd5..e798f76 100644
--- a/gitops/deployment.yaml
+++ b/gitops/deployment.yaml
@@ -6,7 +6,7 @@ metadata:
   labels:
     app: session20-mini
 spec:
-  replicas: 1
+  replicas: 3
   selector:
     matchLabels:
       app: session20-mini
$ git commit -am "Scale session20-mini to 3 replicas" && git push
   3021eaf..8e108e5  main -> main
$ git log --oneline -1
8e108e5 Scale session20-mini to 3 replicas
```

Pushed at 21:36:00. Argo CD polls Git every 3 minutes by default, and it applied the change at
21:39:28:

```text
$ kubectl get application session20-mini -n argocd -o jsonpath='{.status.sync.revision}  {.status.sync.status}  {.status.health.status}'
8e108e526c2a78633a18ffe7d4ed0050bb9f9799  Synced  Progressing
$ kubectl get deploy,pods -n session20
NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/session20-mini   3/3     3            3           4m

NAME                                  READY   STATUS    RESTARTS   AGE
pod/session20-mini-77c6755446-2dtjx   1/1     Running   0          3s
pod/session20-mini-77c6755446-dqmf8   1/1     Running   0          3s
pod/session20-mini-77c6755446-fr2c6   1/1     Running   0          4m
```

### 3.4 — Self-heal: a manual change is reverted

```text
$ kubectl scale deploy/session20-mini -n session20 --replicas=1
deployment.apps/session20-mini scaled
$ kubectl get deploy session20-mini -n session20
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
session20-mini   1/1     1            1           4m
$ kubectl get deploy session20-mini -n session20   # 15 s later
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
session20-mini   3/3     3            3           4m15s
$ kubectl get applications -n argocd
NAME             SYNC STATUS   HEALTH STATUS
session20-mini   Synced        Healthy
$ kubectl get application session20-mini -n argocd -o jsonpath='{range .status.history[*]}{.id} {.revision} {.deployedAt}{"\n"}{end}'
0 5e922d7743be22c352e826c1d10e348da5aa21ef 2026-10-07T16:05:28Z
1 8e108e526c2a78633a18ffe7d4ed0050bb9f9799 2026-10-07T16:09:25Z
$ kubectl get application session20-mini -n argocd -o jsonpath='{.status.operationState.operation.initiatedBy}{" "}{.status.operationState.message}'
{"automated":true} successfully synced (all tasks run)
```

The events show the full story: the first sync, the Git change, and then a *partial* sync of the
same commit. The partial sync is self-heal: it re-applied only the Deployment I had changed by hand.

```text
$ kubectl get events -n argocd --field-selector involvedObject.name=session20-mini --sort-by=.lastTimestamp
LAST SEEN   TYPE     REASON               OBJECT                       MESSAGE
13m         Normal   ResourceUpdated      application/session20-mini   Updated sync status:  -> Unknown
13m         Normal   ResourceUpdated      application/session20-mini   Updated health status:  -> Unknown
4m34s       Normal   OperationStarted     application/session20-mini   Initiated automated sync to '5e922d7743be22c352e826c1d10e348da5aa21ef'
4m34s       Normal   ResourceUpdated      application/session20-mini   Updated sync status: Unknown -> OutOfSync
4m34s       Normal   ResourceUpdated      application/session20-mini   Updated health status: Unknown -> Missing
4m24s       Normal   ResourceUpdated      application/session20-mini   Updated health status: Missing -> Healthy
4m24s       Normal   OperationCompleted   application/session20-mini   Sync operation to 5e922d7743be22c352e826c1d10e348da5aa21ef succeeded
4m24s       Normal   ResourceUpdated      application/session20-mini   Updated sync status: OutOfSync -> Synced
...
28s         Normal   OperationStarted     application/session20-mini   Initiated automated sync to '8e108e526c2a78633a18ffe7d4ed0050bb9f9799'
28s         Normal   ResourceUpdated      application/session20-mini   Updated sync status: Synced -> OutOfSync
27s         Normal   OperationCompleted   application/session20-mini   Sync operation to 8e108e526c2a78633a18ffe7d4ed0050bb9f9799 succeeded
...
24s         Normal   OperationStarted     application/session20-mini   Initiated automated sync to '8e108e526c2a78633a18ffe7d4ed0050bb9f9799'
24s         Normal   ResourceUpdated      application/session20-mini   Updated sync status: Synced -> OutOfSync
23s         Normal   OperationCompleted   application/session20-mini   Partial sync operation to 8e108e526c2a78633a18ffe7d4ed0050bb9f9799 succeeded
23s         Normal   ResourceUpdated      application/session20-mini   Updated sync status: OutOfSync -> Synced
23s         Normal   ResourceUpdated      application/session20-mini   Updated health status: Healthy -> Progressing
20s         Normal   ResourceUpdated      application/session20-mini   Updated health status: Progressing -> Healthy
```

The history list has no new entry for self-heal, because it re-applied a commit that was already
deployed.

### 3.5 — GitOps concepts

**What GitOps is.** You keep the desired state of the system as files in Git. An agent in the
cluster makes the cluster match those files, all the time. Changes go through Git, not through
`kubectl` from a laptop.

**Git as the source of truth.** The `gitops/` folder of this repo decides what runs in namespace
`session20`. In 3.4 the cluster disagreed with Git, and Git won. Every change has an author, a
review point (pull request) and a revert (`git revert`). The Application history in 3.4 maps each
deploy to a commit SHA.

**Declarative configuration.** The files say *what* to have (`replicas: 3`), not the steps to get
there (`kubectl scale ...`). The controller works out the steps. This is what makes the comparison
in self-heal possible: there is a full desired state to compare against.

**Continuous reconciliation.** The application controller runs a loop: read Git, read the cluster,
compare, apply the difference. It found my Git change by polling (3 min default, so 3 min 28 s in
3.3). It found my manual change by watching the cluster, and reverted it in under 15 s (3.4).

**Push vs pull deployment.**

| | Push (CI runs `kubectl apply`) | Pull (GitOps agent) |
| :--- | :--- | :--- |
| Who has cluster credentials | The CI system | Only the agent inside the cluster |
| Drift (manual changes) | Stays until the next deploy | Found and reverted (self-heal) |
| Record of what runs | CI logs | Git history |
| Rollback | Re-run an old pipeline | `git revert` |

**The workflow.**

```text
developer -- edits YAML --> pull request --> review --> merge to main
                                                            |
                                         Argo CD polls Git (or a webhook)
                                                            |
                                     compare desired (Git) with live (cluster)
                                                            |
                                     OutOfSync --> apply --> Synced / Healthy
                                                            ^
                     manual kubectl change --> OutOfSync ---+  (self-heal)
```

**Kubernetes + GitOps.** Kubernetes is already declarative and already runs control loops (the
Deployment controller makes pods match `replicas`). Argo CD adds one more loop on top: it makes the
Kubernetes objects match Git. The same idea runs at two levels.

## Findings

**1. Grafana OOMKilled at 256Mi — install failed.**

```text
$ kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
NAME                           READY   STATUS             RESTARTS        AGE
kps-grafana-7ccc5c5bdc-jbp7b   2/3     CrashLoopBackOff   7 (4m40s ago)   29m
$ kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana | grep -A3 "Last State"
    Last State:      Terminated
      Reason:        OOMKilled
      Exit Code:     137
      Started:       Wed, 07 Oct 2026 21:16:10 +0530
```

Cause: Grafana 13 needs more than 256Mi at startup. Its last log lines before the kill were app
start-up lines, with no error. Fix: limit 512Mi in `values-slim.yaml`, then `helm upgrade`
(revision 2, `deployed`).

**2. Argo CD core install has no `default` AppProject.** The Application stayed `Unknown` with
`InvalidSpecError: Application referencing project default which does not exist` (3.1). In a full
install, `argocd-server` creates that project on start. The core install has no server. Fix:
`argocd/appproject-default.yaml`.

**3. cAdvisor series have no `container` label.** The common query
`container_memory_working_set_bytes{container!=""}` returns nothing here. With minikube's Docker
driver, the kubelet's cAdvisor reports only pod-level cgroups. I used `pod!=""` and `max by (pod)`.
`max` is safer than `sum`, because pod and container cgroups can both appear and `sum` would count
twice. I also fixed `DemoAppHighMemory` to use the same labels (commit `3021eaf`).

**4. node-exporter shows the host, not the minikube node.**

```text
$ curl -s localhost:42090/api/v1/query --data-urlencode 'query=(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100' | jq -r '.data.result[0].value[1]'
63.87568921546962
```

64 % matches my 24 GB Fedora host, not the 4 GB node. The minikube node is a Docker container, and
it shares the host kernel's `/proc/meminfo`. On a real VM or cloud node this metric is correct.

## Clean-up

I stopped my port-forwards, ran `minikube delete -p s20` and removed the profile's kubeconfig.
