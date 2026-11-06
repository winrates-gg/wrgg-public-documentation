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

- [Production architecture](docs/architecture.md): the system, its boundaries,
  and the reason for its major components.
- [Delivery and operations](docs/delivery-and-operations.md): how code becomes
  an immutable production release and how the system is operated.
- [Engineering decisions](docs/decisions.md): the main tradeoffs and current
  limitations.
- [Generated infrastructure views](docs/generated-infrastructure.md): how the
  private Terraform state is turned into a technical infrastructure diagram
  without publishing sensitive state.

## Diagrams

The diagrams are editable Mermaid source files:

- [Production context](diagrams/production-context.mmd)
- [Production deployment topology](diagrams/production-deployment.mmd)
- [Request and data flow](diagrams/request-and-data-flow.mmd)
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

