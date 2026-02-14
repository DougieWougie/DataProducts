# Implementation Planning Prompt

Use this prompt to continue from the approved architecture design into implementation planning.

---

## Prompt

You are a senior enterprise solution architect with significant experience designing data lakes in AWS.

Read the approved architecture design at `docs/plans/2026-02-14-data-products-design.md`. This design has been reviewed and approved. Do not revisit architectural decisions — proceed to implementation planning.

### Context

We are building a Data Products Platform on an existing AWS lakehouse. The key stack decisions are:

- **Storage:** S3 + Apache Iceberg
- **Primary query engine:** Athena (serverless)
- **Additional engines:** Snowflake (external storage via central enterprise Glue mesh catalog), Starburst (Trino)
- **Catalog:** DynamoDB + AppSync (GraphQL) for data product metadata, Glue Data Catalog for table schemas, central enterprise Glue mesh catalog + Alation for enterprise discovery
- **FGAC:** Cedar (Verified Permissions) as policy-of-record, translated to Lake Formation TBAC, Snowflake RBAC/DDM, and Starburst BIAC
- **Tokenisation:** Jurisdictional PII separation across EU, India, Switzerland, US, and Monaco with per-region token vaults (DynamoDB) and HMAC keys (KMS)
- **Consumption:** AppSync (internal GraphQL), Apollo Federation on ECS Fargate (external GraphQL), Athena SQL (BI/analysts)
- **BI:** Tableau (existing estate) and Power BI (adopting), both via Athena with semantic views; jurisdictional detokenisation enforced via Athena UDF + Cedar
- **Observability:** CloudWatch + OpenTelemetry, SLO evaluation engine, health scores synced to Alation
- **Audit:** Immutable Iceberg tables with S3 Object Lock, jurisdictional residency, Merkle hash tamper evidence
- **Security:** Defence in depth — WAF, VPC endpoints, per-domain KMS keys, per-jurisdiction HMAC keys, GuardDuty, Macie, Security Hub
- **Governance:** Federated model with central standards, policy-as-code (Cedar in Git), data product maturity model (L1-L5), subscription-based access
- **IaC:** Terraform
- **API preference:** GraphQL

### Task

Create a detailed implementation plan that:

1. **Decomposes the architecture into workstreams** — identify logical groupings of work that can be developed in parallel where possible (e.g. platform foundation, catalog service, FGAC engine, hydration framework, consumption layer, observability, etc.)

2. **Sequences the workstreams** — identify dependencies between workstreams and determine the critical path. Some components must exist before others can be built (e.g. catalog before governance workflows, token vaults before hydration pipelines).

3. **Defines milestones** — group workstreams into delivery milestones that each produce a usable increment of the platform. Aim for a walking skeleton first, then layer on capability.

4. **Specifies deliverables per workstream** — for each workstream, list the Terraform modules, Lambda functions, Step Functions state machines, GraphQL schema components, Cedar policies, and configuration needed.

5. **Identifies risks and mitigations** — technical risks (e.g. LF-TBAC tag limits, Snowflake external catalog latency, cross-engine FGAC drift) and organisational risks (e.g. domain team readiness, Alation integration complexity).

6. **Defines the MVP** — what is the minimum viable platform that allows one domain team to publish and consume a single data product with FGAC, tokenisation, and observability?

Do not write implementation code at this stage. Focus on the plan structure, sequencing, and deliverable definitions. Write the plan to `docs/plans/2026-02-14-data-products-implementation-plan.md`.
