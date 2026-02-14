# Data Products Platform — Architecture Design

**Date:** 2026-02-14
**Status:** Approved
**Author:** Architecture Team

## Context & Constraints

| Dimension | Value |
|---|---|
| Storage | S3 Lakehouse |
| Table format | Apache Iceberg |
| Query engine (primary) | Athena (serverless) |
| Additional engines | Snowflake (external storage), Starburst (Trino) |
| Catalog & FGAC | Lake Formation (column/row-level perms) |
| Enterprise catalog | Central Glue Mesh Catalog + Alation |
| Org model | Medium (6-15 domains), federated governance |
| Consumers | Internal apps, analytics/BI (Tableau + Power BI), external partners |
| API preference | GraphQL |
| IaC | Terraform |
| Jurisdictions | EU, India, Switzerland, US, Monaco |

## Approach

Lake Formation-centric data mesh as the foundation, with targeted elements from a custom platform approach:

- **Lake Formation** as the governance/FGAC control plane, **AppSync** for internal GraphQL
- **Apollo Federation** for the external-facing and app-facing consumption layer
- **Cedar (Verified Permissions)** for policy-as-code governance
- **Tokenisation** for jurisdictional PII separation
- **AWS DataZone rejected** — adds an abstraction layer that conflicts with custom contracts and flexible GraphQL

---

## 1. Data Product Specification & Contract Model

### Data Product Specification

A data product is a versioned, discoverable, governed unit of data with a defined owner, schema contract, SLO, and access policy. Each data product is represented as a specification document stored in the catalog:

```
data-product-spec/
├── manifest.yaml          # Identity, ownership, classification
├── schema/
│   ├── contract.avro      # Output port schema (Avro for Iceberg compat)
│   └── evolution-policy   # BACKWARD, FORWARD, FULL, NONE
├── quality/
│   ├── slo.yaml           # Freshness, completeness, accuracy targets
│   └── expectations.yaml  # Great Expectations / Deequ rules
├── access/
│   ├── policy.cedar       # Cedar access policy
│   └── classification.yaml # PII/PHI/confidential field tags
└── lineage/
    └── upstream.yaml      # Declared dependencies on other data products
```

### Manifest Structure

```yaml
apiVersion: dataproduct/v1
kind: DataProduct
metadata:
  name: customer-360
  domain: customer-intelligence
  owner:
    team: ci-data-engineering
    contact: ci-data-eng@company.com
    escalation: ci-platform-lead@company.com
  tier: gold                    # gold/silver/bronze SLO tiers
  classification: confidential  # public/internal/confidential/restricted
spec:
  description: "Unified customer view combining CRM, transactions, and engagement"
  outputPorts:
    - name: customer_profile
      format: iceberg
      location: s3://lakehouse-curated/customer-intelligence/customer-360/
      catalog: glue:customer_intelligence.customer_profile
      schema: schema/contract.avro
      freshness: PT1H            # ISO 8601 duration — 1 hour
    - name: customer_api
      format: graphql
      endpoint: /data-products/customer-360/graphql
  inputPorts:
    - dataProduct: crm/contacts
    - dataProduct: transactions/orders
    - source: kinesis:clickstream-events
  slo:
    availability: 99.9%
    freshness: PT1H
    completeness: 99.5%
```

### Ownership Model

```mermaid
graph TD
    subgraph "Enterprise Governance"
        CDO[Chief Data Officer]
        DG[Data Governance Council]
    end

    subgraph "Platform Team"
        PT[Data Platform Team]
        PT -->|maintains| INFRA[Shared Infrastructure]
        PT -->|operates| CATALOG[Data Product Catalog]
        PT -->|manages| FGAC[FGAC Engine]
    end

    subgraph "Domain Team (e.g. Customer Intelligence)"
        DO[Domain Data Owner]
        DE[Domain Data Engineers]
        DS[Domain Data Steward]

        DO -->|accountable for| DP[Data Product]
        DE -->|build & maintain| DP
        DS -->|governs quality & classification| DP
    end

    CDO -->|sets policy| DG
    DG -->|defines standards| PT
    DG -->|appoints| DO
    DO -->|reports SLO compliance to| DG
    DP -->|registered in| CATALOG
    DP -->|permissions via| FGAC
```

### Schema Contract & Evolution

| Policy | Meaning | When to Use |
|---|---|---|
| **BACKWARD** | New schema can read old data | Default for most products |
| **FORWARD** | Old schema can read new data | When consumers can't update quickly |
| **FULL** | Both backward and forward | Critical shared products (gold tier) |
| **NONE** | Breaking changes allowed | Dev/experimental products only |

Schema contracts are enforced at registration time — the catalog service validates the proposed schema against the evolution policy before a new version is published.

---

## 2. High-Level Architecture

```mermaid
graph TB
    subgraph "Domain Teams"
        D1[Domain: Customer Intelligence]
        D2[Domain: Transactions]
        D3[Domain: Risk]
        DN[Domain: ...]
    end

    subgraph "Data Product Platform"
        subgraph "Control Plane"
            CAT[Data Product Catalog<br/>DynamoDB + AppSync]
            REG[Schema Registry<br/>Glue Schema Registry]
            GOV[Governance Engine<br/>Step Functions + Cedar]
            OBS[Observability<br/>CloudWatch + OTel]
        end

        subgraph "Data Plane"
            subgraph "Storage Layer"
                S3C[S3 Curated Zone<br/>Iceberg Tables]
                S3R[S3 Raw Zone]
                S3S[S3 Staging Zone]
            end

            subgraph "Catalog & FGAC"
                LF[Lake Formation<br/>Tag-Based Access Control]
                GC[Glue Data Catalog]
            end

            subgraph "Compute"
                ATH[Athena<br/>Ad-hoc & BI queries]
                EMR[EMR Serverless<br/>Heavy transforms]
                LAM[Lambda<br/>Lightweight transforms]
            end
        end

        subgraph "Consumption Layer"
            subgraph "Internal"
                AS[AppSync<br/>Internal GraphQL]
                ADS[Athena Direct<br/>SQL access]
            end
            subgraph "External"
                APIGW[API Gateway]
                AF[Apollo Federation<br/>on ECS Fargate]
            end
        end

        subgraph "Hydration Layer"
            KIN[Kinesis Data Streams<br/>Real-time ingest]
            GLJ[Glue ETL Jobs<br/>Batch transforms]
            SF[Step Functions<br/>Orchestration]
        end
    end

    subgraph "Consumers"
        BI[BI Tools<br/>QuickSight / Tableau]
        APP[Internal Applications]
        EXT[External Partners]
        ML[ML Pipelines]
    end

    D1 & D2 & D3 & DN -->|publish specs| CAT
    D1 & D2 & D3 & DN -->|hydrate| KIN & GLJ

    KIN & GLJ -->|write| S3C
    S3C -->|registered in| GC
    GC -->|governed by| LF

    CAT -->|triggers| GOV
    GOV -->|applies policies| LF
    OBS -->|monitors| S3C & ATH & AF

    ATH -->|reads| S3C
    AS -->|queries| ATH
    AF -->|queries| ATH

    BI -->|connects via| ADS
    APP -->|calls| AS
    EXT -->|calls| APIGW --> AF
    ML -->|reads| ADS
```

### Layer Responsibilities

| Layer | Purpose | Key Services |
|---|---|---|
| **Control Plane** | Manage data product lifecycle, governance, observability | DynamoDB, AppSync, Step Functions, Cedar, CloudWatch |
| **Data Plane** | Store, catalog, and secure the actual data | S3, Iceberg, Glue Catalog, Lake Formation |
| **Hydration Layer** | Ingest and transform data into data products | Kinesis, Glue ETL, Step Functions |
| **Consumption Layer** | Expose data products to consumers via appropriate interfaces | AppSync (internal GraphQL), Apollo Federation (external GraphQL), Athena (SQL) |

### Data Product Lifecycle Flow

```mermaid
sequenceDiagram
    participant DT as Domain Team
    participant CAT as Catalog (AppSync)
    participant GOV as Governance Engine
    participant REG as Schema Registry
    participant LF as Lake Formation
    participant S3 as S3 / Iceberg
    participant OBS as Observability

    DT->>CAT: Register data product spec
    CAT->>REG: Validate schema contract
    REG-->>CAT: Schema valid / rejected
    CAT->>GOV: Trigger governance workflow
    GOV->>GOV: Validate classification, ownership, SLO
    GOV->>LF: Apply LF-TBAC tags from Cedar policy
    LF-->>GOV: Permissions applied
    GOV-->>CAT: Approved / rejected
    CAT-->>DT: Registration confirmed

    Note over DT,S3: Hydration Phase
    DT->>S3: Write Iceberg data (batch or stream)
    S3-->>OBS: Table metrics emitted

    Note over OBS: Continuous Monitoring
    OBS->>OBS: Evaluate SLO (freshness, completeness)
    OBS-->>CAT: Update product health status
    OBS-->>DT: Alert on SLO breach
```

### Network Topology

