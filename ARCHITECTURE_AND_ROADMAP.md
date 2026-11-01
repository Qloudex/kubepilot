# Architecture and roadmap

This describes code currently present in KubePilot and separates implemented
behavior from plans. It is not a production architecture or delivery
commitment.

## Implemented architecture

```text
Static web console (nginx)
          |
FastAPI routes and streaming responses
          |
          +-- authentication, allowlists, rate limiting, audit, metrics
          +-- deterministic or optional LangGraph workflow orchestration
          +-- keyword or optional FAISS runbook retrieval
          +-- deterministic or configured external answer provider
          +-- Kubernetes adapter
                |-- in-process fixtures
                |-- Go HTTP tool service
                `-- optional Python Kubernetes client
          |
Postgres incident store or in-memory fallback
```

`web/` contains the static console. `services/api/` contains FastAPI endpoints
for chat, status, capabilities, runbook search, cluster evidence, diagnosis,
incident reports, remediation plans, audit events, and traces.

`agent/` routes supported intents and gathers evidence through typed tools. The
default workflow and answer provider are deterministic; LangGraph is optional.
`rag/` loads Markdown runbooks. Keyword retrieval is default and FAISS is an
optional local index. Retrieved evidence does not guarantee answer correctness.

The API supports fixture, Go service, kubeconfig, and in-cluster Kubernetes
adapters. Compose calls the Go service in fixture mode. Python live-cluster
modes require an optional dependency that the base Dockerfile does not install.
Implemented Kubernetes operations are reads; remediation endpoints generate
plans but do not execute them.

Platform assets include a fixture Docker Compose stack, an API-only Helm chart,
reference Argo CD manifests, Prometheus/Grafana configuration, and GitHub
Actions checks. Each requires environment-specific review beyond the local
demo.

## Current limitations

- No supported release image or versioned production release is published.
- The root API image lacks the optional dependency for live Kubernetes modes.
- Helm does not include the web console, database, monitoring, or Go tool.
- Fixtures cannot validate real permissions, scale, latency, or data handling.
- External models, OIDC, telemetry, secret stores, ingress, and GitOps require
  operator-owned services and configuration.
- Production availability, security assurance, recovery, multi-tenancy, and
  support commitments have not been established.

## Roadmap

### Now

- Keep the fixture demo, docs, API schema, and checks reproducible.
- Align live-cluster guidance with actual packaging and permissions.
- Expand tests around evidence, policy, and failure handling.

### Next

- Package and test a dedicated real-cluster image path.
- Add an automated kind test for read-only live-cluster inspection.
- Define deployment paths for the web console and persistent storage.
- Improve authentication, authorization, redaction, and retention tests.

### Later, after validation

- Evaluate broader workload types and evidence correlation.
- Evaluate approval workflows without unattended writes by default.
- Decide whether hosted or commercial directions are justified by user
  feedback.

Roadmap items are plans, not completed features or promised dates.
