# Kubernetes - Cluster Design, Resources, Networing, and Security

## Purpose

Kubernetes (K3s cluster) is the container orchestration layer for WR.GG. It turns
immutable images and Helm templates into a running cluster, schedules workloads
across hybrid-cloud nodes (AWS and an On-Prem data cener), and continually works to keep cluster state aligned
with desired state.

The responsibilities are split across multiple concerns:

- Terraform and Ansible provide the cloud, host, network, storage, and access
  foundations.
- `INTERNAL_REPOSITORY_1` defines Helm charts, defining the Kubernetes resources and their relationships.
- `INTERNAL_REPOSITORY_2` supplies the desired state and release selections.
- EC2 user data provides the initial bootstrap gateway: it installs K3s and Argo
  CD, clones the desired-state repository, and applies the root Argo resources.
- Argo CD then renders the selected Helm sources and reconciles them into the
  cluster.

This document explains the primary cluster resource model and runtime behavior. The
`backend` document explains application request and data flows; the CI/CD
document explains how artifacts and desired state are published.

### Diagram Legend

The diagrams in this document use the shared architecture palette:

| Color | Meaning |
| --- | --- |
| Gray | Neutral actors, boundaries, orchestration, and configuration concepts |
| Green | Highlighted WR.GG application code/workload runtime (AWS or on-prem; see [Infrastructure](infrastructure.md)) |
| Blue | AWS-hosted infrastructure |
| Red | External services outside WR.GG's ownership |

## Bootstrap Gateway And Reconciliation

The cluster has a one-time bootstrap path and a separate steady-state
reconciliation path. They are related, but they are not the same mechanism.

During control-plane bootstrap, EC2 user data downloads the full bootstrap
script from the environment's S3 bootstrap bucket, installs and starts K3s,
retrieves the required credentials from SSM, and clones the GitHub desired-state
repository to the control-plane host. It then uses `kubectl` to apply the Argo
CD installation, repository credentials, and the environment's root
`Application`.

This is the initial gateway that makes Argo possible. The local repository clone
is a bootstrap workspace containing manifests; it is not the application runtime
and it is not the ongoing deployment engine. EC2 user data does not render the
full application itself.

Once Argo CD and the root `Application` exist, Argo's repo-server fetches the
remote desired-state repository itself. The root `Application` loads the
environment overlay, which creates the `ApplicationSet` resources. Those
`ApplicationSet` resources generate the bootstrap and runtime Argo
`Application`s, and Argo renders the pinned Helm charts and applies the
resulting Kubernetes resources.

Normal application changes therefore do not require EC2 to clone or apply the
repository again. CI updates the desired state or release selection in GitHub,
and Argo reconciles the existing cluster. Replacing the control-plane host
repeats the bootstrap gateway, after which Argo reconstructs the selected
cluster state from the remote repository.

## Cluster Network Boundary

This first diagram is a highlevel networking overview. It shows how public API traffic and
private operator traffic enter the cluster, the primary Services receive that
traffic, and which runtime dependencies those workloads use. It is not a
complete inventory of Kubernetes resources. See next diagram.

```mermaid
flowchart LR
    Public[Public API traffic]
    PublicIngress[K3s public ingress Traefik controller]
    BackendService[backend-svc]
    API[Backend API Deployment]

    Operator[Private/Interal Operator]
    PrivateIngress[Private ingress Traefik controller]
    DashboardService[dashboard-svc]
    DashboardAPIService[dashboard-api-svc]
    Dashboard[Dashboard Deployment]
    DashboardAPI[Dashboard API Deployment]

    Redis[(Redis Deployment)]
    PostgreSQL[(PostgreSQL)]
    Riot[Riot API]

    Public --> PublicIngress --> BackendService --> API
    Operator --> PrivateIngress
    PrivateIngress --> DashboardService --> Dashboard
    PrivateIngress --> DashboardAPIService --> DashboardAPI

    API --> Redis
    API --> PostgreSQL
    API --> Riot
    DashboardAPI --> Redis
    DashboardAPI --> PostgreSQL

    classDef neutral fill:#f2f2f2,stroke:#858585,color:#202020
    classDef app fill:#dff1e5,stroke:#3b8f5a,color:#1d3b26
    classDef aws fill:#e5efff,stroke:#4c78b8,color:#162a47
    classDef external fill:#fde5e5,stroke:#c54b4b,color:#4a1717

    class Public,Operator neutral
    class PublicIngress,BackendService,API,PrivateIngress,DashboardService,DashboardAPIService,Dashboard,DashboardAPI,Redis app
    class PostgreSQL aws
    class Riot external
```

The public API uses the cluster's public ingress path, largely standard & bundled with K3s, and routes through a
ClusterIP Service to the backend API pod(s). WR.GG has a supporting traffic & health private dashbaord that is accessed through a separate
private IngressClass and private ingress controller.

