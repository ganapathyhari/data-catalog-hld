# 20PB Data Catalog — High Level Design

!!! info "Document Status"
    **Version:** 1.0 | **Status:** Draft | **Owner:** Solution Architecture Team

---

## Executive Summary

**Challenge:** Catalog and categorize **20PB of heterogeneous data** — genomics, research, and generalist — spanning structured, unstructured, and semi-structured formats stored across **IBM Storage Scale** (hot/warm) and **IBM Tape** (cold/archive).

**Core Requirements:**

- Automated metadata extraction from filesystem attributes and file contents
- API-driven cataloguing and categorisation with a user-friendly discovery interface
- All workloads must run on **Red Hat OpenShift** (on-premises)
- IBM-native tooling preferred; third-party where IBM has no fit

**Secondary Requirements (nice to have):**

- Multi-tenancy isolation per research team
- Data lineage tracking
- Governance policies and PII masking
- Semantic / natural language discovery

---

## Proposed Solution Stack

| Layer | Component | Product | IBM-Native |
|---|---|---|---|
| Container Platform | Orchestration | Red Hat OpenShift | ✅ |
| Ingestion | Filesystem crawl & extraction | Apache NiFi | ➡️ OSS |
| Orchestration | Pipeline scheduling | Astronomer Airflow (Software) | ➡️ 3rd party |
| ETL & Lineage | Complex transforms | IBM DataStage | ✅ |
| AI Enrichment | Classification, PII, quality | IBM watsonx.ai | ✅ |
| Primary Catalog | All asset types + governance | IBM Knowledge Catalog (CP4D) | ✅ |
| Structured Catalog | Lakehouse / tabular assets | IBM watsonx.data + Hive Metastore | ✅ |
| Vector Store | Semantic similarity search | DataStax AstraDB (watsonx.data Premium) | ✅ |
| NL Query Layer | Natural language → structured query | DataStax Langflow (watsonx.data Premium) | ✅ |
| Event Streaming | Real-time metadata events | DataStax Astra Streaming (watsonx.data Premium) | ✅ |
| Distributed Store | High-throughput metadata KV | DataStax Cassandra (watsonx.data Premium) | ✅ |

!!! success "Key Outcome"
    Enterprise-grade data catalog with 95%+ automated metadata extraction, sub-second discovery queries, multi-tenant governance — running entirely on-premises on OpenShift with a strongly IBM-native stack.

---

## Document Structure

| Section | Contents |
|---|---|
| [Architecture Overview](architecture/overview.md) | Full system diagram, layer breakdown |
| [Storage Integration](architecture/storage-integration.md) | IBM Storage Scale + Tape connectivity |
| [OpenShift Deployment](architecture/deployment.md) | Namespace design, operators, pod topology |
| [Ingestion Pipeline](components/ingestion.md) | NiFi vs Airflow analysis, processor map |
| [Data Catalog](components/catalog.md) | IKC vs OpenMetadata vs watsonx.data trade-offs |
| [Semantic Layer](components/semantic-layer.md) | DataStax Langflow, AstraDB, NL query design |
| [File Formats](components/file-formats.md) | Full format inventory, metadata schema per type |
| [Trade-off Analysis](decisions/trade-offs.md) | All architectural decision records |
| [Phased Roadmap](decisions/roadmap.md) | Delivery phases and milestones |
