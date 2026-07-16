# Trade-off Analysis

## Architectural Decision Records

This page documents all key architectural decisions made during the design of the 20PB Data Catalog platform, including the options evaluated, trade-offs considered, and rationale for each decision.

---

## ADR-001: Ingestion Tool — Apache NiFi vs Astronomer Airflow

**Decision:** Use **Apache NiFi** for data ingestion and **Astronomer Airflow (Software)** for pipeline orchestration. The two tools are complementary, not competing.

**Context:** 20PB of heterogeneous files across IBM Storage Scale and IBM Tape requiring automated metadata extraction. Files include binary genomics formats (BAM, HDF5, DICOM), compressed archives, and tape stub files.

### Options Evaluated

| Dimension | Apache NiFi | Astronomer Airflow |
|---|---|---|
| **Primary purpose** | Real-time data flow, routing, transformation | Workflow orchestration — scheduling DAGs |
| **Data handling** | Moves actual file bytes and metadata natively | Orchestrates tasks that move data — does not move data itself |
| **Filesystem crawling** | Native `ListFile`, `FetchFile`, `GetSFTP` processors | No native filesystem crawler — requires custom operators |
| **Backpressure and flow control** | Built-in — queues, prioritisation, rate limiting | Not applicable |
| **Content extraction** | Native Tika, custom processors per format | Requires custom Python operators per file type |
| **Event-driven triggers** | Yes — GPFS callbacks, sub-minute latency | Primarily schedule-based cron; event-driven is complex |
| **Tape stub detection** | Reads filesystem xattr natively | Cannot distinguish stub vs hot file without custom code |
| **Built-in data provenance** | Yes — per FlowFile provenance chain | No — must be captured externally |
| **Scale at 20PB** | Cluster mode — linear horizontal scale | Viable with custom engineering |
| **OpenShift deployment** | StatefulSet — fully on-prem | Astronomer Software — fully on-prem operator available |

### Decision Rationale

NiFi is the right tool because the core problem is **file-level data movement and content extraction at scale** — a use case NiFi was purpose-built for. Airflow adds value as the scheduling and orchestration layer on top.

**Airflow (Astronomer Software) is used for:**

- Scheduling when NiFi runs full filesystem re-scans
- Orchestrating watsonx.ai batch enrichment jobs
- Managing tape recall batch windows (sequential by cartridge)
- Monitoring pipeline health and SLA compliance
- Coordinating downstream analysis pipeline dependencies

!!! warning "Astronomer Software — Not Cloud"
    Astronomer **Software** (self-hosted on OpenShift) must be used, not Astronomer Cloud. The SaaS version introduces an external dependency and data residency risk for genomics data.

---

## ADR-002: Primary Catalog — OpenMetadata vs watsonx.data Hive Metastore vs IBM Knowledge Catalog

**Decision:** Use **IBM Knowledge Catalog (IKC)** as the primary catalog for all asset types, with **watsonx.data + Hive Metastore** for structured/tabular data, federated into IKC.

**Context:** The estate contains raw genomics files, microscopy images, research documents, and structured tabular data. The catalog must support API-driven discovery, governance, and multi-tenancy.

### Options Evaluated

| Criterion | OpenMetadata | watsonx.data + Hive Metastore | IBM Knowledge Catalog |
|---|:---:|:---:|:---:|
| Catalogs raw files (fastq, bam, pdf, tif) | ✅ | ❌ | ✅ |
| API-driven metadata query | ✅ | ⚠️ SQL only | ✅ |
| Governance and masking policies | ✅ | ⚠️ | ✅✅ |
| Data lineage | ✅ | ⚠️ table-level | ✅✅ with Manta |
| Multi-tenancy | ✅ | ✅ | ✅ |
| IBM-native integration | ❌ | ✅ | ✅ |
| watsonx.ai enrichment connector | ❌ custom | ✅ | ✅ |
| DataStax / AstraDB integration | ❌ | ✅ | ✅ |
| OpenShift operator | ✅ Helm | ✅ IBM Operator | ✅ CP4D Operator |
| Licence cost | Free OSS | IBM commercial | IBM commercial |
| Structured data cataloguing | ✅ | ✅✅ | ✅ |
| Unstructured and file cataloguing | ✅ | ❌ | ✅ |
| Maturity | Medium | High | High |

### Decision Rationale

- **watsonx.data + Hive Metastore alone** is insufficient — Hive Metastore catalogues tables and partitions, not arbitrary files. It covers only ~30% of the estate (structured data).
- **OpenMetadata** is a strong fit functionally but has no IBM-native integration, requiring custom connectors to watsonx.ai, DataStage, and IKC.
- **IBM Knowledge Catalog** handles all asset types, is fully IBM-native, and provides the strongest governance capability including IBM Manta deep lineage.
- **Hybrid approach:** IKC as primary + watsonx.data for structured data with schema sync into IKC gives full coverage with no gaps.

