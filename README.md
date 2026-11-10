# WR.GG Public Architecture Documentation

This repository presents the production architecture and engineering decisions
behind [WINRATES.GG](https://winrates.gg), a League of Legends companion app
with player profiles, match history, ranked ladders, champion analytics, live
game support, and advanced performance analysis.

The application is implemented across several private repositories. This
repository is intentionally a public, sanitized portfolio view rather than a
copy of the implementation. It contains no credentials, Terraform state,
internal hostnames, private URLs, or operator-only recovery instructions.

## Start Here

- [Production architecture](docs/architecture.md): the system boundary and
  the relationships between its major components.
- [Concern guide](docs/concerns.md): the map of ownership, documentation, and
  implementation repositories.
- [Backend runtime](docs/backend-runtime.md): the API, workers, queues, Riot
  requests, and operational state.
- [Frontend delivery](docs/frontend-delivery.md): the public React site, CDN,
  generated SEO content, and private dashboard image.
- [Data and analytics](docs/data-and-analytics.md): PostgreSQL, Redis, S3
  archives, ingestion, and derived product data.
- [Cloud infrastructure](docs/cloud-infrastructure.md): AWS account
  boundaries, Terraform, networking, storage, DNS, and delivery.
- [Host and cluster foundation](docs/host-and-cluster.md): Ansible, SSM,
  Tailscale, K3s, and the control-plane/worker boundary.
- [Kubernetes delivery](docs/kubernetes-delivery.md): Helm, GitOps desired
  state, Argo reconciliation, and runtime ordering.
- [Delivery and operations](docs/delivery-and-operations.md): immutable
  releases, promotion, observability, and recovery.
- [Engineering decisions](docs/decisions.md): the major tradeoffs and current
  limitations.
- [Generated infrastructure views](docs/generated-infrastructure.md): how
  private Terraform state can become a technical appendix without publishing
  sensitive state.

## How To Read This Repository

The documents are organized by engineering concern rather than by repository.
That is intentional: a reader should be able to understand frontend delivery,
for example, without first learning the entire frontend repository. Each
concern page identifies the repositories and technologies that implement it,
then explains the lifecycle, source of truth, and failure boundary.

The intended reading order is:

1. Production architecture for the system shape.
2. Concern guide for ownership and navigation.
3. The concern pages relevant to the reader's interest.
4. Delivery and operations for promotion and recovery.
5. Engineering decisions for the reasoning behind the design.

## Diagrams

The diagrams are editable Mermaid source files:

- [Production context](diagrams/production-context.mmd)
- [Production deployment topology](diagrams/production-deployment.mmd)
- [Request and data flow](diagrams/request-and-data-flow.mmd)
- [Backend runtime](diagrams/backend-runtime.mmd)
- [Frontend delivery](diagrams/frontend-delivery.mmd)
- [Data lifecycle](diagrams/data-lifecycle.mmd)
- [Cloud foundation](diagrams/cloud-foundation.mmd)
- [Host attachment and cluster foundation](diagrams/host-attachment.mmd)
- [Kubernetes delivery](diagrams/kubernetes-delivery.mmd)
- [Release pipeline](diagrams/release-pipeline.mmd)
- [Recovery model](diagrams/recovery-model.mmd)

GitHub renders Mermaid directly when the source is included in a Markdown
document. The source files are kept separately so the diagrams can also be
rendered with Mermaid-compatible tooling for a resume, presentation, or case
study page.

## Scope

Production is the reference architecture. Staging intentionally uses the same
Terraform, Kubernetes, Helm, GitOps, and release shape in a separate AWS
account. Its differences are limited to environment identity, credentials,
domains, data, and operational scale; it does not need a second architecture
manual.

The system is an early public beta, not a claim of multi-region high
availability. Its current design favors a small operating footprint, explicit
recovery paths, reproducible deployments, and clear ownership boundaries.
