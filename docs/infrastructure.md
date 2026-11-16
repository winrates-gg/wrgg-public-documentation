# Infrastructure

## High Level Overview

WR.GG is built on hybrid cloud infrastructure, partly to demonstrate skills in
solution architecture and partly to save on infrastructure costs. Terraform
(IaC) owns AWS resources and state. Ansible owns the on-prem worker's host
lifecycle and its integration with either the production or staging
environment.

## AWS Organization Chart

```mermaid
flowchart TB
    WRGG["WR.GG"]
    Organization["WR.GG AWS Organization"]
    Management["Management account"]
    Production["Production account"]
    Staging["Staging account\nEphemeral; symmetrical to production"]
    OnPrem["On-premises data center"]
    RemoteHost["Remote Linux worker"]

    WRGG --> Organization
    WRGG --> OnPrem
    Organization --> Management
    Organization --> Production
    Organization --> Staging
    OnPrem --> RemoteHost

    classDef neutral fill:#f3f4f6,stroke:#6b7280,color:#111827
    classDef aws fill:#dbeafe,stroke:#2563eb,color:#111827
    classDef runtime fill:#dcfce7,stroke:#16a34a,color:#111827

    class WRGG,OnPrem neutral
    class Organization,Management,Production,Staging aws
    class RemoteHost runtime
```

By design, Production and Staging use the same shape in separate accounts with
different identity, credentials, domains, data, and scale preferences.

## Hybrid Cloud Shape

The ARM64 EC2 host runs the K3s control plane and the critical components required for
core application behavior. The on-premises Linux host joins the cluster as a
worker for non-critical, but high value recurring workloads.

```mermaid
flowchart LR
    subgraph AWS["AWS / Cloud"]
        ControlPlane["ARM64 EC2 K3s control plane"]
        CoreServices["API and core data services"]
        ControlPlane --> CoreServices
    end

    subgraph OnPrem["On-premises data center"]
        Worker["Remote Linux worker"]
        Background["Background processing capacity"]
        Worker --> Background
    end

    ControlPlane --- Worker

    classDef aws fill:#dbeafe,stroke:#2563eb,color:#111827
    classDef runtime fill:#dcfce7,stroke:#16a34a,color:#111827

    class ControlPlane,CoreServices aws
    class Worker,Background runtime
```

The two hosts form one logical cluster for the selected environment. The AWS
control plane carries the API and core data services. The on-premises worker
provides additional capacity for background processing and can be attached to
staging or production but not both simulatensouly.

## Hybrid Network

The public internet (end user) reaches the AWS control plane. The private worker path uses
AWS Systems Manager for registration and Ansible for host preparation and
attachment. Tailscale is the WR.GG-controlled private network spanning the
AWS control plane and the on-premises worker.

```mermaid
flowchart LR
    Internet["Public internet"]
    PublicEdge["Public web and API edge"]
    ControlPlane["ARM64 EC2 K3s control plane"]
    SSM["AWS Systems Manager\nhybrid activation"]
    Ansible["Ansible attachment path"]
    Tailscale["WR.GG Tailscale tailnet"]
    Worker["Remote Linux worker"]

    subgraph AWS["AWS / Cloud"]
        PublicEdge
        ControlPlane
        SSM
    end

    subgraph OnPrem["On-premises data center"]
        Ansible
        Worker
    end

    Internet --> PublicEdge
    PublicEdge --> ControlPlane
    SSM -.-> Ansible
    Ansible --> Worker
    Tailscale --- ControlPlane
    Tailscale --- Worker

    classDef neutral fill:#f3f4f6,stroke:#6b7280,color:#111827
    classDef aws fill:#dbeafe,stroke:#2563eb,color:#111827
    classDef runtime fill:#dcfce7,stroke:#16a34a,color:#111827

    class Internet,Ansible neutral
    class PublicEdge,ControlPlane,SSM aws
    class Tailscale,Worker runtime
```

Ansible prepares the worker, registers it through a short-lived SSM hybrid
activation, validates the Tailscale network, obtains the selected K3s join
information, and attaches it to the target environment.

## Resource Details

The detailed production resource inventory is documented in
[Infrastructure Resource Topology](infrastructure-resources.md). It covers the Terraform managed, AWS-deployed networking, compute, storage, monitoring and computing solution