---

## ADR-003: Semantic / Context Layer — DataStax (watsonx.data Premium)

**Decision:** Use **DataStax Langflow** (NL pipeline), **DataStax AstraDB** (vector store), and **DataStax Astra Streaming** (event bus) — all available as IBM-native components within watsonx.data Premium.

**Context:** Researchers need natural language query capability over the catalog. Content embeddings are needed for semantic similarity search.

### DataStax & IBM

DataStax was acquired by IBM in 2024 and integrated into **IBM watsonx.data Premium**. This makes the entire semantic layer IBM-native, eliminating external dependencies.

| DataStax Component | Role | Replaces |
|---|---|---|
| **Langflow** | Visual RAG pipeline builder — NL to structured query translation | Custom LangChain implementation |
| **AstraDB** | Managed vector store — content embeddings and similarity search | Standalone Milvus deployment |
| **Astra Streaming** | Apache Pulsar event backbone — real-time metadata events | Standalone Kafka deployment |
| **Cassandra** | High-throughput KV metadata store — billion-file scale lookups | Custom sharded PostgreSQL |

### Decision Rationale

Using DataStax within watsonx.data Premium:

- Reduces the number of independently managed OSS components (no Milvus, no Kafka)
- Keeps the stack IBM-native in alignment with the original requirement
- Langflow is a no-code visual tool — editable without AI assistance or engineering expertise
- AstraDB is purpose-built for the vector + metadata hybrid access pattern needed here

---

## ADR-004: Tape Recall Strategy — Immediate vs Deferred

**Decision:** Catalog tape stub metadata **immediately** on crawl. Trigger content extraction **only in scheduled batch windows** or **on-demand by user request** — never during the initial scan.

**Context:** IBM Tape (IBM Spectrum Archive / LTFS) files are represented as stub files on the Storage Scale filesystem. Reading content from tape requires an HSM recall which is expensive if done randomly.

### Options Evaluated

| Option | Approach | Risk |
|---|---|---|
| **Eager recall** | Recall all tape files during initial crawl | Extremely high tape seek overhead — could take months and damage media |
| **Deferred batch** | Catalog stub metadata immediately, recall in sequential nightly batches | ✅ Optimal tape utilisation, no blocking |
| **On-demand only** | Only recall when a user requests a file | Catalog remains metadata-only for all tape assets indefinitely |
| **Hybrid (selected)** | Immediate stub catalog + scheduled batch + on-demand | ✅ Best of all approaches |

### Decision Rationale

- **Sequential batch recall by cartridge** is orders of magnitude more efficient than random access
- Researchers get immediate catalog visibility of tape-resident data (path, size, dates) even before content is extracted
- Content enrichment (file type-specific metadata) fills in progressively as Airflow batch jobs process cartridges
- On-demand recall via the catalog UI gives urgent access when needed outside the batch window

---

## ADR-005: Vector Store — Milvus vs DataStax AstraDB

**Decision:** Use **DataStax AstraDB** (watsonx.data Premium) instead of standalone Milvus.

| Dimension | Milvus | DataStax AstraDB |
|---|---|---|
| IBM-native | ❌ OSS | ✅ IBM watsonx.data Premium |
| OpenShift deployment | StatefulSet — self-managed | StatefulSet — IBM operator managed |
| Integration with Langflow | Custom HTTP connector | ✅ Native built-in connector |
| Integration with watsonx.ai | Custom HTTP | ✅ Native |
| Operational overhead | High — separate cluster to manage | Lower — managed within watsonx.data |
| Vector index type | IVF, HNSW, multiple | HNSW |
| Scale | Proven at billions of vectors | Proven at billions of vectors |

### Decision Rationale

AstraDB provides equivalent vector search capability to Milvus while eliminating a standalone OSS cluster to manage, and integrating natively with Langflow and watsonx.ai within the IBM stack.

---

## ADR-006: Event Streaming — Kafka vs DataStax Astra Streaming

**Decision:** Use **DataStax Astra Streaming** (Apache Pulsar, IBM watsonx.data Premium) instead of standalone Kafka.

| Dimension | Apache Kafka | DataStax Astra Streaming |
|---|---|---|
| IBM-native | ❌ OSS | ✅ IBM watsonx.data Premium |
| Protocol | Kafka | Apache Pulsar |
| NiFi integration | `PublishKafka` processor | `PublishPulsar` processor |
| Multi-tenancy | Requires careful topic namespacing | Native multi-tenancy built-in |
| Geo-replication | Complex setup | Built-in |
| Operational overhead | High — ZooKeeper / KRaft cluster | Lower — managed within watsonx.data |

### Decision Rationale

Astra Streaming eliminates a standalone Kafka cluster while providing equivalent (and in some ways superior) streaming capability, natively integrated into the IBM stack.
