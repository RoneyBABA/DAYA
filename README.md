<div align="center">

# DAYA
### Document Aware hYbrid Architecture

**The RAG pipeline that sees *everything* — text, tables, charts, and visuals — without losing the plot.**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/python-3.10%2B-brightgreen)](https://www.python.org/)
[![Built on Docling](https://img.shields.io/badge/built%20on-Docling-orange)](https://github.com/docling-project/docling)
[![Inspired by PageIndex](https://img.shields.io/badge/inspired%20by-PageIndex-purple)](https://github.com/VectifyAI/PageIndex)

</div>

---

## Why DAYA Exists

Every document understanding framework today has a fatal flaw. Pick one, and you're making a tradeoff you didn't sign up for.

**Docling** is exceptional at parsing complex documents — it captures tables, charts, images, and rich layouts with impressive fidelity. But the moment it comes to retrieval, it falls back on chunking. Chunks don't know what page they came from. Chunks don't know their neighbors. Chunks don't *think*.

**PageIndex** flips the script beautifully — no chunking, hierarchical tree indexing, reasoning-based retrieval. Elegant and effective for text-heavy documents. But throw a heavily illustrated PowerPoint or a figure-dense research paper at it, and the visual context simply disappears.

**DAYA bridges this gap.** It takes Docling's rich layout and illustration capture, wires it to a PageIndex-inspired hierarchical tree structure, and delivers retrieval that is both visually complete and structurally aware. Every chart gets described. Every slide gets indexed. Every query gets answered with page-level precision.

---

## How It Works

DAYA is a four-stage pipeline:

```
Document (PDF / PPTX / DOCX)
        │
        ▼
 ┌──────────────────┐
 │  Layout Engine   │  ← Docling parses structure; VLM (Llama 4 Scout) describes figures
 └────────┬─────────┘
          │
          ▼
 ┌──────────────────┐
 │   Tree Builder   │  ← Page-aware hierarchical index (no arbitrary chunking)
 └────────┬─────────┘
          │
          ▼
 ┌──────────────────┐
 │ Embed + Store    │  ← Jina AI embeddings → ChromaDB
 └────────┬─────────┘
          │
          ▼
 ┌──────────────────┐
 │    Retrieval     │  ← Distance-filtered, frequency-weighted node selection
 └────────┬─────────┘
          │
          ▼
    LLM Inference (Groq)
```

### Stage 1 — Layout Extraction (Docling + VLM)

Docling processes the document and preserves its full structure: section headers, body text, tables (exported as Markdown), and crucially, **figures**. Every `PictureItem` is passed to a vision-language model (Llama 4 Scout via Groq) with a structured extraction prompt that identifies chart types, reads axis labels, enumerates data series, extracts UI element states, and captures all visible text verbatim. The result is a rich textual description attached to each figure.

### Stage 2 — Tree Building (PageIndex-inspired, no chunking)

The extracted content is organized into a hierarchical JSON tree — not by splitting text at token boundaries, but by following the document's actual structure. Section headers drive the hierarchy. Figures are inserted as first-class nodes alongside the text they accompany. Every node carries a page index, document order, and the accumulated text for its section. The result looks like a "table of contents that knows what's on every page."

### Stage 3 — Embedding and Storage (Jina AI + ChromaDB)

Each tree node is embedded using Jina AI's `jina-embeddings-v5-text-small` (up to 8192 tokens, text-matching task). The embeddings are stored in ChromaDB with metadata that preserves the node ID, current page, and next page — enabling range-aware retrieval later.

### Stage 4 — Retrieval and Inference (Groq LLM)

Queries are first decomposed into atomic sub-questions (pronoun resolution included) by the LLM. Each sub-question is embedded and matched against ChromaDB using distance thresholding (≤ 0.85) and frequency-based filtering to identify the most relevant page ranges. The corresponding tree nodes are extracted, and the LLM answers strictly from the retrieved text — with page-level citations enforced by the system prompt.

---

## What DAYA Handles That Others Don't

| Capability | Docling (alone) | PageIndex (alone) | DAYA |
|---|:---:|:---:|:---:|
| Rich PDF layout (tables, formulas) | ✅ | ❌ | ✅ |
| PPTX / heavily illustrated docs | ✅ | ❌ | ✅ |
| Figure / chart understanding | ✅ (VLM) | ❌ | ✅ (VLM) |
| No arbitrary chunking | ❌ | ✅ | ✅ |
| Hierarchical tree index | ❌ | ✅ | ✅ |
| Page-level retrieval | ❌ | ✅ | ✅ |
| Multi-question decomposition | ❌ | ❌ | ✅ |

---

## Quickstart

Open `DAYA.ipynb` and run all cells. When prompted, upload your document (PDF, PPTX, or DOCX). The pipeline will:

1. Convert and render your document
2. Extract layout, text, tables, and figure descriptions
3. Build the hierarchical tree index
4. Embed and store all nodes
5. Enter an interactive Q&A loop — ask anything, get cited answers

---

## Key Components

**`layout_ext(filename)`** — Runs Docling with EasyOCR and VLM figure description on the input document. Outputs a JSON tree file and a display-ready PDF.

**`build_heading_table(document, ...)`** — Traverses the Docling document model to extract section headings and figure annotations, building a flat list that respects document order and slide-to-page mapping for PPTX inputs.

**`build_ideal_output(tree_headings, section_texts, ...)`** — Assembles the final hierarchical tree by enriching each heading node with its corresponding page text and figure VLM descriptions.

**`embed(filename, collection)`** — Chunks large node texts conservatively (512-token windows with 16-token overlap, only when necessary), generates Jina embeddings, and upserts into ChromaDB with full metadata.

**`get_nodes(result)`** — Applies distance filtering and frequency-based deduplication to ChromaDB results, returning the most relevant node IDs and page ranges.

**`ask_question()`** — The top-level entry point. Decomposes the user query, retrieves relevant nodes, and calls the LLM with a strict citation-enforcing system prompt.

---

## Tech Stack (Free tier)

| Component | Technology |
|---|---|
| Document parsing | [Docling](https://github.com/docling-project/docling) |
| Figure understanding | Llama 4 Scout 17B (via Groq) |
| PPTX rendering | LibreOffice + PyMuPDF |
| Tree indexing | Custom (PageIndex-inspired) |
| Embeddings | Jina AI `jina-embeddings-v5-text-small` |
| Vector store | ChromaDB |
| LLM inference | Groq (`openai/gpt-oss-120b`) |

---

## 🤝 Contributing

Contributions are welcome! Here's how to get started:

```bash
# 1. Fork the repository
# 2. Create your feature branch
git checkout -b feature/your-feature-name

# 3. Commit your changes
git commit -m "feat: add your feature description"

# 4. Push to your branch
git push origin feature/your-feature-name

# 5. Open a Pull Request
```

Please follow [Conventional Commits](https://www.conventionalcommits.org/) for commit messages.

---

## Acknowledgements

DAYA stands on the shoulders of two outstanding open-source projects:

**[Docling](https://github.com/docling-project/docling)** by IBM Research Zurich — for its best-in-class document parsing, layout understanding, and rich multi-format support. Licensed under MIT.

**[PageIndex](https://github.com/VectifyAI/PageIndex)** by VectifyAI — for the insight that retrieval should be structured and reasoning-based, not similarity-based and chunked. Licensed under MIT.

Both projects are used in accordance with their respective MIT licenses.

---

## License

DAYA is released under the Apache License 2.0.

---
