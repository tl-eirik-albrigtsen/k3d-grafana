# k3d-grafana

## Setup
Create a [grafana cloud account](https://grafana.com/login) (maybe start cloud trial).
Generate an admin api token (it is scoped across all products).

### User Values
Save your generated values (for your own sanity):

```yaml
user:
  prometheus: 53715
  loki: 25836
  tempo: 22348
  password: MY_TOKEN=
urls:
  prometheus_remote_write: https://prometheus-us-central1.grafana.net/api/prom/push
  loki_hostname: logs-prod-us-central1.grafana.net
  tempo_endpoint: tempo-us-central1.grafana.net:443
```


### Cluster
On `k3d` 4.2.0, using kubernetes 1.20.1

```sh
mkdir ~/k3d/storage
k3d cluster create cluxfana \
  --servers 1 --agents 1 \
  -v $HOME/k3d/storage/:/var/lib/k3d/agent-k3d/storage/ \
  -v /etc/machine-id:/etc/machine-id

k3d kubeconfig get cluxfana > ~/.kube/k3d
```

### Install

```sh
k create ns monitoring
k apply -f agent.yml
k apply -f loki.yml
k apply -f tempo.yml
```

## Verifying
Go to `expore` on [clux.grafana.net](https://clux.grafana.net/explore), and verify:

- prometheus input: `topk(10, count by (__name__)({__name__=~".+"}))` has output
- loki input: `{namespace="monitoring"}` has correctly formatted output
- tempo: run an app configured against the collector


## Managing kubernetes files
Update the snapshot by utilizing scripts from [grafana agent installation](https://github.com/grafana/agent/#getting-started) giving it [#user-values](#user-values) when prompted.

```sh
NAMESPACE="monitoring" /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/grafana/agent/release/production/kubernetes/install.sh)" > agent.yml
NAMESPACE="monitoring" /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/grafana/agent/release/production/kubernetes/install-loki.sh)" > loki.yml
NAMESPACE="monitoring" /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/grafana/agent/release/production/kubernetes/install-tempo.sh)" > tempo.yml
```

### Known issues
- default loki logging plugin is `docker: {}` (in its configmap), change to `cri: {}` to get [cri log parsing](https://grafana.com/docs/loki/latest/clients/promtail/stages/cri/)
- grafana-agent-logs [crashing with machine id mounting problems](https://github.com/grafana/agent/issues/451)


### Debugging
Install and use the debugger pod:

```
k apply -f debugger.yml
kubectl exec -it deploy/debugger -- sh
apk add --no-cache bash curl
bash
curl grafana-agent-traces.monitoring.svc.cluster.local:55680
```

### Apps
Official otel demo with exemplars (doesn't work) + my rust controller with tracing:

```
k apply -f otel-demo-go.yml
k apply -f foo-crd.yml
k apply -f foo-controller.yml
```
