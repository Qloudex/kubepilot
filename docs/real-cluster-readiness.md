# Real-cluster readiness

KubePilot defaults to deterministic fixture data. Real-cluster adapters exist,
but live access is experimental and requires dependency, image, RBAC,
authentication, and data-handling review for the target environment.

## Runtime modes

| Component | Mode | Purpose | Runtime requirement |
| --- | --- | --- | --- |
| Python API | `fixture` | In-process deterministic data | Base install |
| Python API | `service` | Calls the Go Kubernetes tool service | Reachable `KUBEPILOT_K8S_SERVICE_URL` |
| Python API | `kubeconfig` | Reads a cluster from a local kubeconfig | `kubernetes` extra and readable kubeconfig |
| Python API | `in_cluster` | Reads the cluster containing the API pod | `kubernetes` extra, service account, and RBAC |
| Go tool | `fixture` | Deterministic HTTP responses | Default tool image |
| Go tool | `cluster` | Reads a live cluster with client-go | Kubeconfig or in-cluster credentials and RBAC |

The Compose demo uses Python API `service` mode and Go tool `fixture` mode. It
is not a real-cluster test.

## Current packaging boundary

The root `Dockerfile` runs `pip install .`, which installs base dependencies but
not the optional Python `kubernetes` package. The Helm chart deploys that API
image only. Setting `KUBEPILOT_K8S_MODE=in_cluster` on the image built by the
current `scripts/kind-demo.sh` therefore causes a missing-dependency error when
the Kubernetes client is initialized.

Before testing `kubeconfig` or `in_cluster`, create and validate an image that
installs `.[kubernetes]` or `.[all]`. Changing the image is outside this
documentation-only phase; do not treat the current kind script as an end-to-end
real-cluster demo. The chart also does not deploy the web console, Postgres,
Grafana, Prometheus, or Go tool service.

## Local API with kubeconfig

Use a narrowly scoped test cluster and context. Python 3.11 or newer is
required.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[dev,kubernetes]"
export KUBEPILOT_K8S_MODE=kubeconfig
export KUBEPILOT_KUBECONFIG=/absolute/path/to/test-kubeconfig
export KUBEPILOT_ALLOWED_NAMESPACES=payments,platform,notifications
export KUBEPILOT_ALLOWED_ACTIONS=cluster:health,deployment:diagnose,deployment:incident-report,deployment:remediation-plan
uvicorn kubepilot_api.main:app --reload
```

In another shell, using names that exist in the connected cluster:

```bash
curl --fail http://127.0.0.1:8000/api/v1/status
curl --fail http://127.0.0.1:8000/api/v1/cluster/health
curl --fail http://127.0.0.1:8000/api/v1/cluster/namespaces/payments/deployments/checkout/diagnose
```

Include the `x-api-key` header when API keys are configured.

## In-cluster API evaluation

After producing an image with the Kubernetes extra, load or publish it and
override the chart image explicitly:

```bash
helm upgrade --install kubepilot ./helm/kubepilot \
  --namespace kubepilot \
  --create-namespace \
  --set image.repository=REGISTRY/kubepilot-api \
  --set image.tag=IMMUTABLE_TAG \
  --set image.pullPolicy=IfNotPresent \
  --set env.KUBEPILOT_K8S_MODE=in_cluster \
  --set env.KUBEPILOT_ALLOWED_NAMESPACES=payments,platform,notifications
```

Check effective permissions before calling the API:

```bash
kubectl auth can-i list deployments --all-namespaces \
  --as=system:serviceaccount:kubepilot:kubepilot-kubepilot
kubectl auth can-i get pods --all-namespaces \
  --as=system:serviceaccount:kubepilot:kubepilot-kubepilot
kubectl auth can-i get pods --subresource=log --all-namespaces \
  --as=system:serviceaccount:kubepilot:kubepilot-kubepilot
```

Then expose only the API locally:

```bash
kubectl rollout status deployment/kubepilot-kubepilot --namespace kubepilot
kubectl port-forward service/kubepilot-kubepilot 18000:8000 --namespace kubepilot
curl --fail http://127.0.0.1:18000/healthz
curl --fail http://127.0.0.1:18000/readyz
curl --fail http://127.0.0.1:18000/api/v1/cluster/health
```

## Production-shaped chart controls

The chart includes optional ingress/TLS, autoscaling, disruption budget,
secret-backed environment variables, External Secrets, and network policy.
These are templates, not evidence of a production deployment.

Production values refer to `ghcr.io/aryansharma9917/kubepilot-api` with a
commit-style tag. Publish the referenced immutable image or override repository
and tag. Validate the destination secret store, ingress, certificate issuer,
egress policy, and telemetry endpoint.

```bash
helm template kubepilot ./helm/kubepilot \
  --namespace kubepilot \
  --values helm/kubepilot/values-production.yaml
```

## Security review

- Enable API-key or OIDC authentication before network exposure.
- Namespace/action allowlists complement rather than replace Kubernetes RBAC.
- Review cluster-wide list/watch permissions and `pods/log` access; prefer
  namespace-scoped permissions where possible.
- Assume events, logs, prompts, reports, traces, and audit data can be sensitive.
- Keep credentials out of Helm values and Git.
- Remediation plans are not executed; validate every proposed command manually.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Optional package error | Confirm the environment or image contains the `kubernetes` extra. |
| Kubeconfig cannot connect | Confirm absolute path, context, reachability, and certificate credentials. |
| In-cluster request is forbidden | Check the pod service account and effective RBAC. |
| Diagnosis returns not found | Confirm allowlist, deployment name, and cluster context. |
| Logs are missing | Confirm `pods/log` access, pod/container selection, and retention. |
| Image pull fails | Override repository, immutable tag, pull policy, and pull secret. |

For a chart smoke test without live reads, use
[fixture-mode local cluster validation](local-cluster.md).
