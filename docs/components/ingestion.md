# Ingestion Pipeline

## Apache NiFi + Astronomer Airflow

The ingestion layer uses two complementary tools that operate at different levels — NiFi as the **data plane** and Airflow as the **control plane**.

---

## Why Two Tools?

```mermaid
flowchart LR
    subgraph NIFI_ROLE["Apache NiFi — Data Plane"]
        N1["Moves actual file bytes\nand metadata"]
        N2["Event-driven reaction\nto new files"]
        N3["Built-in backpressure\nand flow control"]
        N4["Per-file provenance\nchain tracking"]
        N5["Handles binary formats\nBAM, HDF5, DICOM natively"]
    end

    subgraph AIRFLOW_ROLE["Astronomer Airflow — Control Plane"]
        A1["Schedules WHEN\nNiFi runs full re-scans"]
        A2["Orchestrates watsonx.ai\nbatch enrichment jobs"]
        A3["Manages tape recall\nbatch windows"]
        A4["Coordinates downstream\npipeline dependencies"]
        A5["Monitors pipeline\nhealth and SLAs"]
    end

    AIRFLOW_ROLE -->|Triggers| NIFI_ROLE
    NIFI_ROLE -->|Events/metrics| AIRFLOW_ROLE
```

!!! success "Key Principle"
    NiFi and Airflow are **complementary, not competing**. NiFi handles individual file-level parallelism at scale — a concept Airflow has no equivalent for. Airflow handles DAG-level workflow coordination — a concept NiFi is not designed for.

---

## NiFi vs Airflow Detailed Comparison

| Dimension | Apache NiFi | Astronomer Airflow |
|---|---|---|
| **Primary purpose** | Real-time data flow, routing, transformation | Workflow orchestration — scheduling DAGs |
| **Data handling** | Moves actual file bytes and metadata natively | Orchestrates tasks that move data — does not move data itself |
| **Filesystem crawling** | Native `ListFile`, `FetchFile`, `GetSFTP` processors | No native filesystem crawler — requires custom operators |
| **Backpressure and flow control** | Built-in — queues, prioritisation, rate limiting | Not applicable — Airflow does not buffer data |
| **Content extraction** | Native `ExtractText`, `IdentifyMimeType`, Apache Tika | Requires custom Python operators per file type |
| **Event-driven triggers** | Yes — reacts to new files, GPFS callbacks | Primarily schedule-based (cron); event-driven is complex |
| **20PB scale** | Cluster mode — linear horizontal scale, millions of files/day | Viable but requires significant custom operator engineering |
| **Low-latency incremental** | Sub-minute on new file arrival | Minimum 30s scheduler loop; not truly event-driven |
| **Tape stub detection** | Reads filesystem xattr natively — no recall triggered | Cannot distinguish stub vs hot file without custom operator |
| **Built-in data provenance** | Yes — per FlowFile provenance chain | No — lineage must be captured externally |
| **OpenShift deployment** | StatefulSet + ZooKeeper — fully on-prem | Astronomer Software — fully on-prem operator available |
| **IBM ecosystem fit** | OSS — integrates via HTTP/REST | OSS — integrates via HTTP/REST |

---

## NiFi Pipeline Architecture