Redis is runtime coordination & state management system
for rate limiting, Riot Games (external) API Health observations, and dashboard metrics. PostgreSQL is
durable product data and runs outside of the cluster. S3 is the durable object boundary for match
archives, static assets, releases, and generated SEO content. These external
systems are shown here only to clarify the cluster boundary; their ownership
and lifecycle are documented in [Infrastructure](infrastructure.md).

## Cluster Resource Composition

The network diagram explains how traffic moves. This is the primary
cluster-design view. It shows the initial EC2 bootstrap gateway, the Argo
resources that take over reconciliation, and how the Helm layers render into
the cluster.

```mermaid
flowchart TB
    BootstrapGateway[EC2 Control Plane user-data bootstrap]
    Clone[One-time Git clone]
    Desired[GitHub desired-state repository]
    RootApply[kubectl applies Argo CD + root Application]
    Argo[Argo CD reconciliation loop]

    BootstrapGateway --> Clone --> RootApply --> Argo
    Desired --> Clone
    Desired --> Argo
    Argo -.-> Desired

    BootstrapSet[Bootstrap ApplicationSet]
    RuntimeSet[Runtime ApplicationSet]
    Argo --> BootstrapSet
    Argo --> RuntimeSet

    subgraph BootstrapLayer[BOOTSTRAP]
        direction LR
        BootstrapChart[Bootstrap Helm chart]
        subgraph BootstrapResources[Bootstrap resources]
            direction LR
            IngressController["Deployment: private-ingress-controller"]
            IngressResources["IngressClass + private-argocd-ingress"]
            AccessResources["ServiceAccount + RBAC + admission policy"]
            DNSResource["ConfigMap: CoreDNS customization"]
        end
        BootstrapChart --> IngressController
        BootstrapChart --> IngressResources
        BootstrapChart --> AccessResources
        BootstrapChart --> DNSResource
    end

    subgraph InitLayer[INIT]
        direction LR
        InitChart[Init Helm chart]
        subgraph InitResources[Initialization resources]
            direction LR
            Migration["Job: database-migration"]
            Seed["Job: database-seed"]
            DataInit["Job: database-data-init"]
            AssetInit["Jobs: initial asset refreshes"]
        end
        InitChart --> Migration
        InitChart --> Seed
        InitChart --> DataInit
        InitChart --> AssetInit
    end

    subgraph RuntimeLayer[RUNTIME]
        direction LR
        RuntimeChart[Runtime Helm chart]
        subgraph RuntimeResources[Runtime resources]
            direction LR
            Config["ConfigMap: backend-config"]
            RedisResource["Deployment + Service: Redis"]
            APIResource["Deployment + Service + Ingress: backend API"]
            MatchWorkers["Deployments: match worker per platform"]
            LadderWorkers["Deployments: ladder worker per region"]
            Recurring["CronJobs -> Jobs: data, assets, archives, analytics, SEO"]
            DashboardResources["Deployments + Services + Ingress: dashboard"]
        end
        RuntimeChart --> Config
        RuntimeChart --> RedisResource
        RuntimeChart --> APIResource
        RuntimeChart --> MatchWorkers
        RuntimeChart --> LadderWorkers
        RuntimeChart --> Recurring
        RuntimeChart --> DashboardResources
    end

    BootstrapSet --> BootstrapChart
    RuntimeSet --> InitChart
    RuntimeSet --> RuntimeChart

    classDef neutral fill:#f2f2f2,stroke:#858585,color:#202020
    classDef app fill:#dff1e5,stroke:#3b8f5a,color:#1d3b26
    classDef aws fill:#e5efff,stroke:#4c78b8,color:#162a47

    class Clone,Desired,RootApply,Argo,BootstrapSet,RuntimeSet,BootstrapChart,InitChart,RuntimeChart neutral
    class BootstrapGateway aws
    class IngressController,IngressResources,AccessResources,DNSResource,Config,Migration,Seed,DataInit,AssetInit,APIResource,MatchWorkers,LadderWorkers,Recurring,DashboardResources,RedisResource app
```

The diagram reads from top to bottom as the Argo control plane, bootstrap, init,
and runtime. The two ApplicationSets are peers above the resource bands. Each
band then reads left to right from its Helm chart into the resources it renders;
the runtime ApplicationSet supplies both the init and runtime chart paths.

**Cluster creation**

- EC2 user data installs K3s, creating the cluster, then
  clones the desired-state repository and uses `kubectl` to apply the Argo CD
  installation and root `Application`.

**Reconciliation**

- Argo CD takes over and continuously compares the
  remote desired-state repository with the observed cluster, reconciling the
  difference. It does not write changes back to GitHub.

