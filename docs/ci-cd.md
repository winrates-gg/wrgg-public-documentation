# CI/CD And Release

## High Level Overview

CI/CD turns validated source into immutable artifacts and delivers them through
the appropriate promotion path: direct web publication, infrastructure apply,
or Kubernetes desired state. It does not create the AWS foundation or directly
manage running Kubernetes resources.

## Release Lifecycle

```mermaid
flowchart LR
    BackendSource[Validated backend source commit]
    FrontendSource[Validated frontend source commit]
    DashboardSource[Validated dashboard source commit]
    ChartSource[Validated Helm source commit]
    Backend[Backend and worker image]
    Frontend[Frontend S3 release]
    Dashboard[Dashboard image]
    Chart[Helm chart release]
    Desired[Environment desired state]
    Web[Public S3 and CloudFront delivery]
    Argo[Argo CD]
    Runtime[Running production state]

    BackendSource --> Backend
    FrontendSource --> Frontend
    DashboardSource --> Dashboard
    ChartSource --> Chart
    Backend --> Desired
    Frontend --> Web
    Dashboard --> Desired
    Chart --> Desired
    Desired --> Argo
    Web --> Runtime
    Argo --> Runtime
```

The system keeps four identities separate:

- **Source history:** the code that was written and validated.
- **Artifact history:** the backend, frontend, dashboard, or chart that was
  built and published.
- **Desired state:** the exact releases an environment is requested to run.
- **Observed state:** what Argo, Kubernetes, AWS, and the private dashboard
  report about the running system.

Building an artifact does not deploy it. Selecting an artifact in desired state
does not rebuild it. Argo reconciles the selected state.

## Continuous Integration

CI validates source in the repository that owns the concern and publishes an
immutable release when that concern is ready. The frontend repository has two
release outputs: the public static site and the private dashboard image.

```mermaid
flowchart TB
    subgraph BackendRepo["Backend repository"]
        BackendCommit["Validated backend source commit"] --> BackendChecks["Repository validation"]
        BackendChecks --> BackendArtifact["Immutable API and worker image"]
    end

    subgraph FrontendRepo["Frontend repository"]
        FrontendCommit["Validated frontend source commit"] --> FrontendChecks["Repository validation"]
        FrontendChecks --> FrontendArtifact["Immutable frontend S3 release"]
        DashboardCommit["Validated dashboard source commit"] --> DashboardChecks["Repository validation"]
        DashboardChecks --> DashboardArtifact["Immutable dashboard image"]
    end

    subgraph HelmRepo["Helm chart repository"]
        ChartCommit["Validated Helm source commit"] --> ChartChecks["Chart validation and render"]
        ChartChecks --> ChartArtifact["Immutable Helm chart release"]
    end

    subgraph TerraformRepo["Terraform repositories"]
        TerraformCommit["Validated Terraform source commit"] --> TerraformChecks["Format, validation, and plan"]
        TerraformChecks --> TerraformArtifact["Immutable Terraform release"]
    end

    classDef source fill:#f2f2f2,stroke:#858585,color:#202020
    classDef checks fill:#dff1e5,stroke:#3b8f5a,color:#1d3b26
    classDef artifact fill:#e5efff,stroke:#4c78b8,color:#162a47

    class BackendCommit,FrontendCommit,DashboardCommit,ChartCommit,TerraformCommit source
    class BackendChecks,FrontendChecks,DashboardChecks,ChartChecks,TerraformChecks checks
    class BackendArtifact,FrontendArtifact,DashboardArtifact,ChartArtifact,TerraformArtifact artifact
```

The four application and chart releases are intentionally separate identities.
The backend release contains both the API and worker image, while the frontend
repository produces both the public static release and the dashboard image.
Terraform releases remain an infrastructure concern and are applied through
their environment-specific plan path rather than through Argo.

## Continuous Delivery

CD selects immutable releases and delivers them through the mechanism owned by
that concern.

```mermaid
flowchart LR
    TerraformArtifact["Immutable Terraform release"] --> TerraformApply["Environment plan and apply"] --> AWS["AWS infrastructure"]

    FrontendArtifact["Immutable frontend S3 release"] --> SiteBucket["Public site bucket"] --> CloudFront["CloudFront"]
    AWS --> CloudFront

    BackendArtifact["Immutable API and worker image"] --> Desired["GitOps environment desired state"]
    DashboardArtifact["Immutable dashboard image"] --> Desired
    ChartArtifact["Immutable Helm chart release"] --> Desired
    Desired --> Argo["Argo CD"] --> Runtime["Kubernetes runtime"]

    classDef artifact fill:#e5efff,stroke:#4c78b8,color:#162a47
    classDef delivery fill:#dff1e5,stroke:#3b8f5a,color:#1d3b26
    classDef aws fill:#e5efff,stroke:#4c78b8,color:#162a47
    classDef neutral fill:#f2f2f2,stroke:#858585,color:#202020

    class TerraformArtifact,FrontendArtifact,BackendArtifact,DashboardArtifact,ChartArtifact artifact
    class TerraformApply,SiteBucket,Argo,Runtime delivery
    class AWS,CloudFront aws
    class Desired neutral
```

Terraform applies AWS infrastructure directly. The frontend release is
published to S3 and served through CloudFront without a Kubernetes rollout.
Backend, dashboard, and Helm releases are selected together in GitOps desired
state; Argo renders the pinned chart and reconciles the resulting Kubernetes
resources.

## Repository Responsibilities

| Concern | Responsibility |
| --- | --- |
| Backend repository | API and worker code, tests, backend image releases |
| Frontend repository | React site, dashboard bundle, static publication |
| Helm chart repository | Runtime templates and chart releases |
| GitOps repository | Environment artifact pointers and desired state |
| Terraform repositories | AWS infrastructure and remote state |
| Ansible repository | On-prem host preparation and attachment |

The exact repository names, workflow commands, account identifiers, and
environment hostnames remain private. The public boundary is the important
part: application repositories publish artifacts, the GitOps repository selects
them, and Argo reconciles the selection.

## Normal Promotion

1. Validate the changed source repositories.
2. Publish immutable backend, frontend, dashboard, and chart artifacts as
   applicable.
3. Review and apply infrastructure changes separately when the Terraform plan
   requires them.
4. Update the environment's Kubernetes desired state with exact artifact tags.
5. Allow Argo to reconcile the selected releases.
6. Verify API health, static delivery, data access, queue processing, and
   relevant operational metrics.

Production is the reference promotion target. Staging follows the same release
shape with isolated credentials and data so it can validate the real path
without becoming a separate architecture.

## Rollback And Recovery

Application rollback selects an earlier immutable image, frontend release, or
Helm chart and lets the normal reconciliation path apply it. It does not
rebuild the artifact, replace the control plane, or destroy the database.

Control-plane recovery, worker reattachment, database volume adoption, and
snapshot restore are separate infrastructure operations. A destructive database
reset is explicitly guarded and is never a normal application rollback.

## Design Decisions

- GitOps selects artifacts; it does not build them.
- Static delivery is separated from Kubernetes so public traffic does not
  compete with API and worker workloads.
- PostgreSQL storage has a lifecycle separate from application deployment.
- Redis coordinates runtime behavior but is not the durable product database.
- Production and staging use the same architecture with isolated environment
  state.
- The platform favors immutable releases, explicit recovery, low operating
  cost, and visible failure boundaries over automatic multi-zone failover.