```mermaid
graph LR
    subgraph "VPC - Data Platform"
        subgraph "Private Subnets"
            ECS[Apollo Federation<br/>ECS Fargate]
            ATH[Athena Endpoints]
            DDB[DynamoDB Endpoints]
        end
        subgraph "Public Subnets"
            APIGW[API Gateway<br/>Regional Endpoint]
            WAF[AWS WAF]
        end
    end

    subgraph "Shared Services VPC"
        IDP[Corporate IdP<br/>Okta / Azure AD]
        COG[Cognito User Pool]
    end

    subgraph "Domain Account A"
        LFA[Lake Formation<br/>Data Sharing]
        S3A[S3 Data]
    end

    EXT[External Partners] -->|HTTPS + API Key + OAuth| WAF --> APIGW --> ECS
    INT[Internal Apps] -->|IAM + AppSync| AS[AppSync]
    IDP -->|SAML/OIDC federation| COG
    COG -->|JWT tokens| APIGW
    LFA -->|cross-account share| ATH
```

### Cross-Account Model

Each domain gets its own AWS account (or shares accounts within a domain boundary). Data products are shared via Lake Formation cross-account grants, with the platform account acting as the central catalog and governance hub.

```mermaid
graph TB
    subgraph "Platform Account (Hub)"
        CAT[Catalog + Governance]
        LFC[Lake Formation Central]
    end

    subgraph "Domain Account: Customer Intelligence"
        S3A[S3: customer data]
        LFA[LF: local permissions]
        GCA[Glue Catalog: local]
    end

    subgraph "Domain Account: Transactions"
        S3B[S3: transaction data]
        LFB[LF: local permissions]
        GCB[Glue Catalog: local]
    end

    subgraph "Consumer Account: Analytics"
        ATH[Athena workgroups]
        BI[BI Tools]
    end

    LFA -->|resource share| LFC
    LFB -->|resource share| LFC
    LFC -->|grant access| ATH
    CAT -->|governs| LFC
```

---

## 3. Data Product Catalog & Metadata

### Architecture Choice

| Option Considered | Verdict |
|---|---|
| **DynamoDB + AppSync** (chosen) | Serverless, scales to zero, native GraphQL, single-table design handles hierarchical product specs well, pay-per-request fits medium scale |
| **Aurora PostgreSQL + Apollo** | Relational model is natural for metadata, but introduces cluster management overhead that isn't justified at 6-15 domains |
| **AWS DataZone** | Rejected — too opinionated, poor GraphQL fit |

### DynamoDB Single-Table Design

All catalog entities live in one table with a composite key design:

| Entity | PK | SK | Attributes |
|---|---|---|---|
| Data Product | `DP#customer-360` | `META` | name, domain, owner, tier, classification, status, created, updated |
| Version | `DP#customer-360` | `VER#2024-03-15T10:00:00Z` | version, schema_ref, changelog, status |
| Output Port | `DP#customer-360` | `PORT#customer_profile` | format, location, catalog_ref, freshness_slo |
| Input Port | `DP#customer-360` | `INPUT#crm/contacts` | source_product, source_type |
| SLO | `DP#customer-360` | `SLO#availability` | target, current, breach_count, last_evaluated |
| Access Policy | `DP#customer-360` | `POLICY#default` | cedar_policy_ref, lf_tags, classification |
| Domain | `DOMAIN#customer-intelligence` | `META` | name, owner_team, contact, products_count |
| Subscription | `SUB#analytics-team` | `DP#customer-360` | subscriber, status, granted_ports, approved_by, expires |

**GSI-1 (Domain index):** `GSI1PK = DOMAIN#<domain>`, `GSI1SK = DP#<product>` — list all products in a domain.

**GSI-2 (Status index):** `GSI2PK = STATUS#<status>`, `GSI2SK = DP#<product>` — find all products by lifecycle status.

**GSI-3 (Subscriber index):** `GSI3PK = SUB#<subscriber>`, `GSI3SK = DP#<product>` — list all subscriptions for a consumer.

### AppSync GraphQL Schema (Key Types)

```graphql
type DataProduct {
  id: ID!
  name: String!
  domain: Domain!
  owner: OwnerInfo!
  tier: Tier!
  classification: Classification!
  status: ProductStatus!
  currentVersion: ProductVersion!
  versions: [ProductVersion!]!
  outputPorts: [OutputPort!]!
  inputPorts: [InputPort!]!
  slos: [SLO!]!
  accessPolicy: AccessPolicy!
  subscriptions: [Subscription!]!
  health: HealthStatus!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}

type OutputPort {
  name: String!
  format: PortFormat!
  location: String!
  catalogRef: String!
  schema: SchemaContract!
  freshnessSLO: String
}

type Subscription {
  subscriber: String!
  dataProduct: DataProduct!
  grantedPorts: [String!]!
  status: SubscriptionStatus!
  approvedBy: String
  expiresAt: AWSDateTime
}

type Query {
  getDataProduct(id: ID!): DataProduct
  listDataProducts(domain: String, tier: Tier, status: ProductStatus): DataProductConnection!
  searchDataProducts(query: String!, filters: SearchFilters): DataProductConnection!
  getSubscription(subscriberId: String!, productId: ID!): Subscription
  listSubscriptions(subscriberId: String): [Subscription!]!
}

type Mutation {
  registerDataProduct(input: RegisterProductInput!): DataProduct!
  publishVersion(productId: ID!, input: PublishVersionInput!): ProductVersion!
  requestSubscription(input: SubscriptionRequestInput!): Subscription!
  approveSubscription(subscriptionId: ID!): Subscription!
  revokeSubscription(subscriptionId: ID!): Subscription!
  updateSLO(productId: ID!, input: UpdateSLOInput!): SLO!
  deprecateDataProduct(productId: ID!, sunset: AWSDateTime!): DataProduct!
}

type Subscription @aws_subscribe(mutations: ["publishVersion", "updateSLO"]) {
  onProductUpdated(domain: String): DataProduct
  onSLOBreach(tier: Tier): SLO
}

enum ProductStatus { DRAFT PUBLISHED DEPRECATED SUNSET }
enum Tier { GOLD SILVER BRONZE }
enum Classification { PUBLIC INTERNAL CONFIDENTIAL RESTRICTED }
enum PortFormat { ICEBERG GRAPHQL }
enum SubscriptionStatus { PENDING APPROVED REVOKED EXPIRED }
```

### Subscription Model

Data access is subscription-based. Consumers request subscriptions to specific output ports, which triggers a governance workflow:

| Product Tier | Classification | Approval |
|---|---|---|
| Bronze | Public / Internal | Auto-approved |
| Silver | Public / Internal | Auto-approved |
| Silver | Confidential | Domain owner approval |
| Gold | Any | Domain owner + governance council approval |
| Any | Restricted | Governance council + CDO approval |

Approved subscriptions result in Lake Formation grants being applied to the subscriber's IAM role. Subscriptions have optional expiry dates and can be revoked.

---

## 4. Fine-Grained Access Control (FGAC)

### FGAC Strategy: Lake Formation TBAC + Cedar Policies

| System | Role | Scope |
|---|---|---|
| **Cedar policies** | Express business intent | Policy authoring, version control, audit |
| **Lake Formation TBAC** | Enforce at query time — column/row/cell filtering on Iceberg tables | Runtime enforcement on Athena, EMR, Redshift Spectrum |
| **AppSync/Apollo resolvers** | Enforce on GraphQL — field-level authorization based on caller identity | Runtime enforcement on API layer |

Cedar is the **policy-of-record**. LF tags and GraphQL resolver rules are **derived** from Cedar policies by the governance engine.

### Tag Taxonomy

| Tag Key | Example Values | Applied To | Purpose |
|---|---|---|---|
| `domain` | `customer-intelligence`, `transactions`, `risk` | Database | Domain ownership boundary |
| `tier` | `gold`, `silver`, `bronze` | Table | SLO tier / governance strictness |
| `classification` | `public`, `internal`, `confidential`, `restricted` | Table, Column | Data sensitivity |
| `pii_type` | `email`, `phone`, `ssn`, `address`, `dob` | Column | PII sub-classification |
| `product` | `customer-360`, `order-history` | Table | Data product identity |
| `environment` | `prod`, `staging`, `dev` | Database | Environment isolation |
| `jurisdiction` | `EU`, `IN`, `CH`, `US`, `MC` | Table, Column | Data residency and regulatory scope |
| `residency_region` | `eu-west-1`, `ap-south-1`, `eu-central-2`, `us-east-1`, `eu-west-3` | Table | AWS region where PII must stay |
| `tokenised` | `true`, `false` | Column | Whether the column contains tokens vs raw PII |
| `erasure_status` | `active`, `erased`, `pending_erasure` | Row-level | GDPR/DPDPA erasure tracking |
| `consent_basis` | `consent`, `contract`, `legitimate_interest`, `legal_obligation` | Table | Lawful basis for processing |
| `cross_border_approved` | `true`, `false` | Table | Whether cross-border transfer has been approved |

### Cedar Policy Model

```cedar
// Gold-tier products require explicit subscription
permit(
  principal in Group::"subscribed-consumers",
  action in [Action::"read", Action::"query"],
  resource
) when {
  resource has tier && resource.tier == "gold" &&
  principal has subscription &&
  principal.subscription.contains(resource.product)
};

// PII columns restricted to PII-trained users
forbid(
  principal,
  action in [Action::"read"],
  resource
) when {
  resource has classification && resource.classification == "confidential" &&
  resource has pii_type &&
  !(principal has pii_certified && principal.pii_certified == true)
};

// External partners only see public columns
forbid(
  principal in Group::"external-partners",
  action in [Action::"read"],
  resource
) when {
  resource has classification && resource.classification != "public"
};
```

### FGAC Across Consumption Paths