```mermaid
flowchart TD
    subgraph CRAWL["File Discovery"]
        LS["ListFile\nGPFS filesystem walk\nor GPFS callback receiver"]
        FF["FetchFile\nRead content bytes\nonly if not tape stub"]
    end

    subgraph STUB["Tape Stub Detection"]
        DETECT["RouteOnAttribute\nCheck xattr hsm.state"]
        META_ONLY["Filesystem metadata\nextract only\nno content read"]
        RECALL["InvokeHTTP\nIBM Spectrum Archive\nrecall API"]
    end

    subgraph ROUTE["Content Routing"]
        MIME["IdentifyMimeType\nDetect file type"]
        ROUTE_P["RouteOnAttribute\nby mime.type and extension"]
    end

    subgraph EXTRACT["Format-Specific Extraction"]
        TIKA["ExtractText via Apache Tika\nPDF, DOCX, PPTX, HTML, TXT"]
        SAMTOOLS["ExecuteStreamCommand\nSAMtools header parse\nBAM, SAM, CRAM, VCF"]
        HDF5["ExecuteStreamCommand\nh5dump / h5py script\nHDF5, Zarr, NetCDF"]
        PYDICOM["ExecuteStreamCommand\nPyDICOM / libCZI\nDICOM, CZI, OME-TIFF"]
        SCHEMA["ConvertRecord\nCSV, Parquet, Arrow\nschema inference"]
        JSON_P["EvaluateJsonPath\nJSON, JSONL, IPYNB, YAML"]
        XML_P["EvaluateXPath\nXML, GFF, GTF, SVG"]
        UNPACK["UnpackContent\nTAR, ZIP, GZ, BGZF\nre-routes members back into pipeline"]
    end

    subgraph ENRICH["AI Enrichment"]
        WX["InvokeHTTP\nwatsonx.ai REST API\nClassification + PII + Quality score"]
    end

    subgraph EMIT["Output"]
        PG_OUT["PutDatabaseRecord\nPostgreSQL\nraw metadata"]
        IKC_OUT["InvokeHTTP\nIBM Knowledge Catalog\nREST API asset creation"]
        ASTRA["InvokeHTTP\nDataStax AstraDB\nvector embeddings"]
        STREAM["PublishPulsar\nAstra Streaming\nreal-time events"]
    end

    LS --> DETECT
    DETECT -->|hsm.state = MIGRATED| META_ONLY
    DETECT -->|hsm.state = RESIDENT| FF
    META_ONLY --> IKC_OUT
    META_ONLY -->|scheduled batch| RECALL
    FF --> MIME
    MIME --> ROUTE_P
    ROUTE_P --> TIKA & SAMTOOLS & HDF5 & PYDICOM & SCHEMA & JSON_P & XML_P & UNPACK
    UNPACK -->|re-enter pipeline| MIME
    TIKA & SAMTOOLS & HDF5 & PYDICOM & SCHEMA & JSON_P & XML_P --> WX
    WX --> PG_OUT & IKC_OUT & ASTRA & STREAM
```

---

## Airflow DAG Design

```mermaid
flowchart LR
    subgraph DAGS["Airflow DAGs on Astronomer"]
        DAG1["dag: full-crawl-scheduled\n• Weekly full Storage Scale scan\n• Triggers NiFi ListFile via API\n• Monitors completion"]

        DAG2["dag: tape-recall-batch\n• Nightly off-peak window\n• Submits HSM recall jobs\n• Queues content extraction"]

        DAG3["dag: watsonx-enrichment-batch\n• Daily batch enrichment\n• Runs watsonx.ai on\n  newly catalogued assets"]

        DAG4["dag: catalog-quality-check\n• Weekly data quality\n• Flags incomplete assets\n• Triggers re-extraction"]

        DAG5["dag: ikc-sync-verify\n• Daily sync check\n• Verifies NiFi output\n  matches IKC record count"]
    end
```

---

## NiFi Cluster Sizing

| Parameter | Value | Rationale |
|---|---|---|
| NiFi nodes | 10 | Parallel processing across 20PB |
| ZooKeeper nodes | 3 | Cluster state quorum |
| FlowFile repo per node | 500 GB SSD | Fast local FlowFile queuing |
| Content repo per node | 2 TB | Temporary content buffer |
| Target crawl throughput | ~700 TB/day | 20PB in 30 days |
| Max concurrent tasks per node | 20 | Tunable per processor |

!!! tip "Post-Crawl Scale Down"
    After the initial 20PB crawl is complete, the NiFi cluster can be scaled down to 3 nodes for steady-state event-driven incremental updates, reducing resource consumption significantly.

---

## Astronomer Airflow Deployment Note

!!! warning "Astronomer Software — Not Astronomer Cloud"
    This architecture requires **Astronomer Software** (self-hosted on OpenShift via the Astronomer operator) — **not** Astronomer Cloud (SaaS).

    Astronomer Cloud would introduce an external SaaS dependency and potential data residency concerns for genomics and research data. Astronomer Software runs entirely within the OpenShift cluster with no outbound data transmission.
