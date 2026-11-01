<p align="center"><img src="web/kubepilot-icon.svg" alt="KubePilot logo" width="96" /></p>
<h1 align="center">KubePilot</h1>
<p align="center">A Kubernetes troubleshooting assistant that combines workload evidence with operational runbooks.</p>

<p align="center">
  <a href="#reproducible-local-demo">Run the demo</a> ·
  <a href="#verified-capabilities">Capabilities</a> ·
  <a href="docs/real-cluster-readiness.md">Real-cluster notes</a> ·
  <a href="https://qloudex.tech">Qloudex</a>
</p>

## Overview

KubePilot is an open engineering project from [Qloudex](https://github.com/Qloudex),
an AI & Cloud Engineering Studio. It provides a web console and API for asking
Kubernetes operations questions, inspecting workload evidence, retrieving
Markdown runbooks, and preparing incident summaries.

The project supports an operator's judgment. It does not execute remediation
commands or replace cluster access controls. The deterministic fixture demo is
the supported, reproducible evaluation path; live-cluster and
production-shaped assets require additional configuration and validation.

## Project maturity

| Area | Status | Notes |
| --- | --- | --- |
| Local fixture demo | Working | Docker Compose runs the web console, API, Postgres, and fixture Kubernetes tool service. |
| API, retrieval, diagnosis, and reports | Working | Covered by repository tests using deterministic data and answer synthesis. |
| Local Kubernetes packaging | Experimental | Helm and kind scripts exercise an API-only deployment; see the dependency caveat below. |
| Real-cluster inspection | Experimental | Code paths exist for kubeconfig and in-cluster reads, but permissions, dependencies, and environment behavior require validation. |
| Production deployment | Not provided | The repository contains production-shaped examples, not a supported production release or managed service. |

## Verified capabilities

- Answer Kubernetes operations questions through a browser console or FastAPI
  endpoint.
- Search repository Markdown runbooks with deterministic keyword retrieval;
  optional FAISS retrieval is available through an extra dependency group.
- Inspect deployment health, related pods, events, and log excerpts through
  fixture, HTTP service, kubeconfig, or in-cluster adapters.
- Diagnose signals including unavailable replicas, failing readiness, pending
  pods, `CrashLoopBackOff`, and `ImagePullBackOff`.
- Generate evidence-based incident reports in JSON or Markdown.
- Generate remediation **plans** that require approval; KubePilot does not run
  the proposed commands.
- Expose health, readiness, Prometheus metrics, audit events, request traces,
  and an agent execution trail.
- Use deterministic answer synthesis by default, with optional HTTP,
  OpenAI-compatible, or Azure OpenAI-compatible providers configured by the
  operator.

These are implementations in this repository, not claims of production
deployment or customer outcomes.

## Architecture

```text
Browser console
      |
FastAPI API ---- authentication / policy / audit / metrics
      |
      +---- agent workflow ---- deterministic or configured LLM provider
      +---- runbook retrieval ---- keyword (default) or FAISS (optional)
      +---- Kubernetes adapter
              |-- Go fixture service (default Compose demo)
              |-- Python kubeconfig client (optional)
              `-- Python in-cluster client (optional)
      |
Postgres incident store (Compose) or in-memory store (development fallback)
```

| Path | Purpose |
| --- | --- |
| `web/` | Static browser console and nginx proxy configuration |
| `services/api/` | API, settings, auth, policy, audit, storage, metrics, and tracing |
| `agent/` | Intent routing, workflows, Kubernetes tools, and answer synthesis |
| `rag/` | Runbook loading, indexing, retrieval, and evaluation |
| `services/k8s-tool/` | Go HTTP service with fixture and cluster modes |
| `docs/runbooks/` | Troubleshooting content used by retrieval |
| `helm/kubepilot/` | API Helm chart and environment-specific examples |
| `monitoring/` | Prometheus configuration, alerts, and Grafana dashboard |

## Reproducible local demo

This demo uses built-in fixtures. It does **not** connect to a Kubernetes
cluster or cloud account.

### Prerequisites

- Git
- Docker Engine or Docker Desktop with the Compose plugin
- Ports `3000`, `8000`, `8081`, and `5432` available, unless overridden through
  the Compose port variables

```bash
git clone https://github.com/Qloudex/kubepilot.git
cd kubepilot
docker --version
docker compose version
docker compose up -d --build
```

Open <http://127.0.0.1:3000>, then try:

```text
Show unhealthy workloads
Why is checkout failing?
Why is email-worker pending?
How do I troubleshoot ImagePullBackOff?
Create an incident report for deployment checkout
```

Verify and stop the stack:

```bash
./scripts/web-smoke.sh
docker compose ps
docker compose down
```

The web console is at <http://127.0.0.1:3000>, OpenAPI documentation at
<http://127.0.0.1:8000/docs>, and API health at
<http://127.0.0.1:8000/healthz>. `docker compose down` preserves the Postgres
volume.

Optional monitoring services start with
`docker compose --profile monitoring up -d`; Prometheus uses port `9090` and
Grafana uses port `3001`. Grafana's anonymous viewer access is for the local
demo only.

## Real-cluster evaluation

Keep fixture and live-cluster testing separate. Live access can expose pod
metadata, events, and log content and must be reviewed for the target
environment.

- Local API against a cluster: install `.[dev,kubernetes]` or `.[all]` and use
  `KUBEPILOT_K8S_MODE=kubeconfig`.
- API inside Kubernetes: build an image that includes the `kubernetes` optional
  dependency and use `KUBEPILOT_K8S_MODE=in_cluster`.
- The current root `Dockerfile` installs base dependencies only. The unmodified
  image produced by `scripts/kind-demo.sh` therefore lacks the Python
  Kubernetes client required by `in_cluster` mode.
- The Helm chart and kind script deploy the API only—not the browser console,
  Postgres, Grafana, or Prometheus.

See [real-cluster readiness](docs/real-cluster-readiness.md) for the current
boundary and [local cluster validation](docs/local-cluster.md) for the working
fixture-mode Helm smoke test.

## Local development

Python 3.11 or newer is required.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[dev]"
uvicorn kubepilot_api.main:app --reload
```

Run the checks:

```bash
python -m pytest -q
python -m ruff check .
(cd services/k8s-tool && go test ./...)
```

Optional dependency groups are `kubernetes`, `rag`, `agent`, and
`observability`. `.[all]` installs every optional runtime integration.

## Configuration and security

Compose defaults are development credentials and must not be reused outside an
isolated local demo. Before network or live-cluster use:

1. Enable API-key or OIDC authentication.
2. Restrict `KUBEPILOT_ALLOWED_NAMESPACES` and `KUBEPILOT_ALLOWED_ACTIONS`.
3. Review service-account RBAC, including `pods/log`, for least privilege.
4. Store database, LLM, identity, and telemetry credentials in a secret manager.
5. Use TLS and define retention/redaction for logs, traces, audit records,
   prompts, and incident reports.
6. Review indexed runbooks for sensitive information.
7. Treat diagnoses and commands as suggestions requiring human review.

Read [SECURITY.md](SECURITY.md) and the
[production checklist](docs/production-checklist.md) before non-local use.

## Known limitations

- Fixture data is synthetic and does not establish real-cluster behavior.
- Kubernetes access is read-only by design, but can expose sensitive logs and
  events to API users.
- Retrieval is bounded by available runbooks; generated answers can be wrong or
  incomplete.
- Live-cluster paths have not been load-tested or validated as a supported
  release here.
- The repository provides no availability target, backup strategy, production
  support commitment, or supported release image.
- GitOps and environment values are reference assets with operator-owned
  registry and environment configuration.

## Documentation

- [Architecture and roadmap](ARCHITECTURE_AND_ROADMAP.md)
- [Local cluster validation](docs/local-cluster.md)
- [Real-cluster readiness](docs/real-cluster-readiness.md)
- [Production checklist](docs/production-checklist.md)
- [GitOps setup](docs/gitops.md)
- [Security policy](SECURITY.md)

## License and attribution

No license file is currently present in this repository. Source availability
alone does not grant permission to copy, modify, or redistribute the code.
Third-party dependencies remain subject to their own licenses. Repository
owners should add an explicit license before accepting or distributing reused
contributions.
