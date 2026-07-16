# Architecture Overview

## High-Level System Architecture

The platform is organised into **five horizontal layers**, each running as containerised workloads on Red Hat OpenShift, communicating with physical storage at the base.

```mermaid
graph TB
    subgraph STORAGE["📦 Physical Storage Tier"]
        SS["IBM Storage Scale\n20PB Hot/Warm\nNFS / GPFS"]
        TAPE["IBM Tape\nArchive / Cold\nvia GPFS HSM"]
    end

    subgraph INGEST["🔄 Layer 1 — Ingestion and Extraction"]
        NIFI["Apache NiFi\n• Filesystem crawler\n• Content parser\n• OCR / NLP\n• Format handlers"]
        DS["IBM DataStage\n• Complex transforms\n• Lineage capture\n• Batch ETL"]
        WX_AI["IBM watsonx.ai\n• Auto-categorisation\n• Quality scoring\n• PII detection\n• Relationship inference"]
    end

    subgraph CATALOG["🗄️ Layer 2 — Metadata Storage and Catalog"]
        PG["PostgreSQL\n• Metadata store\n• Lineage graph\n• Governance records"]
        CASSANDRA["DataStax Cassandra\n• High-throughput KV\n• Billion-file scale"]
        IKC["IBM Knowledge Catalog\n• Central catalog\n• REST and GraphQL APIs\n• Governance policies\n• Multi-tenancy"]
        WXD["watsonx.data\n• Hive Metastore\n• Structured tables\n• Open Lakehouse"]
    end

    subgraph SEMANTIC["🧠 Layer 3 — Semantic and Context Layer"]
        LANGFLOW["DataStax Langflow\n• NL to structured query\n• RAG pipeline\n• Ontology mapping"]
        ASTRA_DB["DataStax AstraDB\n• Vector embeddings\n• Semantic similarity\n• Fuzzy search"]
        ASTRA_STR["DataStax Astra Streaming\n• Real-time metadata events\n• NiFi to catalog pipeline"]
    end

    subgraph UI["🖥️ Layer 4 — User Interface and APIs"]
        IKC_UI["IBM Knowledge Catalog UI\n• Search and Discovery\n• Lineage visualisation\n• Governance workflows"]
        REST["REST APIs\n• Metadata CRUD\n• Search\n• Lineage"]
        GQL["GraphQL APIs\n• Complex queries\n• Relationship traversal"]
        NL_UI["Natural Language Interface\n• Plain English queries\n• watsonx.ai powered"]
    end

    subgraph USERS["👥 Consumers"]
        RESEARCHER["Research Teams"]
        ADMIN["Data Stewards"]
        APP["Downstream Applications"]
    end

    SS -->|NFS/GPFS mount| NIFI
    TAPE -->|HSM connector| NIFI
    SS -->|Batch structured data| DS
    NIFI --> WX_AI
    DS --> WX_AI
    NIFI -->|Events| ASTRA_STR
    ASTRA_STR --> CASSANDRA
    WX_AI --> PG
    WX_AI --> ASTRA_DB
    WX_AI --> IKC
    PG --> IKC
    CASSANDRA --> IKC
    WXD -->|Schema sync| IKC
    IKC --> LANGFLOW
    LANGFLOW --> ASTRA_DB
    LANGFLOW -->|Structured query| IKC
    IKC --> IKC_UI
    IKC --> REST
    IKC --> GQL
    LANGFLOW --> NL_UI
    IKC_UI --> RESEARCHER
    IKC_UI --> ADMIN
    REST --> APP
    GQL --> APP
    NL_UI --> RESEARCHER
```

---

## Layer Responsibilities

### Layer 1 — Ingestion & Extraction

| Component | Role |
|---|---|
| **Apache NiFi** | Filesystem crawl, file content extraction, format routing, tape stub detection, provenance chain |
| **IBM DataStage** | Structured/tabular ETL, complex transformation, deep lineage capture |
| **IBM watsonx.ai** | AI-powered enrichment — auto-classification, PII detection, quality scoring, relationship inference |
| **Astronomer Airflow** | Orchestrates when NiFi runs re-scans, schedules watsonx.ai batch jobs, manages tape recall windows |

### Layer 2 — Metadata Storage & Catalog