**Argo CD ApplicationSets**

The desired-state repository owns two ApplicationSets:

- **Bootstrap ApplicationSet:** Generates the Argo application for cluster
  support.
- **Environment ApplicationSet:** Generates the Argo application that selects
  the initialization and runtime Helm sources.

**Helm charts**

The Helm chart repository owns three charts. The desired-state repository
selects their immutable release versions and environment values:

- **Bootstrap chart:** Supports the private Tailscale-connected operator path,
  including private ingress, Argo access, RBAC, CoreDNS customization, and
  cluster security policy.
- **Init chart:** Renders finite Jobs that prepare the database and load
  external catalog data needed by the main application.
- **Runtime chart:** Renders the main cluster resources and operating patterns:
  the API, Redis, workers, dashboard, Services, Ingresses, ConfigMaps, and
  recurring CronJobs.

## Namespaces And Foundations
| Namespace | Main responsibility |
| --- | --- |
| `argocd` | Argo CD (defaulted by Argo CD) and the private Argo ingress surface |
| `private-ingress` | The private Traefik controller, its ServiceAccount, and cluster-scoped read permissions |
| `staging` or `production` | The application ConfigMap, Secrets, Services, Deployments, initialization Jobs, and recurring CronJobs |
| `kube-system` | K3s/K8s (defaulted) system services, including the CoreDNS customization used by the private network |

## Workload Model

The runtime is composed from a small set of Kubernetes resource types. Each type
has a different lifecycle and should not be treated as interchangeable.

| Resource type | WRGG role | Lifecycle |
| --- | --- | --- |
| `Deployment` | Backend API, Redis, per-platform match workers, per-region ladder workers, dashboard, and dashboard API | Continuously ingest fresh data and serve cached product data |
| `Job` | Database migration, seed, data initialization, and initial asset refresh | Ensure PostresQL Database readiness |
| `CronJob` | Keep catalogs constantly up to date: data refresh, icon refresh, archive reconciliation and cleanup, batched analytics, and SEO publication | Creates finite Jobs on a schedule; overlapping runs are forbidden |
| `Service` | Stable in-cluster names for the backend, Redis, dashboard, and dashboard API | ClusterIP routing to selected pods |
| `Ingress` | Public API routing and private dashboard/Argo routing | HTTP boundary from the relevant cluster entrypoint to a Service |
| `ConfigMap` | Non-secret environment identity, deployment scope, runtime policy, and tuning | Shared configuration rendered by Helm |
| `Secret` | Riot credentials, database credentials, and other sensitive runtime values | Mounted or injected only into workloads that need them |

The worker image is shared by the match worker, ladder worker, initialization
Jobs, and CronJobs. The API image is used by the API, database migration, seed,
and dashboard API workloads. These are separate runtime roles even though they
are published from the same backend release by design for the sake of simplicity.

## Helm Layers

The helm chart repository spans 3 concerns:
1. Cluster prerequisites
2. Idempotent datanase initialization
3. Steady-state runtime

### 1. Bootstrap Chart

The bootstrap chart installs resources that support the cluster and private
operator path:

- private Traefik ingress controller;
- private `IngressClass` and Argo ingress;
- ServiceAccount, ClusterRole, and ClusterRoleBinding for ingress discovery;
- CoreDNS customization for the private network; and
- cluster admission policies for AWS credential mounts.

The bootstrap ApplicationSet is reconciled before the environment application.
It is cluster support, not an application release.

### 2. Initialization Chart

The initialization chart renders finite Jobs using the exact backend release
selected by `INTERNAL_REPOSITORY_1`:

- database migration;
- database seed;
- partition and reference-data initialization; and
- initial icon refreshes.

These Jobs are intentionally separate from the long-running runtime. A failed
initialization Job is a failed prerequisite to investigate, not a reason to
start an incomplete API stack and hope that later reconciliation repairs it.

### 3. Runtime Chart

The runtime chart renders the shared ConfigMap, Redis, API, workers, dashboard,
Services, Ingress resources, and recurring CronJobs. Runtime values explicitly
select the environment identity, image repositories and tags, deployment scope,
New Destiny settings, data retention policy, worker topology, and dashboard
state.

The chart has no useful production or staging fallback for these values. Empty
required values fail rendering rather than silently selecting the wrong
environment or rate-limit profile.

## Security Boundaries

Security is layered around private operator access, application secrets, and
AWS credential mounts:

- **Private operator path:** Argo CD and the operations dashboard use the
  private IngressClass and Tailscale-connected access path. The public API uses
  a separate ingress path.
- **Application secrets:** Riot, PostgreSQL, and other runtime credentials are
  delivered through the environment's `backend-secrets` Secret and injected
  only into workloads that require them.
