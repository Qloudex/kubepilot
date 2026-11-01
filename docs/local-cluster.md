# Local cluster validation

KubePilot can deploy its API to a local cluster while keeping the Kubernetes
adapter in fixture mode. This validates the base API image, Helm chart, rollout,
probes, metrics, and fixture response. It does not inspect the local cluster or
deploy the web console and supporting Compose services.

## Prerequisites

- Docker
- kubectl
- Helm
- kind (recommended); without kind, the script uses the current kubectl context

Confirm the current context first. On a non-kind context, the locally built
image must be available to the cluster by another mechanism.

## One-command Smoke Test

```bash
./scripts/local-cluster-smoke.sh
```

The script:

1. Builds `kubepilot-api:local`.
2. Loads the image into kind when kind is available.
3. Renders the Helm chart.
4. Installs or upgrades the chart into the `kubepilot` namespace.
5. Waits for rollout completion.
6. Port-forwards the service.
7. Runs the API-side local-cluster validator against `/healthz`, `/readyz`, `/metrics`, and `/api/v1/cluster/health`.

## Manual Flow

```bash
docker build -t kubepilot-api:local .
kind load docker-image kubepilot-api:local
helm template kubepilot ./helm/kubepilot --namespace kubepilot
helm upgrade --install kubepilot ./helm/kubepilot \
  --namespace kubepilot \
  --create-namespace \
  --set image.repository=kubepilot-api \
  --set image.tag=local \
  --set image.pullPolicy=IfNotPresent
kubectl rollout status deployment/kubepilot-kubepilot --namespace kubepilot
kubectl port-forward service/kubepilot-kubepilot 18000:8000 --namespace kubepilot
```

Then in another shell:

```bash
curl http://127.0.0.1:18000/healthz
curl http://127.0.0.1:18000/readyz
curl http://127.0.0.1:18000/metrics
curl http://127.0.0.1:18000/api/v1/cluster/health
```

You can also run the reusable validator directly once the service is reachable:

```bash
PYTHONPATH=services/api python -m kubepilot_api.local_cluster --base-url http://127.0.0.1:18000
```

## Real-cluster boundary

Do not switch this base image directly to `in_cluster` mode. The root Dockerfile
does not install the optional Python `kubernetes` package required by that mode.
The chart includes read-only RBAC for deployments, pods, events, and `pods/log`,
but packaging and effective permissions still require validation. Follow
[real-cluster readiness](real-cluster-readiness.md) for the experimental path.
