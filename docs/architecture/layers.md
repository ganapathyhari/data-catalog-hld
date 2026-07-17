# Layered Architecture

## Platform Layers — Storage to Natural Language

The platform is built as **five horizontal layers**, each running as containerised workloads on Red Hat OpenShift. Data and metadata flow upward from physical storage at the base through to natural language conversation at the top.

Use the **tabs at the bottom of the diagram** to navigate each layer in detail.

| Tab | Layer | Contents |
|---|---|---|
| **1. Full Layered Stack** | All layers | Complete top-to-bottom view — Storage → NL Interface |
| **2. Layer 0 — Physical Storage** | Foundation | IBM Storage Scale, IBM Tape, HSM, file type distribution |
| **3. Layer 1 — Ingestion** | Data Plane | NiFi pipeline, tape stub detection, all format extractors, Airflow DAGs |
| **4. Layer 2 — Metadata Catalog** | Catalog | IKC, watsonx.data, custom asset types, PostgreSQL, Elasticsearch |
| **5. Layer 3 — Semantic Context** | Intelligence | Langflow RAG, AstraDB embeddings, Astra Streaming events, watsonx.ai |
| **6. Layer 4 — UI and APIs** | Access | NL interface, IKC UI, REST/GraphQL APIs, data access routing |

---

## Layer Summary

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 4 — Natural Language, IKC UI, REST/GraphQL APIs          │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 3 — Semantic Context (DataStax Langflow + AstraDB)       │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 2 — Metadata Catalog (IBM Knowledge Catalog)             │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 1 — Ingestion and Extraction (NiFi + DataStage)          │
├─────────────────────────────────────────────────────────────────┤
│  LAYER 0 — Physical Storage (IBM Storage Scale + IBM Tape)      │
└─────────────────────────────────────────────────────────────────┘
           All layers run on Red Hat OpenShift
```

---

## Interactive Diagram

<div style="width:100%;border:1px solid #e5e7eb;border-radius:4px;overflow:hidden;">
<iframe src="https://viewer.diagrams.net/?url=https://raw.githubusercontent.com/ganapathyhari/data-catalog-hld/main/docs/architecture/architecture-layers.drawio&highlight=0000ff&edit=_blank&layers=1&nav=1&title=Architecture%20Layers"
  width="100%"
  height="820px"
  style="border:none;">
</iframe>
</div>

!!! tip "Edit this diagram"
    Go to [app.diagrams.net](https://app.diagrams.net) → **File → Open from → GitHub** → navigate to `docs/architecture/architecture-layers.drawio`. Save back to GitHub and this page updates automatically.

!!! note "Open directly in Draw.io"
    [Click here to open the diagram in Draw.io](https://app.diagrams.net/#Uhttps://raw.githubusercontent.com/ganapathyhari/data-catalog-hld/main/docs/architecture/architecture-layers.drawio)