| Component | Role |
|---|---|
| **IBM Knowledge Catalog** | Primary catalog for ALL asset types — files, tables, models; governance, lineage, multi-tenancy |
| **IBM watsonx.data + Hive Metastore** | Structured/tabular catalog for Parquet, Iceberg, CSV — syncs schemas into IKC |
| **PostgreSQL** | Relational metadata persistence (IKC and OpenMetadata native backend) |
| **DataStax Cassandra** | High-throughput key-value metadata lookups at billion-file scale |

### Layer 3 — Semantic & Context Layer

| Component | Role |
|---|---|
| **DataStax Langflow** | Visual RAG pipeline builder — translates natural language queries to structured catalog queries |
| **DataStax AstraDB** | Managed vector store — stores content embeddings, powers similarity and semantic search |
| **DataStax Astra Streaming** | Real-time event backbone — streams metadata events from NiFi into the catalog |

!!! note "DataStax & IBM"
    DataStax was acquired by IBM in 2024 and is now part of **IBM watsonx.data Premium**. Langflow, AstraDB, and Astra Streaming are all available as IBM-native components within the watsonx.data premium tier.

### Layer 4 — User Interface & APIs

| Component | Role |
|---|---|
| **IBM Knowledge Catalog UI** | Primary interface for researchers and data stewards — browse, search, lineage, governance |
| **REST API** | `/api/v1/` — full CRUD for all asset types, programmatic metadata management |
| **GraphQL API** | Multi-hop relationship queries, complex entity graphs |
| **NL Interface** | Plain English querying via Langflow + watsonx.ai — for non-technical researchers |

---

## Multi-Tenancy Design

```mermaid
graph TD
    subgraph OCP["OpenShift — Namespace Isolation"]
        NS_A["Namespace: genomics-team-a\n• Own catalog domain\n• RBAC policies\n• Storage quota"]
        NS_B["Namespace: research-team-b\n• Own catalog domain\n• RBAC policies\n• Storage quota"]
        NS_C["Namespace: shared-generalist\n• Public/shared data\n• Read-only for all"]
    end
    subgraph IKC_MT["IBM Knowledge Catalog — Multi-Tenancy"]
        DOMAIN_A["Data Domain: Genomics"]
        DOMAIN_B["Data Domain: Research"]
        DOMAIN_C["Data Domain: Generalist"]
    end
    NS_A --> DOMAIN_A
    NS_B --> DOMAIN_B
    NS_C --> DOMAIN_C
```

- **OpenShift RBAC** — enforces namespace-level isolation per team
- **IKC Projects/Catalogs** — domain-level metadata access control
- **watsonx.ai** — queries filtered by user's team membership at inference time
- **Storage Scale ACLs** — file-level access control independent of catalog

---

## Data Lineage Model

```mermaid
flowchart LR
    A["Raw File\nStorage Scale/Tape"] --> B["NiFi Crawl Event"]
    B --> C["Metadata Extract\nDataStage/NiFi"]
    C --> D["watsonx.ai Enrichment"]
    D --> E["IKC Asset Record"]
    E --> F["Published Dataset"]
    F --> G["Downstream Analysis\nPipeline"]
```

All transitions are captured as lineage edges in IBM Knowledge Catalog's lineage graph — surfaced via the UI and REST lineage endpoint. IBM DataStage feeds deep column-level lineage for structured data.

---

## Non-Functional Requirements

| NFR | Target | Design Approach |
|---|---|---|
| **Crawl throughput** | 20PB initial in ≤ 30 days | 10-node NiFi cluster, parallel GPFS scan |
| **Incremental update latency** | < 5 minutes for new files | GPFS callbacks → NiFi event trigger |
| **Search response time** | < 1 second keyword search | IKC built-in Elasticsearch index |
| **Semantic search latency** | < 3 seconds | AstraDB ANN index, pre-computed embeddings |
| **API availability** | 99.9% | OpenShift pod replicas + health probes |
| **Metadata storage** | ~5–50 KB per file | PostgreSQL + partitioning; ~50TB metadata est. |
| **Security** | RBAC, TLS 1.3, no secrets in code | OpenShift Secrets, cert-manager, Vault |
| **Data residency** | On-premises only | All workloads inside OpenShift cluster |
