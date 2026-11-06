# Generated Infrastructure Views

## Purpose

The public architecture diagrams explain the system. A generated Terraform
diagram is useful as a technical appendix and as a drift check, but the source
state belongs to the private infrastructure repositories.

The recommended tool is **InfraMap**. It reads a Terraform working directory
or state file and produces a cleaner resource-level graph than native
`terraform graph`. Graphviz renders the resulting DOT output as SVG or PNG.

## Code View Without AWS Access

From a local Terraform root, after the required modules are available:

```sh
inframap generate . | dot -Tsvg > /tmp/wrgg-production-code.svg
```

This represents the infrastructure declared by the code. It does not prove
that the resources currently exist.

## Deployed View With Read-Only AWS Access

To represent the actual deployed production state, use a read-only AWS session
that can read the Terraform backend state:

```sh
terraform init
terraform state pull > /tmp/wrgg-production.tfstate
inframap generate /tmp/wrgg-production.tfstate \
  | dot -Tsvg > /tmp/wrgg-production-state.svg
rm /tmp/wrgg-production.tfstate
```

The exact Terraform root and AWS profile are intentionally omitted from this
public repository. In the real project, the production infrastructure is split
into database storage, core control-plane infrastructure, and web distribution
roots. Generate each root separately or assemble the resulting views during
private documentation work.

## Plan View

For a proposed change, generate a plan and inspect the dependency graph:

```sh
terraform plan -out=/tmp/wrgg.tfplan
terraform graph -plan=/tmp/wrgg.tfplan \
  | dot -Tsvg > /tmp/wrgg-production-plan.svg
rm /tmp/wrgg.tfplan
```

## Publication Rules

Never upload or commit raw Terraform state, plan files, or generated diagrams
that expose sensitive metadata. Review diagrams for:

- account IDs;
- public IP addresses;
- bucket names;
- internal hostnames;
- IAM role names;
- secret values accidentally rendered as attributes.

The public diagrams in this repository are curated abstractions. They are more
useful to a reader than a literal dump of every AWS resource and safer to
publish.