```mermaid
graph TB
    subgraph "Policy Source of Truth"
        CEDAR[Cedar Policies<br/>in Git + Verified Permissions]
    end

    subgraph "SQL Path (Athena)"
        LF[Lake Formation TBAC]
        ATH[Athena Query]
        ICE[Iceberg Table]

        ATH -->|query| ICE
        LF -->|column filter<br/>row filter<br/>cell mask| ATH
    end

    subgraph "GraphQL Path (Internal - AppSync)"
        COG[Cognito JWT]
        ASR[AppSync Resolvers]
        ASR -->|field-level auth| RESP1[Filtered Response]
        COG -->|claims| ASR
    end

    subgraph "GraphQL Path (External - Apollo)"
        APIKEY[API Key + OAuth]
        AFR[Apollo Resolvers]
        VP[Verified Permissions<br/>Cedar Runtime]
        AFR -->|isAuthorized?| VP
        VP -->|permit/deny| AFR
        AFR -->|field-level filter| RESP2[Filtered Response]
        APIKEY -->|identity| AFR
    end

    subgraph "Direct S3 Path"
        S3AP[S3 Access Points<br/>per consumer]
        IAM[IAM Policy<br/>prefix-scoped]
    end

    CEDAR -->|translated to tags| LF
    CEDAR -->|resolver rules| ASR
    CEDAR -->|runtime evaluation| VP
    CEDAR -->|prefix policies| IAM
```

### Column-Level Security Example

| Column | Classification | PII Type | Public | Internal User | PII-Certified User | External Partner |
|---|---|---|---|---|---|---|
| `customer_id` | internal | - | hidden | visible | visible | hidden |
| `name` | confidential | name | hidden | hidden | visible | hidden |
| `email` | confidential | email | hidden | hidden | visible | hidden |
| `segment` | internal | - | hidden | visible | visible | visible |
| `lifetime_value` | internal | - | hidden | visible | visible | hidden |
| `phone` | restricted | phone | hidden | hidden | visible (with approval) | hidden |
| `region` | public | - | visible | visible | visible | visible |

### Row-Level Security

| Scenario | LF Row Filter Expression |
|---|---|
| Regional data residency | `region = 'eu-west-1'` |
| Domain scoping | `domain = 'customer-intelligence'` |
| Partner data isolation | `partner_id = '{caller.partner_id}'` |
| Time-boxed access | `event_date >= '2024-01-01'` |

---

## 4a. Jurisdictional Complexity & Tokenisation

### Jurisdictional Landscape

