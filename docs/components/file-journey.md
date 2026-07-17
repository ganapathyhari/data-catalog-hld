# Day in the Life of a File

## File Journey Through the Data Catalog Stack

This page contains an interactive Draw.io diagram showing how each file type travels through the full platform stack — from IBM Storage Scale or IBM Tape through to being searchable in IBM Knowledge Catalog.

Open the diagram below and use the **tabs at the bottom** to navigate between:

| Tab | Content |
|---|---|
| **1. Full Stack Overview** | All 10 steps shared across every file type |
| **2. PDF Journey** | Unstructured document path |
| **3. JSON Journey** | Semi-structured data path |
| **4. SQLite Journey** | Structured database path |
| **5. Image Journey** | Binary imaging path (.dcm / .tif / .czi) |
| **6. Genomics Journey** | Custom domain formats (.bam / .fastq / .vcf) |
| **7. Divergence Comparison** | Side-by-side table of where journeys differ |

---

## The 10 Steps (Shared by All File Types)

| Step | Action | Component | Same for All? |
|---|---|---|---|
| 1 | File detected | NiFi ListFile / GPFS callback | ✅ Yes |
| 2 | Tape stub check | xattr `hsm.state` | ✅ Yes |
| 3 | Filesystem metadata captured | NiFi FetchFile | ✅ Yes |
| 4 | MIME type identified | NiFi IdentifyMimeType | ✅ Yes |
| **5** | **Content extractor chosen** | **Tika / JsonPath / SQLite / PyDICOM / SAMtools** | **❌ DIVERGES** |
| **6** | **Domain metadata extracted** | **Format-specific fields** | **❌ DIVERGES** |
| 7 | AI enrichment | IBM watsonx.ai | ✅ Yes — PII risk level varies |
| 8 | IKC asset record created | IBM Knowledge Catalog REST API | ✅ Yes — asset type varies |
| 9 | Embedding stored | DataStax AstraDB | ✅ Yes — embedding source varies |
| 10 | Searchable in catalog | IKC UI + NL interface | ✅ Yes |

---

## Interactive Diagram

<div style="width:100%;border:1px solid #e5e7eb;border-radius:4px;overflow:hidden;">
<iframe src="https://viewer.diagrams.net/?url=https://raw.githubusercontent.com/ganapathyhari/data-catalog-hld/main/docs/components/file-journey.drawio&highlight=0000ff&edit=_blank&layers=1&nav=1&title=File%20Journey"
  width="100%"
  height="820px"
  style="border:none;">
</iframe>
</div>

!!! tip "Edit this diagram"
    Go to [app.diagrams.net](https://app.diagrams.net) → **File → Open from → GitHub** → navigate to `docs/components/file-journey.drawio`. Save back to GitHub and this page updates automatically — no export needed.

!!! note "Open directly in Draw.io"
    [Click here to open the diagram in Draw.io](https://app.diagrams.net/#Uhttps://raw.githubusercontent.com/ganapathyhari/data-catalog-hld/main/docs/components/file-journey.drawio)
