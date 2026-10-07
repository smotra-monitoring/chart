# Smotra Monitoring — Helm Chart

## Purpose

This repository contains the Helm chart for deploying the **smotra-monitoring** application stack to Kubernetes. It is managed by [Flux](https://fluxcd.io/) via a single `HelmRelease` resource pointed at this umbrella chart.

## Architecture: umbrella chart with sub-charts

The top-level chart (`smotra`) is an **umbrella chart** that composes three sub-charts, each representing an independently versioned, namespace-isolated component:

| Sub-chart | Namespace | What it deploys |
|-----------|-----------|-----------------|
| `frontend` | `www-smotra-net` | Web frontend (static, stateless) |
| `api` | `api-smotra-net` | API server + TimescaleDB (stateful) |
| `openapi` | `openapi-smotra-net` | OpenAPI documentation site |

Sub-charts live under [`sub-charts/`](/sub-charts) and are referenced via `file://` dependencies in [`Chart.yaml`](/Chart.yaml). Resolved versions are pinned in `Chart.lock` for reproducible deployments.

## Why sub-charts and not a single flat chart?

This has come up before. The separation is intentional and maps onto real boundaries:

- **Namespace isolation per component.** Each sub-chart owns its namespace. This keeps RBAC, network policies, and resource quotas cleanly scoped. Mixing everything into one chart with cross-namespace resources is possible but works against Kubernetes' isolation model.

- **Different lifecycles and release cadences.** The docs site (`openapi`) changes independently of the API or frontend. Sub-chart versioning lets you bump one without touching the others. The umbrella chart version advances only when the composition itself changes.

- **The `api` sub-chart intentionally bundles the database.** The TimescaleDB instance is a private dependency of the API server — not shared with other components. Keeping them in the same sub-chart reflects that cohesion and ensures they are always deployed and versioned together.

- **Controlled blast radius.** A broken template or bad value in one sub-chart does not prevent the others from rendering or reconciling.

- **Flux compatibility.** A single `HelmRelease` pointing at the umbrella chart is clean and sufficient. The `file://` sub-chart references mean sub-charts cannot be targeted independently by Flux, which is a deliberate trade-off: everything is reconciled as one unit, with sub-chart versions locked for reproducibility.

## Release workflow: compatibility matrix (`comp-matrix.yaml`)

Microservice images (frontend, api, db-migrations, openapi) are versioned and released independently in their own repositories. This creates a coordination problem: **Flux is event-driven and would pick up each image tag update as it arrives, deploying sub-charts at different times and potentially leaving the cluster in a mixed-version state that breaks the application.**

The [`comp-matrix.yaml`](/.github/workflows/comp-matrix.yaml) workflow solves this by acting as the single release gate:

1. **Fetch** — queries the latest git tag from every microservice repository via the GitHub API.
2. **Compare** — diffs each latest tag against the currently pinned tag in [`values.yaml`](/values.yaml). If nothing changed, the workflow exits early.
3. **Update atomically** — rewrites all image tags in `values.yaml` in a single step using `yq`, so the file always describes a consistent, tested combination of versions (the compatibility matrix).
4. **Bump the umbrella chart version** — increments the patch version in [`Chart.yaml`](/Chart.yaml) so Flux detects a new release.
5. **Commit and push** — one commit lands all changes together. Flux reconciles once against the new umbrella chart version, deploying all components simultaneously.

This means **all sub-charts always deploy as a matched set**, not staggered. The tradeoff is that the workflow is `workflow_dispatch` (manual trigger), which is intentional: a human decides when to cut a coordinated release rather than having components race to deploy.

### Key invariant

> `values.yaml` must always reflect a single coherent release. Never update individual image tags in `values.yaml` by hand or via separate automation — always go through `comp-matrix.yaml`.

## When to reconsider this structure

Split a sub-chart into its own independent chart (published to an OCI registry with its own `HelmRelease`) only if:

- A component needs its own Flux reconciliation schedule or source.
- A component is shared across multiple umbrella charts.
- The team owning a component is different and needs independent release control.

None of these conditions currently apply to smotra-monitoring.
