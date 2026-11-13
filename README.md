# WR.GG Public Architecture Documentation

This repository presents a concise, production-focused view of the
WINRATES.GG solution and architecture. It contains no credentials or deep
product-specific implementation details, but it does walk through most of the
high-level design decisions and steady-state surfaces spanning the UI, most of
the backend code, the purpose-built rate-limiting solution for this product
([New Destiny](https://github.com/victorhaynes/new-destiny)), the hybrid-cloud
infrastructure, and the Kubernetes cluster. The design favors a small operating footprint with scalable patterns and technologies to demonstrate skills in design, operation, and integration. It is not a demonstration of a globally redundant, highly available enterprise-scale platform. The product already incurs enough cost as it is :)

Thank you for reading.

## Concern Map

| Concern | Answers | Documentation |
| --- | --- | --- |
| Backend | How does the API acquire, protect, process, and serve Riot data? | [Backend runtime](docs/backend-runtime.md) |
| Frontend | How are the public site, SEO objects, and private dashboard delivered? | [Frontend delivery](docs/frontend-delivery.md) |
| Infrastructure | What cloud, storage, host, network, and access foundations exist? | [Infrastructure](docs/infrastructure.md) |
| Kubernetes | How are cluster resources composed, scheduled, and reconciled? | [Kubernetes runtime](docs/kubernetes-runtime.md) |
| CI/CD | How does validated source become an immutable production release? | [CI/CD and release](docs/ci-cd.md) |