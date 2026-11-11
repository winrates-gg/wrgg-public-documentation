# Backend Runtime

## High Level Overview

At its core the WINRATES.GG (WR.GG) API, or `backend` implements a read-through cache that sits between UI application users and the various Riot Games APIs for League of Legends. WR.GG also augments Riot payloads and implements product- and domain-specific data ingesting, data cleaning, data normalization, data analysis, and data storage patterns. Beyond the core Riot APIs WR.GG also relies on with supporting Riot and third-party services such as Community Dragon which provide critical and ever-changing foundational information. Much of this automatic upkeep code uses the same data models and patterns as the API, so that logic is defined and bundled here as well.

## Primary Technology
The core API is written in the Python web framework FastAPI. FastAPI was selected when WR.GG was migrated out of Django because its async support and syntax were simpler for this workload. WR.GG relies on more than over 1 million external API calls per day to hydrate the baseline product, some of these calls happen concurrently inside the HTTP request-response cycle so the simplicity of FastAPI async path (endpoint) functions made it more attractive.


## Portfolio Architecture Views

The following three views separate runtime context from request sequencing and
background ingestion so that each diagram answers one architectural question.

### Diagram: Runtime Topology And Boundaries

This is a context view. The arrows show component relationships, not one
specific request's full execution order.

```mermaid
flowchart LR
    Browser[Browser / UI User]
    CloudFront[CloudFront CDN]
    APICloudFront[CloudFront API CDN]
    PublicSite[(S3 React Build)]
    APIIngress[K3S Cluster Ingress]
    API[FastAPI API]
    Deployments[Kubernetes Deployments]
    CronJobs[Kubernetes CronJobs]
    MatchWorker[Match queue workers]
    LadderWorker[Ladder and refresh workers]
    Jobs[Scheduled data and maintenance jobs]
    NewDestiny[New Destiny]
    Redis[(Redis coordination state)]
    Postgres[(PostgreSQL product data)]
    Archives[(S3 match archives)]
    Assets[(S3 assets and published artifacts)]
    Riot[Riot API]
    CommunityDragon[CommunityDragon]
    DataDragon[Data Dragon]

    Browser --> CloudFront
    CloudFront --> PublicSite
    Browser --> APICloudFront
    APICloudFront --> APIIngress
    APIIngress --> API

    Deployments -.-> API
    Deployments -.-> MatchWorker
    Deployments -.-> LadderWorker
    CronJobs -.-> Jobs

    API <--> NewDestiny
    MatchWorker <--> NewDestiny
    LadderWorker <--> NewDestiny
    NewDestiny <--> Redis
    NewDestiny <--> Riot

    API <--> Postgres
    MatchWorker <--> Postgres
    MatchWorker --> Archives
    LadderWorker --> Postgres
    Jobs <--> CommunityDragon
    Jobs <--> DataDragon
    Jobs --> Postgres
    Jobs --> Assets
    Jobs --> Archives

    classDef user fill:#f2f2f2,stroke:#858585,color:#202020
    classDef platform fill:#f2f2f2,stroke:#858585,color:#202020
    classDef backend fill:#dff1e5,stroke:#3b8f5a,color:#1d3b26
    classDef aws fill:#e5efff,stroke:#4c78b8,color:#162a47
    classDef external fill:#fde5e5,stroke:#c54b4b,color:#4a1717

    class Browser user
    class Deployments,CronJobs,APIIngress platform
    class API,MatchWorker,LadderWorker,Jobs,NewDestiny backend
    class CloudFront,APICloudFront,PublicSite,Redis,Postgres,Archives,Assets aws
    class Riot,CommunityDragon,DataDragon external
```

This view establishes the major boundaries: public delivery, cluster workload
execution, protected Riot access, coordination state, and durable outputs. It
intentionally does not explain `update`, stale fallback, queue admission, or
individual response paths.

### Diagram: Synchronous API Flow