- **AWS credentials:** S3-facing initialization Jobs, match workers, archive
  and asset Jobs, and SEO Jobs use the `winrates-gg-aws-credentials`
  ServiceAccount. They receive a read-only mount of the node-local
  `/root/.aws/credentials` file and use `AWS_SHARED_CREDENTIALS_FILE` to find
  it.
- **Admission enforcement:** The Bootstrap chart installs a denying
  `ValidatingAdmissionPolicy` for Pods, Deployments, StatefulSets, DaemonSets,
  Jobs, and CronJobs. In the `staging` and `production` namespaces, a workload
  that mounts `/root/.aws` must use the dedicated AWS-credentials
  ServiceAccount. Other workload identities are rejected at admission.

## Scheduling And Placement

Placement is deliberate because the cluster is hybrid and the node classes have
different operational importance:

| Placement | Workloads | Reason |
| --- | --- | --- |
| `control-plane` | Backend API, Redis, database migration/seed/data initialization, dashboard, dashboard API, private ingress | Core availability and control-plane locality |
| `on-prem` | Match workers, ladder workers, initial asset Jobs, recurring data/icon/archive/analytics/SEO Jobs | Non-core throughput and scheduled processing |

The chart expresses this through required node affinity and matching
tolerations. It does not allow a worker workload to drift onto an unintended
node class. The optional on-prem node can be detached without taking the core
API and Redis path with it, but background throughput, refreshes, and new
initialization work will be degraded or paused.

The deployment scope is regional. `deploymentScope.regions` selects the
enabled regions; the chart derives the platform/server list from its catalog.
It renders one match-worker Deployment per selected platform and one
ladder-worker Deployment per selected region. The current production posture
uses one ladder worker per region and an explicit match-worker replica count
per platform. This is implemented to support WR.GG rollout to different reasons. Like-product users tend be most concerned with data that pertains to their home platform (server, i.e. "NA1" platform representing Canada, The United States of America, and Mexico inside of the "americas" region)

## Initialization And Sync Order

Argo uses separate bootstrap and runtime Applications, then uses sync waves to
make dependencies visible during a fresh or updated environment:

```text
bootstrap Application       wave 0
runtime ConfigMap            wave 0
runtime ApplicationSet       wave 1
database migration           wave 10
database seed                wave 20
partitions and row data      wave 30
initial asset refreshes      waves 40-50
Redis                        wave 60
long-running workers         wave 70
API and dashboard            wave 80
recurring CronJobs           wave 90
```

The important contract is not the exact number assigned to every resource. The
contract is that schema and reference data precede workloads, Redis is present
before components that depend on it, long-running workers start after their
prerequisites, and recurring jobs are not confused with initialization.
The backend and worker pods also use Redis readiness gates during startup. This
protects them from starting their application process before the Redis Service
is reachable.

## Failure Boundaries

The runtime intentionally has visible failure boundaries:

- A failed API Deployment affects public API availability but does not require
  rebuilding the chart or replacing the cluster host.
- A failed worker or unavailable on-prem node delays background processing and
  freshness while the control-plane API path can remain available.
- A failed initialization Job blocks the corresponding startup path until its
  image, configuration, dependency, or data issue is corrected.
- A failed CronJob affects one scheduled concern and is retained in Kubernetes
  history for diagnosis; `concurrencyPolicy: Forbid` prevents overlapping runs
  from silently competing.
- Redis loss affects coordination and rate limiting; PostgreSQL remains the
  durable product database, but the application is not designed as a fully
  independent Redis-free runtime.
- A desired-state change can be synced while the application remains degraded;
  Argo sync status and workload health must both be checked.

The current shape is a production-like early-beta cluster, not a claim of
multi-zone high availability. Core services are intentionally small and
control-plane placed. Recovery of the control-plane host, worker attachment,
PostgreSQL storage, or snapshots belongs to the infrastructure recovery path,
not to an application image rollback.

## Desired State And Release Boundary

```mermaid
flowchart LR
    Artifacts[Immutable backend, dashboard, and chart releases]
    Values[Environment values and deployment scope]
    Live[INTERNAL_REPOSITORY_1 desired state]
    Argo[Argo CD ApplicationSets]
    Helm[Helm bootstrap, init, and runtime charts]
    Cluster[Observed Kubernetes state]

    Artifacts --> Live
    Values --> Live
    Live --> Argo --> Helm --> Cluster
```

The environment deployment file records the exact cluster, backend, chart,
and dashboard releases. A normal application change updates those pointers and
lets Argo reconcile the existing cluster. It does not rebuild an image,
replace the control plane, or destroy PostgreSQL storage.

Rollback is the inverse operation: select a previously published immutable
release and reconcile it through the same path. Control-plane replacement,
database volume adoption, snapshot restore, and destructive reset are separate
infrastructure operations.