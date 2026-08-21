# PowerSync + Restate on GKE with Datadog OpenMetrics autodiscovery

Deploys PowerSync (with throwaway Postgres source + MongoDB bucket storage) and
Restate on Kubernetes, exposes their Prometheus `/metrics` endpoints, and lets the
existing Datadog node agents scrape them via per-pod OpenMetrics autodiscovery
annotations (no change to the Datadog Helm release).

- PowerSync metrics: `:9090/metrics` (enabled via `telemetry.prometheus_port`)
- Restate metrics: `:5122/metrics` (node-ctl port)

Datadog picks these up from the pod annotation `ad.datadoghq.com/<container>.checks`.
Metrics land as `powersync.*` and `restate.restate_*` (the check `namespace` prefixes
the metric names).

## Prerequisites

- `kubectl` context pointing at the target cluster (e.g. `gcloud container clusters get-credentials ...`)
- `helm` 3.8+ (OCI support)
- Datadog Agent already running in the cluster with autodiscovery enabled (default)

## Deploy PowerSync

```bash
kubectl apply -f powersync/
```

MongoDB bucket storage runs as a single-node replica set. The `mongo-rs-init` Job
initiates it on first boot. If the Mongo pod is ever recreated (storage is
ephemeral `emptyDir`), re-initiate the replica set and restart PowerSync:

```bash
MONGO=$(kubectl get pods -n powersync -l app=mongo -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n powersync "$MONGO" -- mongosh --quiet --eval \
  'try{if(rs.status().ok){quit(0)}}catch(e){} rs.initiate({_id:"rs0",version:1,members:[{_id:0,host:"mongo:27017"}]})'
kubectl rollout restart deploy/powersync -n powersync
```

## Deploy Restate

```bash
helm install restate oci://ghcr.io/restatedev/restate-helm \
  --namespace restate --create-namespace \
  -f restate/values.yaml
```

`restate/values.yaml` right-sizes the resources for a small cluster and adds the
OpenMetrics autodiscovery pod annotation.

## Verify

```bash
# Endpoints (from an in-cluster pod)
kubectl run curl-ps --rm -i --restart=Never -n powersync --image=curlimages/curl:8.10.1 \
  --command -- sh -c 'curl -s http://powersync.powersync.svc:9090/metrics | head'
kubectl run curl-rs --rm -i --restart=Never -n restate --image=curlimages/curl:8.10.1 \
  --command -- sh -c 'curl -s http://restate.restate.svc:5122/metrics | head'

# Datadog check status (run on the node agent scheduling each pod)
kubectl exec -n datadog <datadog-agent-pod> -c agent -- agent status | grep -A12 openmetrics
```

## Notes

- Storage is ephemeral (`emptyDir`); this is a metrics test, not production.
- PowerSync `client_auth` points at a placeholder JWKS URI — keys are only fetched
  when a client presents a token, so the service (and its metrics server) start fine
  without a real auth backend.
- To make scraping global instead of per-pod, enable `datadog.prometheusScrape.enabled`
  in the Datadog Helm values and use `prometheus.io/*` annotations.