This sequence view is initiated by the browser. Read each section from top to
bottom. `alt` means that exactly one branch occurs; the shaded blocks make the
alternative request modes and outcomes explicit. The first-level alternatives
share one blue-gray fill; the nested outcome blocks share the next color tier.
Within that tier, green identifies admission and yellow identifies local
blocking.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"background": "#0f172a", "primaryColor": "#172033", "primaryTextColor": "#f8fafc", "primaryBorderColor": "#94a3b8", "lineColor": "#cbd5e1", "actorBkg": "#111827", "actorBorder": "#cbd5e1", "actorTextColor": "#f8fafc", "actorLineColor": "#64748b", "signalColor": "#cbd5e1", "signalTextColor": "#f8fafc", "labelBoxBkgColor": "#1e293b", "labelBoxBorderColor": "#94a3b8", "labelTextColor": "#f8fafc", "noteBkgColor": "#334155", "noteBorderColor": "#cbd5e1", "noteTextColor": "#f8fafc"}}}%%
sequenceDiagram
    actor User as Browser / user
    participant API as FastAPI
    participant ND as New Destiny
    participant Redis as Redis
    participant Riot as Riot API
    participant DB as PostgreSQL
    participant Queue as match_build_queue

    Note over User,Queue: SYNCHRONOUS USER REQUEST
    rect rgb(23, 32, 51)
        alt ALTERNATIVE: update=False
            rect rgb(30, 50, 78)
                User->>API: Request existing profile
                API->>DB: Read durable state
                DB-->>API: Existing profile data
                API-->>User: Current stored response
            end
        else ALTERNATIVE: update=True
            rect rgb(30, 50, 78)
                User->>API: Request refresh
                API->>ND: Protected Riot request
                ND->>Redis: Preflight admission
                Redis-->>ND: Admit or block

                alt OUTCOME: admitted
                    rect rgb(25, 76, 52)
                        ND->>Riot: Riot API request(s)
                        Riot-->>ND: Fresh Riot data
                        ND-->>API: Fresh result
                        API->>DB: Persist refreshed profile state
                        API->>Queue: Enqueue missing match IDs
                        API-->>User: Fresh profile response
                    end
                else OUTCOME: internally blocked
                    rect rgb(82, 69, 24)
                        ND-->>API: Rate-limit exception
                        API->>DB: Read existing profile state
                        DB-->>API: Stale profile data
                        API-->>User: Cached/partial response and retry-after
                    end
                end
            end
        end
    end
```

The stale-data branch is owned by FastAPI, not New Destiny: New Destiny uses
Redis to make the admission decision and raises the exception; the API handler
then reads PostgreSQL and decides what response the user receives. If no
minimum durable profile exists, the API returns the initial-build failure
instead of a stale response.

### Diagram: Background Processing Flow

This sequence view has two kinds of initiator. The Kubernetes controller keeps
long-running worker Deployments alive; Kubernetes CronJobs start finite
scheduled jobs. These paths do not require a browser request. All first-level
background paths share one blue-gray fill, while nested outcomes share the next
color tier. Within that tier, green identifies admission and yellow identifies
local blocking.

```mermaid
%%{init: {"theme": "base", "themeVariables": {"background": "#0f172a", "primaryColor": "#172033", "primaryTextColor": "#f8fafc", "primaryBorderColor": "#94a3b8", "lineColor": "#cbd5e1", "actorBkg": "#111827", "actorBorder": "#cbd5e1", "actorTextColor": "#f8fafc", "actorLineColor": "#64748b", "signalColor": "#cbd5e1", "signalTextColor": "#f8fafc", "labelBoxBkgColor": "#1e293b", "labelBoxBorderColor": "#94a3b8", "labelTextColor": "#f8fafc", "noteBkgColor": "#334155", "noteBorderColor": "#cbd5e1", "noteTextColor": "#f8fafc"}}}%%
sequenceDiagram
    actor Controller as Kubernetes controller
    actor Cron as Kubernetes CronJob
    participant MatchWorker as Match worker
    participant Queue as match_build_queue
    participant LadderWorker as Ladder worker
    participant Jobs as Scheduled jobs
    participant ND as New Destiny
    participant Redis as Redis
    participant Riot as Riot API
    participant DB as PostgreSQL
    participant CDragon as CommunityDragon
    participant DDragon as Data Dragon
    participant S3 as S3 buckets

    Note over Controller,S3: INDEPENDENT BACKGROUND PROCESSING
    rect rgb(23, 32, 51)
        par DEPLOYMENT-MANAGED: match queue drain
            rect rgb(30, 50, 78)
                Controller->>MatchWorker: Keep worker running
                MatchWorker->>Queue: Claim and update rows
                MatchWorker->>ND: Protected match request
                ND->>Redis: Check admission and blocks
                Redis-->>ND: Admit or reject
                alt admitted locally
                    rect rgb(25, 76, 52)
                        ND->>Riot: Request match data
                        Riot-->>ND: Match data or external 429
                        ND-->>MatchWorker: Result or exception
                        MatchWorker->>DB: Persist normalized match data
                        MatchWorker->>S3: Write match archive
                    end
                else blocked locally
                    rect rgb(82, 69, 24)
                        ND-->>MatchWorker: Internal rate-limit exception
                    end
                end
            end
        and DEPLOYMENT-MANAGED: ladder refresh
            rect rgb(30, 50, 78)
                Controller->>LadderWorker: Keep worker running
                LadderWorker->>ND: Protected ladder request
                ND->>Redis: Check admission and blocks
                Redis-->>ND: Admit or reject
                alt admitted locally
                    rect rgb(25, 76, 52)
                        ND->>Riot: Request ladder data
                        Riot-->>ND: Ladder data or external 429
                        ND-->>LadderWorker: Result or exception
                        LadderWorker->>DB: Persist ladder snapshot
                    end
                else blocked locally
                    rect rgb(82, 69, 24)
                        ND-->>LadderWorker: Internal rate-limit exception
                    end
                end
            end
        and CRONJOB-STARTED: catalog and analytics work
            rect rgb(30, 50, 78)
                Cron->>Jobs: Start scheduled processing
                Jobs->>CDragon: Read upstream catalog data
                CDragon-->>Jobs: Catalog response
                Jobs->>DDragon: Read upstream catalog data
                DDragon-->>Jobs: Catalog response
                Jobs->>DB: Write normalized data
                Jobs->>S3: Write assets or published artifacts
            end
        end
    end
