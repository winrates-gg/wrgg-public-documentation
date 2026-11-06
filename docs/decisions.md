# Engineering Decisions

## Small Hybrid Platform Instead Of A Fully Managed Stack

The application uses an EC2-hosted K3s control plane plus an on-prem worker
instead of starting with a managed Kubernetes service and a larger cloud
footprint.

**Why:** low operating cost, direct control over the cluster, and a useful
hybrid infrastructure learning surface.

**Tradeoff:** the control plane is a real availability boundary and the
on-prem worker needs an explicit attachment and recovery procedure.

## Static Delivery Is Separate From Kubernetes

The frontend, static SEO objects, and match archives are delivered through S3
and CloudFront rather than from the API pods.

**Why:** static traffic should not compete with API and worker workloads, and
frontend releases can be promoted independently.

**Tradeoff:** cache behavior and release promotion become part of the delivery
model and require explicit invalidation or versioning decisions.

## PostgreSQL Uses Durable EBS Storage

The database volume has a lifecycle separate from the Kubernetes application.

**Why:** application teardown, control-plane replacement, and normal GitOps
reconciliation should not silently destroy the data source.

**Tradeoff:** restore and volume adoption require deliberate operator action,
and the database remains a single primary storage boundary.

## Redis Is Coordination State, Not The Database

Redis coordinates rate limits, queue state, temporary blocks, and operational
metrics. PostgreSQL remains the durable product store.

**Why:** fast shared coordination is useful across API and worker processes,
while durable facts belong in the database.

**Tradeoff:** Redis loss can affect in-flight coordination and metrics, but it
should not be treated as the only copy of application data.

## GitOps Selects Artifacts; It Does Not Build Them

Application repositories publish immutable artifacts. A separate GitOps
repository selects those artifact versions for an environment, and Argo
reconciles the selection.

**Why:** source, artifact, desired state, and observed state remain distinct.
Rollbacks become revision changes rather than rebuilds.

**Tradeoff:** promotion crosses several repositories and requires disciplined
workflow ordering.

## Explicit Rate-Limit Modeling

The application uses a shared local model of Riot's communicated rate limits,
Redis-backed admission, and bounded fallback behavior when Riot returns a
throttle that does not perfectly match the local model.

**Why:** multiple workers need a coordinated, respectful request posture.

**Tradeoff:** Riot's complete implementation is not observable, so a small
amount of bounded mismatch remains possible and is measured separately from
internal blocking.

## Production Is The Reference Architecture

Staging intentionally reproduces the production topology instead of having a
second design.

**Why:** staging validates the real deployment path while keeping test data,
credentials, and account ownership isolated.

**Tradeoff:** staging can be more expensive or operationally involved than a
minimal local test environment, but its results are more meaningful.

## Current Limitations

The system is intentionally not presented as a globally redundant platform.
Known limitations include:

- a single primary control-plane host;
- one on-prem worker attachment boundary;
- a primary database volume rather than a multi-writer database;
- manual recovery decisions for database restoration;
- a currently limited launch scope rather than all Riot regions.

These are documented tradeoffs for an early beta, not hidden implementation
details.

