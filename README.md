# 20PB Data Catalog HLD — MkDocs Site

This repository contains the High Level Design documentation for the **20PB Genomics & Research Data Cataloging Platform on OpenShift**, published as a GitHub Pages site using MkDocs with the Material theme.

## Live Site

The site is automatically published to GitHub Pages on every push to `main`.

> `https://<your-org>.github.io/data-catalog-hld/`

## Editing the Documentation

All content lives in the `docs/` folder as plain markdown (`.md`) files. You can edit them:

- **In your browser** — navigate to any file on GitHub, click the pencil icon, edit, and commit. The site rebuilds automatically within ~60 seconds.
- **Locally** — clone the repo, edit any `.md` file in `docs/`, push to `main`.

No build tools, no local install required for editing.

## Local Preview (Optional)

If you want to preview changes locally before pushing:

```bash
pip install mkdocs-material pymdown-extensions
mkdocs serve
```

Then open `http://127.0.0.1:8000` in your browser.

## Structure

```
data-catalog-hld/
├── docs/
│   ├── index.md                          # Home — Executive Summary
│   ├── architecture/
│   │   ├── overview.md                   # Full system architecture
│   │   ├── storage-integration.md        # IBM Storage Scale + Tape
│   │   └── deployment.md                 # OpenShift deployment
│   ├── components/
│   │   ├── ingestion.md                  # NiFi + Airflow
│   │   ├── catalog.md                    # IKC + watsonx.data trade-offs
│   │   ├── semantic-layer.md             # DataStax Langflow / AstraDB
│   │   └── file-formats.md               # Full format inventory
│   └── decisions/
│       ├── trade-offs.md                 # Architectural Decision Records
│       └── roadmap.md                    # Phased delivery roadmap
├── mkdocs.yml                            # Site configuration
├── .github/workflows/deploy.yml          # Auto-deploy on push to main
└── README.md
```

## Mermaid Diagrams

All architecture diagrams in this site are written as **Mermaid** inside the markdown files — they are plain text. To update a diagram, edit the text between the ` ```mermaid ` code fences directly in the `.md` file.

No drawing tools required.