| Jurisdiction | Regulation | Data Residency Requirement | Cross-Border Transfer | Key Constraint |
|---|---|---|---|---|
| **EU** | GDPR | No hard residency mandate, but practical preference for EU processing | Adequacy decision, SCCs, or BCRs required | Right to erasure, data minimisation, lawful basis |
| **India** | DPDPA 2023 | Critical personal data must stay in India; general PD can transfer with conditions | Government-notified whitelist of countries, or consent + contract | Consent-based processing, data fiduciary obligations |
| **Switzerland** | nFADP (revDSG) | No hard mandate, but Swiss adequacy is separate from EU | Own adequacy list (overlaps with EU), SCCs recognised | Independent from GDPR — separate compliance track |
| **US** | CCPA/CPRA + state patchwork | No federal residency requirement | No transfer restrictions (but sector-specific: HIPAA, GLBA) | Right to opt-out of sale, right to delete, sensitive PI categories |
| **Monaco** | Law No. 1.565 | Processing should occur within Monaco or adequate countries | CCIN (Monaco's DPA) approval for non-adequate transfers | Not an EU member — separate adequacy assessments needed |

### Data Residency Architecture

Core principle: **PII stays in-jurisdiction; tokenised analytical data flows freely.**

```mermaid
graph TB
    subgraph "Regional Landing Zones"
        subgraph "eu-west-1 (EU)"
            S3_EU[S3: PII Vault EU<br/>EU citizen PII]
            TV_EU[Token Vault EU<br/>DynamoDB]
        end
        subgraph "ap-south-1 (India)"
            S3_IN[S3: PII Vault India<br/>Indian critical PD]
            TV_IN[Token Vault India<br/>DynamoDB]
        end
        subgraph "eu-central-2 (Switzerland)"
            S3_CH[S3: PII Vault CH<br/>Swiss personal data]
            TV_CH[Token Vault CH<br/>DynamoDB]
        end
        subgraph "us-east-1 (US)"
            S3_US[S3: PII Vault US<br/>US consumer data]
            TV_US[Token Vault US<br/>DynamoDB]
        end
        subgraph "eu-west-3 (Monaco)"
            S3_MC[S3: PII Vault Monaco<br/>Monaco personal data]
            TV_MC[Token Vault Monaco<br/>DynamoDB]
        end
    end

    subgraph "Central Analytical Zone (eu-west-1)"
        LAKE[Lakehouse<br/>Tokenised Iceberg Tables]
        CAT[Data Product Catalog]
        GOV[Governance Engine]
    end

    TV_EU & TV_IN & TV_CH & TV_US & TV_MC -->|tokens only<br/>no PII| LAKE
    GOV -->|jurisdictional policies| TV_EU & TV_IN & TV_CH & TV_US & TV_MC
```

### Token Design

| Aspect | Design Decision |
|---|---|
| **Token format** | Format-preserving: `TOK_<jurisdiction>_<type>_<hash>` e.g. `TOK_EU_EMAIL_a8f3b2c1` |
| **Token scope** | One token per (entity, field, jurisdiction) — same person's email gets different tokens in EU vs India vaults |
| **Token generation** | HMAC-SHA256 with jurisdiction-specific rotating keys stored in KMS |
| **Reversibility** | Only via Token Vault lookup — requires explicit `detokenise` permission |
| **Deterministic** | Yes, within a jurisdiction — same input produces same token, enabling joins on tokenised data |
| **Cross-jurisdiction join** | Via a **global entity ID** (non-PII synthetic key) that links tokens across vaults |

### Tokenisation Flow

```mermaid
sequenceDiagram
    participant SRC as Source System
    participant HYD as Hydration Pipeline<br/>(Glue ETL)
    participant TV as Token Vault<br/>(Regional DynamoDB)
    participant KMS as AWS KMS<br/>(Regional Key)
    participant S3P as PII Vault<br/>(Regional S3)
    participant S3A as Analytical Lake<br/>(Central S3, Iceberg)

    SRC->>HYD: Raw record with PII
    HYD->>HYD: Detect jurisdiction<br/>(from country field, IP, explicit flag)
    HYD->>KMS: Get jurisdiction-specific HMAC key
    KMS-->>HYD: Key reference

    par Tokenise and Split
        HYD->>TV: Store PII → token mapping<br/>(email → TOK_EU_EMAIL_a8f3b2c1)
        HYD->>S3P: Write raw PII record<br/>(encrypted, jurisdiction-local)
        HYD->>S3A: Write tokenised record<br/>(PII fields replaced with tokens)
    end

    TV-->>HYD: Token confirmed
    S3P-->>HYD: PII stored
    S3A-->>HYD: Analytical record written
```

### Detokenisation (Controlled Re-identification)

Detokenisation is a privileged operation, never available by default:

```mermaid
sequenceDiagram
    participant USR as Authorised User
    participant GQL as GraphQL API
    participant VP as Verified Permissions<br/>(Cedar)
    participant TV as Token Vault<br/>(Regional)
    participant AUDIT as Audit Log

    USR->>GQL: query { customer(id: "X") { email } }
    GQL->>VP: isAuthorized?<br/>(user, detokenise, email, jurisdiction)

    alt Permitted
        VP-->>GQL: ALLOW
        GQL->>TV: Detokenise TOK_EU_EMAIL_a8f3b2c1
        TV-->>GQL: actual@email.com
        GQL->>AUDIT: Log detokenisation event<br/>(who, what, when, why, jurisdiction)
        GQL-->>USR: { email: "actual@email.com" }
    else Denied
        VP-->>GQL: DENY
        GQL->>AUDIT: Log denied detokenisation attempt
        GQL-->>USR: { email: "TOK_EU_EMAIL_a8f3b2c1" }
    end
```

### Jurisdictional Cedar Policies

```cedar
// Indian critical personal data never leaves ap-south-1
forbid(
  principal,
  action in [Action::"detokenise"],
  resource
) when {
  resource has jurisdiction && resource.jurisdiction == "IN" &&
  resource has pii_type && resource.pii_type in ["aadhaar", "pan", "biometric"] &&
  context has caller_region && context.caller_region != "ap-south-1"
};

// EU right to erasure — detokenisation forbidden for erased entities
forbid(
  principal,
  action in [Action::"read", Action::"detokenise"],
  resource
) when {
  resource has erasure_status && resource.erasure_status == "erased"
};

// Monaco data requires CCIN-approved basis
forbid(
  principal,
  action in [Action::"detokenise"],
  resource
) when {
  resource has jurisdiction && resource.jurisdiction == "MC" &&
  !(principal has monaco_approved_basis && principal.monaco_approved_basis == true)
};

// Swiss data — separate from EU, requires Swiss-specific authorisation
forbid(
  principal,
  action in [Action::"detokenise"],
  resource
) when {
  resource has jurisdiction && resource.jurisdiction == "CH" &&
  !(principal has swiss_data_access && principal.swiss_data_access == true)
};

// US CCPA — honour opt-out flags
forbid(
  principal,
  action in [Action::"detokenise", Action::"share"],
  resource
) when {
  resource has jurisdiction && resource.jurisdiction == "US" &&
  resource has ccpa_opt_out && resource.ccpa_opt_out == true &&
  principal in Group::"third-party-processors"
};
```

### Right to Erasure

```mermaid
sequenceDiagram
    participant DSR as Data Subject Request
    participant API as Erasure API
    participant GOV as Governance Engine
    participant TV as Token Vault<br/>(Regional)
    participant S3P as PII Vault<br/>(Regional)
    participant S3A as Analytical Lake
    participant AUDIT as Audit Log

    DSR->>API: Erasure request (subject ID, jurisdiction)
    API->>GOV: Validate request<br/>(jurisdiction, identity verification)

    GOV->>TV: Delete token mappings<br/>for subject in jurisdiction
    TV-->>GOV: Mappings deleted

    GOV->>S3P: Delete raw PII records<br/>for subject
    S3P-->>GOV: PII purged

    GOV->>S3A: Mark entity tokens as erased<br/>(Iceberg metadata update)
    Note over S3A: Tokens remain in analytical data<br/>but are now irreversible orphans —<br/>no vault entry to resolve them

    GOV->>AUDIT: Log erasure completion<br/>(subject_hash, jurisdiction, timestamp)
    GOV-->>API: Erasure confirmed
    API-->>DSR: Acknowledgement within regulatory timeframe
```

Deleting the vault mapping makes the tokens permanently unresolvable — this constitutes **cryptographic erasure**.

---

## 5. Hydration Layer

### Hydration Patterns

| Pattern | Use Case | Services | Latency |
|---|---|---|---|
| **Batch** | Historical loads, daily/hourly refreshes, large aggregations | Glue ETL → Iceberg | Minutes to hours |
| **Micro-batch** | Near real-time with exactly-once semantics | Kinesis → Glue Streaming → Iceberg | 1-5 minutes |
| **Streaming** | Event-driven, real-time metrics, CDC | Kinesis → Lambda/Flink → Iceberg | Seconds |
| **Federation** | Query-time hydration from external sources, no materialisation | Athena Federated Query | On-demand |

### Hydration Architecture

```mermaid
graph TB
    subgraph "Sources"
        RDS[(RDS / Aurora<br/>Operational DBs)]
        API_SRC[External APIs]
        S3_RAW[S3 Raw Zone<br/>File drops]
        KAF[Kafka / MSK<br/>Event streams]
        CDC[DMS CDC<br/>Change capture]
    end

    subgraph "Ingestion"
        KDS[Kinesis Data Streams]
        DMS[DMS Replication Tasks]
        S3E[S3 Event Notifications]
    end

    subgraph "Orchestration"
        SF[Step Functions<br/>Pipeline Orchestrator]
    end

    subgraph "Processing"
        subgraph "Jurisdiction Router"
            JR[Jurisdiction Detection<br/>Lambda]
        end

        subgraph "Tokenisation"
            TOK[Tokenisation Service<br/>Lambda / Glue]
            TV_R[Token Vault<br/>Regional DynamoDB]
            PII_S[PII Vault<br/>Regional S3]
        end

        subgraph "Transform"
            GLUE[Glue ETL Jobs<br/>Spark on Iceberg]
            GLUE_S[Glue Streaming<br/>Micro-batch]
            FLINK[Managed Flink<br/>Real-time]
        end

        subgraph "Quality Gate"
            DQ[Data Quality<br/>Glue DQ / Deequ]
        end
    end

    subgraph "Destination"
        ICE[Iceberg Tables<br/>S3 Curated Zone]
        GC[Glue Catalog<br/>Table registration]
        LF[Lake Formation<br/>Tag application]
    end

    RDS -->|CDC| DMS --> KDS
    API_SRC -->|pull| SF
    S3_RAW -->|event| S3E --> SF
    KAF -->|bridge| KDS
    CDC --> KDS

    SF -->|orchestrates| JR
    JR -->|route by jurisdiction| TOK
    TOK -->|PII| PII_S
    TOK -->|tokens| TV_R
    TOK -->|tokenised records| GLUE & GLUE_S

    KDS -->|streaming| FLINK
    FLINK -->|tokenised| ICE

    GLUE & GLUE_S -->|transform| DQ
    DQ -->|pass| ICE
    DQ -->|fail| SF
    ICE --> GC --> LF
```

### Pipeline Orchestration

```mermaid
stateDiagram-v2
    [*] --> DetectSource
    DetectSource --> RouteByJurisdiction

    RouteByJurisdiction --> TokeniseEU: EU data
    RouteByJurisdiction --> TokeniseIN: India data
    RouteByJurisdiction --> TokeniseCH: Swiss data
    RouteByJurisdiction --> TokeniseUS: US data
    RouteByJurisdiction --> TokeniseMC: Monaco data

    TokeniseEU --> MergeTokenised
    TokeniseIN --> MergeTokenised
    TokeniseCH --> MergeTokenised
    TokeniseUS --> MergeTokenised
    TokeniseMC --> MergeTokenised

    MergeTokenised --> Transform
    Transform --> QualityGate

    QualityGate --> WriteIceberg: Pass
    QualityGate --> Quarantine: Fail

    Quarantine --> AlertOwner
    AlertOwner --> [*]

    WriteIceberg --> ApplyLFTags
    ApplyLFTags --> UpdateCatalog
    UpdateCatalog --> EmitMetrics
    EmitMetrics --> [*]
```

### Jurisdiction Detection

| Priority | Method | Example |
|---|---|---|
| 1 | Explicit field | `jurisdiction: "EU"` in the record |
| 2 | Country code mapping | `country: "DE"` → EU, `country: "IN"` → India |
| 3 | Phone prefix | `+41` → Switzerland, `+377` → Monaco |
| 4 | Source system tagging | DMS source tagged as `jurisdiction=IN` |
| 5 | Default | Configured per pipeline, fail-safe to most restrictive |

Records with ambiguous jurisdiction are routed to a quarantine queue for manual classification.

### Data Quality Gate

```yaml
expectations:
  - name: customer_id_not_null
    type: column_not_null
    column: customer_id
    severity: critical

  - name: email_format_valid
    type: column_regex
    column: email_token
    pattern: "^TOK_[A-Z]{2}_EMAIL_[a-f0-9]{8}$"
    severity: critical

  - name: record_count_drift
    type: table_row_count
    min_delta_pct: -10
    max_delta_pct: 200
    severity: warning

  - name: freshness_check
    type: column_max_age
    column: updated_at
    max_age: PT2H
    severity: warning

  - name: jurisdiction_coverage
    type: column_distinct_values
    column: jurisdiction
    expected: ["EU", "IN", "CH", "US", "MC"]
    severity: warning
```

| Severity | Behaviour |
|---|---|
| **Critical** | Pipeline halts, data quarantined, owner alerted, catalog health → `FAILING` |
| **Warning** | Data written, alert emitted, catalog health → `DEGRADED` |
| **Info** | Metric recorded, no action |

---

## 5a. Catalog Synchronisation & Alation Integration

### Catalog Topology

| Tier | System | Role | Scope |
|---|---|---|---|
| **Domain Catalog** | Glue Data Catalog (per domain account) | Source of truth for domain Iceberg tables | Single domain |
| **Data Product Catalog** | DynamoDB + AppSync | Data product metadata, contracts, SLOs, subscriptions | Platform-wide |
| **Enterprise Mesh Catalog** | Central Glue Catalog + Alation | Cross-enterprise discovery, business glossary, lineage | Enterprise-wide |

Metadata flows upward (domain → platform → enterprise), governance flows downward (enterprise → platform → domain).

```mermaid
graph TB
    subgraph "Enterprise Tier"
        AL[Alation<br/>Discovery, Glossary,<br/>Stewardship, Lineage]
        CGC[Central Glue Mesh Catalog<br/>Enterprise-wide schema registry]
        AL <-->|bi-directional sync| CGC
    end

    subgraph "Platform Tier"
        DPC[Data Product Catalog<br/>DynamoDB + AppSync]
        DPC -->|publish product metadata| CGC
        CGC -->|enterprise glossary terms,<br/>governance tags| DPC
    end

    subgraph "Domain Tier"
        subgraph "Customer Intelligence Account"
            GC_CI[Glue Catalog: CI]
        end
        subgraph "Transactions Account"
            GC_TX[Glue Catalog: TX]
        end
        subgraph "Risk Account"
            GC_RK[Glue Catalog: Risk]
        end
    end

    GC_CI & GC_TX & GC_RK -->|Lake Formation<br/>resource shares| CGC
    DPC -->|product registration<br/>triggers sync| GC_CI & GC_TX & GC_RK
```

### What Syncs Where

| Metadata Type | Source of Truth | Synced To | Mechanism |
|---|---|---|---|
| **Table schemas & partitions** | Domain Glue Catalog | Central Glue Catalog | LF resource links (automatic) |
| **Product specs, SLOs, contracts** | Data Product Catalog (DynamoDB) | Alation (custom fields) | EventBridge → Lambda → Alation API |
| **Business glossary terms** | Alation | Data Product Catalog, Central Glue (tags) | Alation API → Lambda → DynamoDB + Glue tags |
| **Data classification tags** | Data Product Catalog (Cedar) | Central Glue (LF tags), Alation (custom fields) | Governance engine push |
| **Lineage** | Glue ETL job metadata + product input/output ports | Alation (lineage graph) | Alation Glue connector + custom lineage API |
| **Quality metrics** | Observability layer (CloudWatch) | Alation (quality badges) | Lambda → Alation API |
| **Ownership & stewardship** | Data Product Catalog | Alation (stewardship assignments) | Bi-directional sync via API |

### Alation Custom Fields for Data Products

| Alation Custom Field | Maps From | Type |
|---|---|---|
| `Data Product ID` | Catalog: product ID | String |
| `Domain` | Catalog: domain | Picker |
| `Product Tier` | Catalog: tier | Picker (Gold/Silver/Bronze) |
| `Data Classification` | Catalog: classification | Picker |
| `Freshness SLO` | Catalog: SLO spec | String (ISO 8601) |
| `Current Freshness` | Observability: metric | Rich text (with badge) |
| `Availability SLO` | Catalog: SLO spec | String (percentage) |
| `Current Availability` | Observability: metric | Rich text (with badge) |
| `Schema Contract` | Catalog: evolution policy | Picker |
| `Jurisdictions` | Catalog: jurisdiction tags | Multi-picker |
| `Subscription Status` | Catalog: subscription model | Picker |
| `Owner Team` | Catalog: owner | Object reference |
| `GraphQL Endpoint` | Catalog: output port | URL |

### Conflict Resolution

| Conflict Type | Resolution Rule |
|---|---|
| Schema difference between domain Glue and central Glue | Domain Glue wins — resource links are authoritative |
| Ownership changed in Alation vs Data Product Catalog | Alation wins — stewardship is Alation's domain |
| Business glossary term applied in Alation but missing from Glue tags | Alation term synced to Glue tags via governance engine |
| Classification changed in Cedar policy vs Alation | Cedar policy wins — classification is a security control |
| SLO metrics differ between observability and Alation | Observability wins — Alation is a consumer of metrics |

---

## 6. Consumption Layer

### Consumption Paths

| Consumer | Interface | Auth Mechanism | FGAC Enforcement | Latency Profile |
|---|---|---|---|---|
| Internal applications | AppSync (GraphQL) | IAM / Cognito JWT | AppSync resolvers + Verified Permissions | Sub-second (cached) to seconds |
| External partners | API Gateway → Apollo Federation (GraphQL) | API Key + OAuth2 | Apollo resolvers + Verified Permissions | Seconds |
| BI tools (QuickSight, Tableau) | Athena (SQL) | IAM / Lake Formation | LF-TBAC column/row filters | Seconds to minutes |
| Data engineers / analysts | Athena (SQL) | IAM / Lake Formation | LF-TBAC column/row filters | Seconds to minutes |
| ML pipelines | Athena / S3 direct | IAM | LF-TBAC + S3 Access Points | Varies |
| Other data products | Iceberg table read (Glue ETL / Flink) | IAM cross-account | LF resource shares | Batch |

### Consumption Architecture

```mermaid
graph TB
    subgraph "Consumers"
        APP[Internal Apps]
        EXT[External Partners]
        BI[BI Tools]
        ANALYST[Analysts]
        ML[ML Pipelines]
        DP_OTHER[Other Data Products]
    end

    subgraph "API Layer"
        subgraph "Internal GraphQL"
            AS[AppSync<br/>Internal API]
            CF[CloudFront<br/>Caching]
            COG[Cognito<br/>User Pool]
        end

        subgraph "External GraphQL"
            WAF[AWS WAF]
            APIGW[API Gateway<br/>Usage Plans + Throttling]
            AF[Apollo Federation Gateway<br/>ECS Fargate]
            SUB_CI[Subgraph: Customer Intel<br/>Lambda]
            SUB_TX[Subgraph: Transactions<br/>Lambda]
            SUB_META[Subgraph: Catalog Metadata<br/>Lambda]
        end
    end

    subgraph "Query Layer"
        ATH[Athena<br/>Workgroups per consumer tier]
        CACHE[ElastiCache Redis<br/>Query result cache]
    end

    subgraph "Data Layer"
        ICE[Iceberg Tables<br/>S3 Curated Zone]
        LF[Lake Formation<br/>FGAC enforcement]
        TV[Token Vaults<br/>Detokenisation]
    end

    APP -->|IAM/JWT| CF --> AS
    EXT -->|API Key + OAuth| WAF --> APIGW --> AF
    AF --> SUB_CI & SUB_TX & SUB_META
    BI & ANALYST -->|JDBC/ODBC| ATH
    ML -->|Athena SDK / S3 direct| ATH
    DP_OTHER -->|Glue/Flink read| ICE

    AS -->|VTL resolvers| ATH
    SUB_CI & SUB_TX -->|Athena SDK| ATH
    SUB_META -->|DynamoDB| DPC[Data Product Catalog]

    ATH -->|query| ICE
    LF -->|filter columns/rows| ATH
    AS & AF -->|detokenise if authorised| TV
    ATH -->|cache results| CACHE
```

### Caching Strategy by Tier

| Tier | CloudFront TTL | ElastiCache TTL | Rationale |
|---|---|---|---|
| Gold | No cache | 60 seconds | High freshness SLO, critical data |
| Silver | 60 seconds | 5 minutes | Balanced freshness vs cost |
| Bronze | 5 minutes | 15 minutes | Cost-optimised, relaxed freshness |

### Apollo Federation

Each domain owns a subgraph while presenting a unified schema to external partners:

```graphql
# Customer Intelligence subgraph
type Customer @key(fields: "id") {
  id: ID!
  segment: String
  lifetimeValue: Float
  region: String
  email: String
  jurisdiction: String
}

# Transactions subgraph — extends Customer
extend type Customer @key(fields: "id") {
  orders: [Order!]!
  totalSpend: Float
  lastOrderDate: String
}

type Order @key(fields: "orderId") {
  orderId: ID!
  customer: Customer!
  amount: Float
  currency: String
  status: OrderStatus
  createdAt: String
}
```

### External Partner Controls

| Control | Mechanism |
|---|---|
| Can only query subscribed products | Verified Permissions checks subscription status |
| Can only see their own data | Row-level filter: `partner_id = '{caller.partner_id}'` |
| Never see raw PII | Detokenisation permission denied for external group |
| Rate limited | API Gateway usage plans per partner |
| Geo-restricted | WAF geo-match rules aligned to partner jurisdiction |
| Schema introspection disabled | Apollo Gateway config: `introspection: false` in production |

### Athena Workgroups

| Workgroup | Consumer | Query Limit | Scan Limit | Cost Allocation |
|---|---|---|---|---|
| `wg-bi-gold` | BI dashboards on gold products | 50 concurrent | 10 TB/query | Charged to platform |
| `wg-analyst` | Data analysts (ad-hoc) | 10 concurrent | 1 TB/query | Charged to analyst's domain |
| `wg-ml-pipeline` | ML training jobs | 20 concurrent | 50 TB/query | Charged to ML team |
| `wg-external` | External (via Apollo subgraph) | 5 concurrent | 100 GB/query | Charged to partner |

---

## 6a. BI Integration — Tableau & Power BI

### BI Connectivity

| Dimension | Tableau | Power BI |
|---|---|---|
| **Connection method** | Athena JDBC/ODBC connector (native) | Athena ODBC via Power BI Gateway, or DirectQuery |
| **Authentication** | IAM role via SAML federation | Service principal → IAM role via Power BI Gateway |
| **Lake Formation FGAC** | Fully supported | Fully supported |
| **Recommendation** | Live for gold, extract for silver/bronze | DirectQuery for gold, Import for silver/bronze |

### BI Architecture

```mermaid
graph TB
    subgraph "BI Tools"
        subgraph "Tableau Estate (Existing)"
            TD[Tableau Desktop]
            TS[Tableau Server/Cloud]
        end
        subgraph "Power BI (Adopting)"
            PBD[Power BI Desktop]
            PBS[Power BI Service]
            PBG[Power BI Gateway]
        end
    end

    subgraph "Authentication"
        IDP[Corporate IdP] -->|SAML| COG[Cognito]
        COG -->|assume| IAM_T[IAM Role: tableau-bi-role]
        COG -->|assume| IAM_P[IAM Role: powerbi-bi-role]
    end

    subgraph "Query Layer"
        ATH_T[Athena Workgroup<br/>wg-bi-tableau]
        ATH_P[Athena Workgroup<br/>wg-bi-powerbi]
        LF[Lake Formation FGAC]
    end

    subgraph "Data Layer"
        SEM[Semantic Layer<br/>Athena Views]
        ICE[Iceberg Tables]
    end

    TD & TS -->|JDBC/ODBC| ATH_T
    PBD -->|ODBC via Gateway| PBG --> ATH_P
    PBS -->|DirectQuery via Gateway| PBG

    IAM_T --> LF --> ATH_T
    IAM_P --> LF --> ATH_P
    ATH_T & ATH_P -->|query| SEM --> ICE
```

### Semantic Layer

Athena views provide a stable, governed interface. Both Tableau and Power BI query the same views, ensuring consistency during migration.

- Registered as output ports on data products (format: `athena_view`)
- Governed by the same LF-TBAC
- Named using business glossary terms from Alation
- Versioned — breaking changes go through schema contract process

### BI Workgroups

| Workgroup | Tool | Mode | Concurrent Queries | Scan Limit | Result Reuse |
|---|---|---|---|---|---|
| `wg-bi-tableau-live` | Tableau | Live connection | 30 | 5 TB | 15 min |
| `wg-bi-tableau-extract` | Tableau | Extract refresh | 10 | 20 TB | Disabled |
| `wg-bi-powerbi-dq` | Power BI | DirectQuery | 30 | 5 TB | 15 min |
| `wg-bi-powerbi-import` | Power BI | Import refresh | 10 | 20 TB | Disabled |

---

## 6b. Jurisdictional Detokenisation for BI

### Core Rule

Detokenisation in BI is subject to the same Cedar policy enforcement as every other consumption path. A user must satisfy ALL of:

1. **Identity** — PII-certified and subscribed to the product
2. **Jurisdiction** — their session originates from an approved region for that data's jurisdiction
3. **Purpose** — the detokenisation has a declared lawful basis

If any condition fails, the field renders as the token value.

### Enforcement via Athena UDF

```mermaid
sequenceDiagram
    participant BI as BI Tool
    participant ATH as Athena
    participant LF as Lake Formation
    participant UDF as Detokenise UDF<br/>(Lambda)
    participant VP as Verified Permissions<br/>(Cedar)
    participant TV as Token Vault<br/>(Regional)
    participant AUDIT as Audit Log

    BI->>ATH: SELECT customer_id,<br/>  detokenise(email_token, 'EMAIL') as email,<br/>  segment<br/>FROM vw_customer_summary

    ATH->>LF: Apply column/row filters
    LF-->>ATH: Filtered scan

    ATH->>UDF: Invoke detokenise()<br/>with caller context

    UDF->>VP: isAuthorized?<br/>principal, detokenise, EMAIL,<br/>jurisdiction: EU, caller_region: eu-west-1

    alt All conditions met
        VP-->>UDF: ALLOW
        UDF->>TV: Resolve token (EU vault)
        TV-->>UDF: real@email.com
        UDF->>AUDIT: Log detokenisation
        UDF-->>ATH: real@email.com
    else Any condition fails
        VP-->>UDF: DENY
        UDF->>AUDIT: Log denied attempt
        UDF-->>ATH: TOK_EU_EMAIL_a8f3b2c1
    end

    ATH-->>BI: Result set
```

### Jurisdiction-to-Region Enforcement Map

| Data Jurisdiction | Permitted Detokenisation Regions | Token Vault Region |
|---|---|---|
| **EU** | `eu-west-1`, `eu-central-1` | `eu-west-1` |
| **India** | `ap-south-1` | `ap-south-1` |
| **Switzerland** | `eu-central-2` (Zurich) | `eu-central-2` |
| **US** | `us-east-1`, `us-west-2` | `us-east-1` |
| **Monaco** | `eu-west-3` (Paris) | `eu-west-3` |

### Regional UDF Deployment

The detokenisation Lambda UDF is deployed in each permitted region. The token prefix (`TOK_EU_`, `TOK_IN_`, etc.) tells the UDF which regional vault to call. The UDF verifies that its own execution region matches the jurisdiction's permitted region.

### Mixed-Jurisdiction Result Sets

A single query can span multiple jurisdictions. The UDF handles this row-by-row:

| Row | `jurisdiction` | `email_token` | Caller in `eu-west-1` | Caller in `us-east-1` |
|---|---|---|---|---|
| 1 | EU | `TOK_EU_EMAIL_a8f3b2c1` | `real@eu.com` | `TOK_EU_EMAIL_a8f3b2c1` |
| 2 | IN | `TOK_IN_EMAIL_c3d4e5f6` | `TOK_IN_EMAIL_c3d4e5f6` | `TOK_IN_EMAIL_c3d4e5f6` |
| 3 | US | `TOK_US_EMAIL_d7e8f9a0` | `TOK_US_EMAIL_d7e8f9a0` | `real@us.com` |
| 4 | CH | `TOK_CH_EMAIL_e1f2a3b4` | `TOK_CH_EMAIL_e1f2a3b4` | `TOK_CH_EMAIL_e1f2a3b4` |

---

## 7. Compute

### Compute Taxonomy

| Compute Type | Workload | Service | Scaling Model |
|---|---|---|---|
| **Query** | Ad-hoc SQL, BI dashboards, GraphQL resolution | Athena | Serverless, per-query |
| **Batch transform** | Hydration ETL, compaction, large joins | Glue ETL (Spark) | Serverless, DPU-based |
| **Streaming** | Real-time ingest, CDC processing | Managed Flink | Auto-scaled KPU |
| **Lightweight transform** | Tokenisation, small enrichments, UDFs | Lambda | Serverless, per-invocation |
| **Heavy analytics** | Complex ML feature engineering, large aggregations | EMR Serverless | Auto-scaled, Spark |
| **API compute** | Apollo Federation gateway | ECS Fargate | Task-based auto-scaling |
| **Orchestration** | Pipeline coordination, governance workflows | Step Functions | Serverless, per-transition |

### Compute Selection Decision Tree

```mermaid
flowchart TD
    START[New processing<br/>requirement] --> Q1{Data volume<br/>per run?}

    Q1 -->|< 100 MB| LAM[Lambda]
    Q1 -->|100 MB - 100 GB| Q2{Latency requirement?}
    Q1 -->|> 100 GB| Q3{Frequency?}

    Q2 -->|Real-time < 1 min| FLINK[Managed Flink]
    Q2 -->|Near real-time 1-5 min| GLUE_S[Glue Streaming]
    Q2 -->|Batch > 5 min OK| GLUE[Glue ETL]

    Q3 -->|Hourly or more| EMR[EMR Serverless]
    Q3 -->|Daily or less| GLUE2[Glue ETL]
```

### Glue ETL Configuration per Product Tier

| Tier | Worker Type | Max DPUs | Job Timeout | Retry Strategy |
|---|---|---|---|---|
| Gold | G.2X | 100 | 4 hours | 3 retries, exponential backoff |
| Silver | G.1X | 50 | 2 hours | 2 retries |
| Bronze | G.1X | 20 | 1 hour | 1 retry |

### Compute Quotas per Domain

| Resource | Per-Domain Quota | Platform Reserve |
|---|---|---|
| Athena concurrent queries | 20 | 50 (shared BI + platform) |
| Glue max concurrent DPUs | 150 | 100 (maintenance jobs) |
| Lambda concurrent executions | 200 | 100 (platform functions) |
| EMR Serverless max vCPUs | 200 | 100 (backfill reserve) |
| Step Functions concurrent executions | 50 | 20 (governance workflows) |

### Cost Allocation Tags

| Tag Key | Example Values | Purpose |
|---|---|---|
| `domain` | `customer-intelligence` | Chargeback to domain team |
| `data-product` | `customer-360` | Per-product cost tracking |
| `compute-tier` | `hydration`, `serving`, `maintenance` | Cost by lifecycle stage |
| `environment` | `prod`, `staging` | Environment isolation |
| `consumer` | `bi-tableau`, `partner-acme` | Consumption cost attribution |

---

## 7a. Multi-Engine Compute — Snowflake & Starburst

### Engine Comparison

| Dimension | Athena | Snowflake (External Storage) | Starburst |
|---|---|---|---|
| **Deployment** | Serverless (AWS-managed) | SaaS (Snowflake-managed) | EKS or Starburst Galaxy (SaaS) |
| **Cost model** | Per TB scanned | Credit-based | Node-based (EKS) or credit-based (Galaxy) |
| **Idle cost** | Zero | Zero (if auto-suspends) | Node cost (EKS) or zero (Galaxy) |
| **Iceberg support** | Native, first-class | External Catalog (read) | Native, first-class |
| **Federation** | Limited | Limited | Excellent (core strength) |
| **FGAC mechanism** | Lake Formation (native) | RBAC + DDM (translated from Cedar) | BIAC + masking (translated from Cedar) |
| **Detokenisation** | Lambda UDF (direct) | External Function via API GW | Lambda UDF (direct) |
| **Concurrency** | Moderate | High (multi-cluster warehouse) | High (auto-scaled workers) |
| **Catalog source** | Domain Glue Catalog (direct) | Central Glue Mesh Catalog | Domain or Central Glue Catalog |
| **Data location** | S3 (in-place) | S3 (in-place, via External Volume) | S3 (in-place) |
| **Write capability** | Yes | Read-only (external catalog mode) | Yes |

### Engine Selection Decision Tree

```mermaid
flowchart TD
    START[Query requirement] --> Q1{Need to join across sources?}

    Q1 -->|Yes: Iceberg + RDS or API| SB[Starburst]
    Q1 -->|No: Iceberg only| Q2{Concurrency requirement?}

    Q2 -->|Low: < 20 concurrent| ATH[Athena]
    Q2 -->|High: 50+ concurrent| Q3{Partner already on Snowflake?}

    Q3 -->|Yes| SF[Snowflake]
    Q3 -->|No| Q4{Need sub-second interactive?}

    Q4 -->|Yes| SB
    Q4 -->|No| ATH
```

### FGAC Consistency Across Engines

Cedar remains the single source of truth. The governance engine translates Cedar policies into engine-native controls:

| Cedar Policy Concept | Lake Formation | Snowflake | Starburst |
|---|---|---|---|
| Column hide | LF column filter | Dynamic Data Masking → NULL | Column mask → NULL |
| Column mask (tokenised) | LF cell-level security | DDM policy returning token | Column mask function |
| Row filter | LF row filter expression | Row Access Policy | Row filter predicate |
| Role-based access | LF tag-based grant | Snowflake role (SCIM-synced) | Starburst role (IdP group) |
| Detokenisation | Lambda UDF (direct) | External Function → API GW → Lambda | Lambda UDF (direct) |

### Cross-Engine Consistency Verification

Automated test suite runs standardised queries using test identities across all three engines. Any discrepancy in returned columns, rows, or masking triggers a P1 alert.

---

## 7b. Snowflake — Enterprise External Storage Integration

Snowflake connects via external storage integrations against the **central enterprise Glue mesh catalog**, not domain-level catalogs.

```mermaid
graph TB
    subgraph "Domain Accounts"
        subgraph "Domain: Customer Intelligence"
            S3_CI[S3: Iceberg data files]
            GC_CI[Glue Catalog: CI]
        end
        subgraph "Domain: Transactions"
            S3_TX[S3: Iceberg data files]
            GC_TX[Glue Catalog: TX]
        end
        subgraph "Domain: Risk"
            S3_RK[S3: Iceberg data files]
            GC_RK[Glue Catalog: Risk]
        end
    end

    subgraph "Platform Account (Hub)"
        LF[Lake Formation<br/>Cross-account shares]
        CGC[Central Enterprise<br/>Glue Mesh Catalog]
    end

    subgraph "Snowflake (Enterprise)"
        CI_SF[Catalog Integration<br/>→ Central Glue Mesh Catalog]
        EV_CI[External Volume → CI S3]
        EV_TX[External Volume → TX S3]
        EV_RK[External Volume → Risk S3]
        EIT[External Iceberg Tables]
    end

    GC_CI -->|resource share| LF --> CGC
    GC_TX -->|resource share| LF
    GC_RK -->|resource share| LF

    CGC -->|catalog integration| CI_SF --> EIT
    S3_CI --> EV_CI --> EIT
    S3_TX --> EV_TX --> EIT
    S3_RK --> EV_RK --> EIT
```

### IAM Trust Chain

- Snowflake gets **read-only** access to curated zone prefixes only
- IAM role scoped per S3 prefix, not per bucket
- ExternalId condition prevents confused deputy attacks
- Role lives in the **platform account** — platform team controls access

### Snowflake Object Hierarchy

```
Snowflake Account (Enterprise)
└── Database: DATA_PRODUCTS
    ├── Schema: CUSTOMER_INTELLIGENCE
    │   ├── External Iceberg Table: CUSTOMER_PROFILE
    │   └── Secure View: VW_CUSTOMER_SUMMARY (masking applied)
    ├── Schema: TRANSACTIONS
    │   ├── External Iceberg Table: ORDERS
    │   └── Secure View: VW_ORDER_METRICS (masking applied)
    └── Schema: RISK
        ├── External Iceberg Table: EXPOSURE
        └── Secure View: VW_RISK_SUMMARY (masking applied)
```

External Iceberg Tables are raw external objects. **Secure Views** sit on top with Dynamic Data Masking and Row Access Policies applied.

### Detokenisation in Snowflake

Snowflake uses External Functions via API Gateway. A routing UDF directs detokenisation calls to the correct regional endpoint based on the token prefix (`TOK_EU_`, `TOK_IN_`, etc.). The Lambda behind each regional endpoint performs the same Cedar evaluation as Athena and Starburst paths.

### Metadata Refresh

| Method | Latency | Use Case |
|---|---|---|
| **Auto-refresh** | Minutes | Silver/Bronze products |
| **SNS notification** | Seconds | Gold products with tight freshness SLOs |
| **Manual refresh** | On-demand | Ad-hoc |

---

## 8. Observability

### SLO Framework

| SLO Type | Metric | Measurement Source | Evaluation Frequency |
|---|---|---|---|
| **Freshness** | Time since last Iceberg commit | Iceberg snapshot metadata | Every 5 minutes |
| **Completeness** | Record count vs expected | Glue DQ + Iceberg metadata | Per hydration run |
| **Accuracy** | Quality expectation pass rate | Glue DQ / Deequ results | Per hydration run |
| **Availability** | % successful queries vs total | Query engine logs | Rolling 5-minute window |
| **Latency** | P50/P95/P99 query response time | Query engine logs + GraphQL resolver timing | Rolling 5-minute window |
| **Schema stability** | Days since last breaking change | Glue Schema Registry | On schema change |

### Data Product Health Score

| Health Score | Meaning | Criteria |
|---|---|---|
| **100** | All SLOs met, no recent breaches | All SLOs healthy for 7+ days |
| **75-99** | All SLOs currently met, recent breach history | All SLOs healthy, but breach in last 7 days |
| **50-74** | One or more SLOs degraded | Warning-level SLO breach active |
| **25-49** | Critical SLO breached | One critical SLO in breach |
| **0-24** | Multiple critical SLOs breached | Multiple critical breaches or product down |

Health score is shown in the Data Product Catalog, synced to Alation as a quality badge, and available via the GraphQL API.

### Alert Routing

| Severity | Trigger | Routing | Auto-Remediation |
|---|---|---|---|
| **P1** | Gold + Freshness/Availability breach | PagerDuty page: domain + platform on-call | Restart pipeline / increase concurrency |
| **P2** | Gold + Completeness/Accuracy breach | PagerDuty alert: domain on-call | - |
| **P3** | Silver + any breach | Slack: #data-quality, domain channel | - |
| **P4** | Bronze + any breach | Daily email digest to domain owner | - |

### Observability as a Data Product

All observability data is itself published as a gold-tier data product with output ports for: SLO evaluations, query logs, detokenisation audit, policy sync status, and cost attribution. Domain teams can subscribe to build their own dashboards.

---

## 9. Audit

### Audit Principles

| Principle | Implementation |
|---|---|
| **Immutable** | Iceberg tables with write-once S3 Object Lock |
| **Complete** | Every data access, policy decision, lifecycle event captured |
| **Tamper-evident** | Daily Merkle hash, stored in a separate account |
| **Jurisdictionally aware** | Audit records stored per residency rules |
| **Queryable** | Audit data is a data product, queryable via Athena |
| **Retained** | Per regulation (GDPR: processing + 1yr, DPDPA: 3yr, US: 7yr) |

### Audit Event Taxonomy

- **Data Access Events:** Query executed, detokenisation attempt, data download/export
- **Governance Events:** Policy change, subscription lifecycle, classification change
- **Data Lifecycle Events:** Product published, hydration run, erasure executed
- **Security Events:** Authentication, authorisation denial, anomaly detected

### Audit Record Schema

Common envelope with event-specific detail:

```yaml
audit_event:
  event_id: "uuid-v4"
  event_type: "DATA_ACCESS | DETOKENISATION | POLICY_CHANGE | ERASURE | ..."
  timestamp: "2024-03-15T14:23:01.456Z"
  actor:
    identity: "arn:aws:iam::123:role/analyst-eu"
    user: "jane.smith@company.com"
    idp_groups: ["pii-certified", "customer-intelligence"]
    source_ip: "10.0.1.42"
    user_agent: "Tableau/2024.1"
  resource:
    data_product: "customer-360"
    domain: "customer-intelligence"
    output_port: "customer_profile"
    tier: "gold"
    classification: "confidential"
    jurisdiction: "EU"
  location:
    engine: "athena | snowflake | starburst | appsync | apollo"
    region: "eu-west-1"
    workgroup: "wg-bi-tableau-live"
  decision:
    result: "ALLOW | DENY"
    policy_matched: "pii-certified-jurisdiction-match"
    cedar_policy_version: "v2024-03-14"
  detail: { ... }
```

### Jurisdictional Audit Residency

| Audit Event Jurisdiction | Storage Region | Retention |
|---|---|---|
| EU | `eu-west-1` | Duration of processing + 1 year |
| India | `ap-south-1` | 3 years (DPDPA) |
| Switzerland | `eu-central-2` | 10 years (nFADP + sector) |
| US | `us-east-1` | 7 years (SOX/GLBA default) |
| Monaco | `eu-west-3` | 5 years (CCIN guidelines) |
| Non-jurisdictional | `eu-west-1` | 3 years |

### Immutability

- **S3 Object Lock (GOVERNANCE mode)** prevents deletion/overwrite
- **Daily Merkle hash chain** in separate account provides tamper evidence
- **Weekly verification** recomputes hashes and compares

### Compliance Reports

| Report | Regulation | Frequency |
|---|---|---|
| Data Subject Access Report | GDPR Art 15, DPDPA | On-demand |
| Detokenisation Log | All jurisdictions | Monthly |
| Erasure Compliance | GDPR Art 17, DPDPA | On-demand |
| Cross-border Transfer Log | GDPR Ch V, DPDPA | Quarterly |
| Policy Change History | Internal governance | Monthly |
| FGAC Consistency Report | Internal governance | Weekly |

### Anomaly Detection Rules

| Rule | Threshold |
|---|---|
| Volume spike | 10x normal query count in 1-hour window |
| Off-hours access | Query outside user's normal working hours |
| New jurisdiction access | User accessing jurisdiction they've never queried |
| Bulk detokenisation | >100 detokenisations in single session |
| Denied access spike | >5 denials in 10 minutes from same user |

---

## 10. Security

### Defence in Depth

| Layer | Controls |
|---|---|
| **Perimeter** | AWS WAF (OWASP rules, geo-blocking, rate limiting), Shield Advanced, Route 53 DNSSEC |
| **Network** | VPC isolation, private subnets for all compute, VPC endpoints for all AWS services, NACLs, security groups |
| **Identity & Access** | Corporate IdP federation, Cognito, IAM roles per-service per-domain, Lake Formation FGAC, Verified Permissions |
| **Data Protection** | KMS encryption per-domain per-jurisdiction, S3 SSE-KMS, TLS 1.3 everywhere, tokenisation |
| **Detection & Response** | GuardDuty, Security Hub, Macie (PII discovery), CloudTrail, custom anomaly detection |

### Encryption Strategy

| Key Type | Scope | Purpose |
|---|---|---|
| Per-domain KMS keys | Domain data at rest | Domain isolation |
| Per-jurisdiction token vault keys | Token vault encryption | Jurisdictional isolation |
| Per-jurisdiction HMAC keys | Token generation | Deterministic tokenisation |
| Platform keys | Audit, catalog | Platform data protection |

Key management rules:
- One KMS key per domain for data at rest
- One KMS key per jurisdiction for token vaults
- Separate HMAC keys per jurisdiction for token generation
- Annual key rotation enabled on all keys
- Key policy restricts decrypt to specific IAM roles
- Audit key managed by security team, not platform team

### IAM Role Strategy

| Role | Trust | Permissions |
|---|---|---|
| `dp-platform-admin` | Platform team IdP group | Full platform management, no data access |
| `dp-domain-engineer-<domain>` | Domain team IdP group | Glue, S3, Step Functions within domain |
| `dp-domain-owner-<domain>` | Domain owner IdP group | Catalog mutations for owned products |
| `dp-consumer-analyst` | Analyst IdP group | Athena query, LF-scoped data access |
| `dp-consumer-pii-certified` | PII-certified IdP group | Above + detokenisation UDF invocation |
| `dp-bi-tableau` | Tableau Server service account | Athena workgroup `wg-bi-tableau-*` |
| `dp-bi-powerbi` | Power BI Gateway service account | Athena workgroup `wg-bi-powerbi-*` |
| `dp-external-partner-<id>` | Cognito M2M OAuth | API Gateway invocation, scoped to partner products |
| `dp-snowflake-reader` | Snowflake storage integration principal | S3 GetObject on curated prefixes (read-only) |
| `dp-apollo-gateway` | ECS task role | Athena, DynamoDB, Verified Permissions, Token Vaults |
| `dp-governance-engine` | Step Functions execution role | LF admin, Verified Permissions admin, Snowflake API, Starburst API |

All roles use permission boundaries to cap maximum privileges.

### S3 Bucket Security

- Block Public Access: enabled (account-level and bucket-level)
- Bucket Policy: deny non-SSL, deny non-VPC-endpoint
- Encryption: SSE-KMS with domain-specific key
- Versioning: enabled on all curated and audit buckets
- Object Lock: GOVERNANCE mode on audit buckets
- Access Logging: to dedicated logging bucket
- Lifecycle: Intelligent-Tiering on curated, Glacier on audit after retention

### Threat Detection

- **GuardDuty**: S3 protection, EKS protection (Starburst), Lambda protection
- **Macie**: Scheduled PII scans on curated zone — detects unintentional PII leakage
- **Security Hub**: Aggregates findings via ASFF
- **IAM Access Analyzer**: Detects external access
- **Custom anomaly detection**: Rules from audit layer

Automated response via Step Functions: isolate (revoke IAM sessions), block (WAF IP deny), alert (PagerDuty P1), snapshot (preserve evidence).

### Security Review Cadence

| Review | Frequency | Owner |
|---|---|---|
| IAM role policy review | Quarterly | Platform + Security |
| KMS key access audit | Quarterly | Security |
| Cedar policy review | Monthly | Governance council |
| Macie findings review | Weekly | Platform team |
| GuardDuty findings triage | Daily (automated) | Security team |
| Penetration test (external API) | Annually | External auditor |
| Cross-engine FGAC consistency | Weekly (automated) | Platform team |

---

## 11. Governance

### Governance Model: Federated with Central Standards

```mermaid
graph TB
    subgraph "Central Governance"
        DGC[Data Governance Council<br/>CDO, domain owners, legal, security, DPO]
        DGC -->|Define standards| STD[Classification, SLO tiers, naming]
        DGC -->|Approve policies| POL[Cross-domain Cedar policies]
        DGC -->|Escalation authority| ESC[Gold + restricted approvals]
    end

    subgraph "Platform Governance"
        PT[Data Platform Team]
        PT -->|Infrastructure standards| INFRA[Terraform modules, guardrails]
        PT -->|Tooling & automation| TOOL[Catalog, FGAC, observability]
        PT -->|Enforcement| ENFORCE[Policy sync, consistency checks]
        PT -->|Enablement| ENABLE[Self-service templates, docs]
    end

    subgraph "Domain Governance"
        DO[Domain Owner + Steward]
        DO -->|Own data products| OWN[Quality, freshness, schema]
        DO -->|Classify data| CLASS[PII tagging, jurisdiction]
        DO -->|Approve subscriptions| APPROVE_SUB[Silver + confidential]
        DO -->|Local policies| LOCAL_POL[Domain-specific Cedar]
    end

    DGC -->|sets standards| PT
    DGC -->|appoints & oversees| DO
    PT -->|provides tooling| DO
    DO -->|reports compliance| DGC
```

### Data Product Lifecycle

```mermaid
stateDiagram-v2
    [*] --> DRAFT: Domain engineer creates
    DRAFT --> IN_REVIEW: Submit for review
    IN_REVIEW --> DRAFT: Reviewer requests changes
    IN_REVIEW --> PUBLISHED: Approved
    PUBLISHED --> PUBLISHED: New version published
    PUBLISHED --> DEPRECATED: Owner initiates sunset
    DEPRECATED --> SUNSET: Sunset date reached + all subscribers migrated
    SUNSET --> [*]: Data archived/deleted
```

### Lifecycle Approval Matrix

| Transition | Bronze | Silver | Gold |
|---|---|---|---|
| DRAFT → IN_REVIEW | Automated validation | Automated validation | Automated validation |
| IN_REVIEW → PUBLISHED | Auto-approved | Domain owner approval | Governance council approval |
| Schema evolution (BACKWARD) | Auto-approved | Domain owner notified | Domain owner approval |
| Schema evolution (BREAKING) | Domain owner approval | Governance council approval | Governance council + CDO |
| PUBLISHED → DEPRECATED | Domain owner | Domain owner + subscriber notification | Governance council + all subscriber acknowledgement |
| Subscription (public/internal) | Auto-approved | Auto-approved | Domain owner approval |
| Subscription (confidential) | Domain owner | Domain owner + steward | Governance council |
| Subscription (restricted) | Governance council + CDO | Governance council + CDO | Governance council + CDO + DPO |

### Policy-as-Code CI/CD

Cedar policies managed in Git with full CI/CD:
1. Policy author writes Cedar, opens PR
2. CI: syntax validation, test suite, impact analysis, dry-run translation
3. Approval: domain owner (domain policy) or governance council (cross-domain)
4. CD: deploy to Verified Permissions, translate + sync to LF/Snowflake/Starburst, consistency check
5. Auto-rollback on consistency failure

### Data Product Maturity Model

| Level | Name | Criteria |
|---|---|---|
| **L1** | Managed | Manifest, owner, output port, schema defined |
| **L2** | Governed | Classification, Cedar policy, LF tags, SLOs declared |
| **L3** | Quality-assured | DQ expectations defined and passing, freshness SLO met >95% |
| **L4** | Operationalised | Observability complete, alerting configured, runbook documented |
| **L5** | Optimised | Cost-optimised, multi-engine FGAC verified, Alation curated, consumer satisfaction tracked |

**Minimum maturity for publication:**
- Bronze: L1 (Managed)
- Silver: L2 (Governed)
- Gold: L3 (Quality-assured), L4 within 90 days

### Governance Cadences

| Meeting | Frequency | Attendees |
|---|---|---|
| Governance council | Monthly | CDO, domain owners, DPO, security, platform |
| Domain review | Bi-weekly | Domain owner, steward, engineers |
| Platform standup | Weekly | Platform engineering team |
| Incident review | Per incident (within 5 days) | Relevant domain + platform + security |
| Quarterly business review | Quarterly | CDO, domain owners, executive sponsors |

### Self-Service Operations

| Operation | Self-Service? | Mechanism |
|---|---|---|
| Register data product (DRAFT) | Yes | `mutation registerDataProduct` |
| Submit for review | Yes | `mutation submitForReview` |
| Publish version (BACKWARD) | Yes (bronze/silver) | `mutation publishVersion` |
| Request subscription | Yes | `mutation requestSubscription` |
| View product health/SLOs | Yes | `query getDataProduct { health, slos }` |
| Update quality expectations | Yes | `mutation updateQualityExpectations` |
| Classify a column | Yes | `mutation updateClassification` |
| Create domain Cedar policy | Yes | Git PR to `cedar-policies/<domain>/` |
| View audit events (own products) | Yes | Athena on `wg-audit-domain` |
| View cost attribution | Yes | `query getCostReport(domain)` |

### Deprecation & Sunset

- Grace periods: Gold 6 months, Silver 3 months, Bronze 1 month
- All active subscribers notified at deprecation
- Monthly reminders until sunset date
- Cannot sunset while subscribers remain — extend or escalate to governance council
- On sunset: revoke all grants, archive data to Glacier Deep Archive