```

The queue is the handoff between request-driven discovery and asynchronous
match processing. The controller keeps the worker alive but does not drain the
queue itself. CronJobs independently start scheduled work; those jobs consume
CommunityDragon and Data Dragon inputs and write normalized results to
PostgreSQL or S3.

## Core API

The core API is the synchronous product surface. FastAPI validates requests,
applies deployment-scope rules, reads durable or derived data from PostgreSQL,
coordinates shared state through Redis, and uses protected Riot API requests
when current external data is required.

The API uses a read-through pattern for current player data. A profile can be
served from the latest durable record, or refreshed from Riot when the request
requires an authoritative update. In the profile route, a durable read is
returned with the WRGG contract status `203`; a successful fresh build is
returned with `201`. Players can request a refresh when their profile has not
otherwise been updated by background work. The current profile record includes
mutable facts such as the Riot ID, profile icon, level, and ranked overview.

Match history follows a different lifecycle. The current state of a player can
change frequently, but the outcome and participant facts for a completed match
such as `NA1_5676149852` should not change. The API therefore serves existing
match history and analytics from durable state, while missing match data can be
recorded for asynchronous ingestion. This keeps request latency separate from
the work of fetching, archiving, normalizing, and analyzing matches.

The API is deliberately not responsible for completing every expensive piece
of work before responding. When a request requires match ingestion or another
long-running update, it can return the best available profile or analytics
state while recording work for the worker system to complete.

## Data Lifecycle And Persistence

PostgreSQL is the main structured product persistence boundary. S3 provides
durable object storage for match archives and published static artifacts. Redis
is shared coordination state: it supports request admission, temporary
rate-limit blocks, queue and dashboard observations, and other short-lived
coordination. It is not the durable product database.

The backend generally follows this lifecycle:

1. A request resolves the relevant platform, player, champion, patch, tier, and
   role context.
2. The API reads the best available durable record and determines whether the
   request can be satisfied immediately or needs fresh external data.
3. Protected Riot requests and worker jobs obtain external data when required.
4. Match archives are retained in S3, while normalized entities and derived
   analytics are written to PostgreSQL for product queries.
5. A later API request reads the persisted result instead of repeating the
   entire external fetch.

The resulting system is intentionally mixed: current identity/profile state is
refreshed, completed match facts are retained, and analytics are recomputed or
published from those durable facts as the relevant patch and cohort data change.

## Core Data Model

The backend schema is larger than this diagram. The view below shows the
primary product relationships rather than every catalog, rune, item, queue,
analytics, and operational table.

```mermaid
erDiagram
    REGION ||--o{ PLATFORM : contains
    PLATFORM ||--o{ SUMMONER : hosts
    PLATFORM ||--o{ MATCH : serves
    SUMMONER ||--o{ SUMMONER_MATCH : appears_in
    MATCH ||--o{ SUMMONER_MATCH : contains
    CHAMPION ||--o{ SUMMONER_MATCH : played
    MATCH }o--|| GAME_MODE : uses
    MATCH }o--|| PATCH : belongs_to
    MATCH }o--|| TIER : reports_at
    CHAMPION ||--o{ CHAMPION_ANALYTICS : describes
    ROLE ||--o{ CHAMPION_ANALYTICS : scopes
    PLATFORM ||--o{ CHAMPION_ANALYTICS : scopes
    PATCH ||--o{ CHAMPION_ANALYTICS : scopes
    TIER ||--o{ CHAMPION_ANALYTICS : scopes

    REGION {
        string name PK
    }
    PLATFORM {
        string code PK
        string region_name FK
    }
    SUMMONER {
        string puuid PK
        string platform_code PK
        string game_name
        string tag_line
        int profile_icon_id
    }
    MATCH {
        string match_id PK
        string platform_code PK
        string completion_status
        int queue_id FK
        string patch_version FK
    }
    SUMMONER_MATCH {
        string match_id PK
        string platform_code PK
        string puuid PK
        int champion_id FK
        string role_id FK
        int performance_rank
    }
    CHAMPION {
        int champion_id PK
        string name
    }
    CHAMPION_ANALYTICS {
        int champion_id FK
        string role_id FK
        string tier FK
        string platform_code FK
        string patch_version FK
        string reporting_patch
    }
    ROLE {
        string name PK
    }
    PATCH {
        string full_version PK
    }
    TIER {
        string tier PK
    }
    GAME_MODE {
        int queue_id PK
        string name
    }
```

`Summoner` is keyed by `(puuid, platform_code)` because the same account can
be represented on different platforms over time. `Match` is keyed by
`(match_id, platform_code)`, and `SummonerMatch` joins a player to a match while
holding participant-specific facts such as champion, role, and performance.
The analytics tables reuse the same champion, role, platform, patch, and tier
dimensions; `CHAMPION_ANALYTICS` is a diagram label for that family, not a
literal single table.

## Primary API Surface

The public endpoints are organized around the product concepts users navigate:

| Product concern | Representative endpoint families | Responsibility |
| --- | --- | --- |
| Player profiles | `/summoner/`, `/summoner-search/` | Resolve Riot IDs, build profiles, expose match history, and request freshness work |
| Champion and player discovery | `/champion-search/`, `/champions/`, `/leaderboards/champion-players/` | Search champions and find players associated with them |
| Champion analytics | `/analytics/champion-tier-list/`, `/analytics/champion-performance/`, `/analytics/champion-item-builds/`, `/analytics/champion-runes/`, `/analytics/champion-keystones/` | Serve win rates, builds, runes, matchups, scaling, and performance analysis |
| Champion context | `/analytics/champion-metadata/`, `/analytics/champion-context/`, `/analytics/champion-evolution-paths/`, `/analytics/champion-transformations/` | Supply patch, role, evolution, transformation, and product-specific context |
| Ladders and mastery | `/ladder/`, `/leaderboards/mastery/` | Serve ranked ladder snapshots and mastery leaderboards |
| Live games | `/live-game/` | Resolve and describe an active game when one exists |

The health and capability routes support runtime checks, while the SEO report
manifest supports static publication. The private dashboard API is a separate
dashboard-only runtime and is not part of the public product API surface.

## New Destiny And Rate Limiting

New Destiny is a separately packaged request-protection library, but its
integration was purpose-built around WRGG's operating pattern: a high volume
of Riot API calls produced by the API, match workers, ladder refresh, and
scheduled jobs. A per-process limiter would not coordinate those callers, so
WRGG uses a shared Redis-backed model.

For each protected request, New Destiny can:

- atomically admit or reject the request before it reaches Riot;
- model application, service, and method limits by Riot routing value;
- share temporary blocks between API workers and background workers;
- distinguish an internal preflight block from an external Riot throttle;
- respect Riot's retry timing and record the outcome for operations;
- expose low-cardinality and request-level metrics without becoming the
  application's durable data store.

```mermaid
flowchart LR
    Request[API or worker request]
    Preflight[Shared Redis preflight]
    Riot[Riot API]
    Block[Temporary Redis block]
    Outcome[Application response or retry]

    Request --> Preflight
    Preflight -->|admitted| Riot
    Preflight -->|internally blocked| Outcome
    Riot -->|success| Outcome
    Riot -->|external throttle| Block
    Block --> Outcome
```

The library models the high-level rules Riot communicates, but it cannot
observe every implementation detail of Riot's counters. A small amount of
bounded mismatch is therefore possible even with normal configuration. The
important design property is that internal and external enforcement remain
visible and that all callers share the same protective posture.

## Worker System

The API and worker images are two Docker targets built from the same backend
repository, Dockerfile, and build context. The Dockerfile's shared `base`
stage gives them the same Python runtime, operating-system packages,
`requirements.txt` dependencies, and common modules. They are not identical
images:

- The `api` target adds `main.py`, FastAPI routes, Alembic migrations, and the
  API-only files needed to serve HTTP requests.
- The `worker` target adds worker modules, asset-refresh scripts, and the small
  API utility surface those workers import.

The release workflow builds both targets from the same source release and
publishes them to separate ECR repositories. Kubernetes then selects the API
image for the FastAPI deployment and the worker image for match, ladder,
maintenance, and scheduled data jobs. The `dashboard-api` deployment is a
dashboard-only FastAPI process based on the API image; the dashboard's static
React files are served by a separate, small web image.

This keeps source and dependency versions aligned without pretending that API
availability and background throughput are the same workload.

### Match Queue Worker

The match worker claims queued match jobs, performs protected Match-V5
requests, writes match archives to S3, persists normalized match and player
data to PostgreSQL, and records terminal or retryable outcomes. It owns the
throughput-oriented path that turns user-initiated or system-generated queue
entries into durable match history.

### Ladder Worker

The ladder worker refreshes ranked snapshots for configured regions. It fetches
the relevant ranked pages, resolves the player and mastery data needed for the
ladder view, updates durable records, and publishes the snapshot consumed by
the API. It is long-running regional work rather than a browser request.

### Scheduled Data And Asset Jobs

Recurring jobs keep the rapidly changing game catalog usable:

- patch partitions and reference data;
- champions, items, runes, spells, game modes, and icons;
- CommunityDragon-backed asset reconciliation;
- tier-cohort and other derived analytics refreshes.

Initialization and recurring reconciliation use different patch-readiness
policies so a newly published Data Dragon patch cannot prevent a fresh
environment from starting while still allowing the live environment to catch
up when all sources are ready.

### Archive, SEO, And Maintenance Jobs

Archive reconciliation repairs database references against retained S3 match
objects. Archive cleanup removes expired objects according to the retention
policy. The SEO publisher reads the database-backed report manifest, renders
static pages and sitemap content, and publishes them to the public site bucket.

These jobs produce or maintain derived and operational state; they do not
replace PostgreSQL as the durable application source of truth.

## Database Lifecycle

Database schema changes are versioned with Alembic. Runtime startup does not
silently invent a schema; the initialization sequence applies migrations and
then runs idempotent seed and refresh jobs. Existing data is treated as durable
state, so destructive reset and snapshot restore are explicit infrastructure
operations rather than normal application deployment behavior.

## Failure Boundaries

- A backend image failure affects API and workers using that release, but an
  earlier immutable artifact can be selected without rebuilding.
- A worker failure delays ingestion and derived data, but should not make the
  public static shell unavailable.
- Redis loss affects coordination, rate limiting, queue state, and dashboard
  observations; it is not the durable product database.
- PostgreSQL failure affects durable product behavior and requires the explicit
  storage recovery path described in [Delivery and operations](delivery-and-operations.md).

## Source Of Truth

The backend repository is the source of truth for runtime behavior. The
published backend release is the source of truth for a deployable image. The
production `k8s-live` revision is the source of truth for which release is
requested.
