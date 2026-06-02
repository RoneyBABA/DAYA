<div align="center">

# DAYA
### Document Aware hYbrid Architecture

**The RAG pipeline built for precision, by eliminating *chunking strategies* for illustrated documents.**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/python-3.10%2B-brightgreen)](https://www.python.org/)
[![Built on Docling](https://img.shields.io/badge/built%20on-Docling-orange)](https://github.com/docling-project/docling)
[![Inspired by PageIndex](https://img.shields.io/badge/inspired%20by-PageIndex-purple)](https://github.com/VectifyAI/PageIndex)

</div>

---
## The Problem with Traditional RAG

Vector-based RAG is the industry default — and it is fundamentally broken for complex, real-world documents.

The issues are structural, not superficial:

### **1. Chunking destroys semantic integrity.** 

Documents are split at arbitrary token boundaries — cutting through procedures, checklists, tables, and cross-references. Each chunk is processed in isolation, therefore looses **context**. 

### **2. Similarity is not relevance.** 

Vector search matches text that **looks** like the query, not text that **answers** it. In technical and professional documents, the truly relevant section is frequently not retrieved.

### **3. Redundant retrievals cause context confusion.** 

When multiple similar chunks are returned together, the LLM is forced to reconcile contradictory or overlapping content. This **enables LLMs to hallucinate**.

### **4. Visual content is invisible.** 

Charts, figures, diagrams, and illustrated slides are discarded at the preprocessing stage. No embedding model reads a bar chart. No chunk captures what a schematic actually shows.

**DAYA was built to solve all four of these failures — not patch around them.**

---

## PageIndex-inspired

PageIndex is a genuine step forward: hierarchical tree indexing, no chunking, LLM-driven structure extraction with self-correction across up to three retry rounds.

But PageIndex is **visually blind**. Feed it a slide deck or a figure-dense manual, and the visual content simply disappears. At query time, every retrieval step burns an LLM call to navigate the tree.

DAYA takes PageIndex's reasoning paradigm and goes further by combining it with Docling's industry-leading layout extraction and a vision-language model — so nothing in your document is ever invisible.No LLM at retrieval time means lower latency and zero cost per query.

---

## How DAYA Solves Each Failure

### No Chunking — Structure-Preserved Indexing

### Reasoning over the Tree, Not Similarity over Chunks

### High Precision Through Structural Targeting

### Full Visual Understanding via Docling + VLM

| Capability | Vector RAG | PageIndex | DAYA |
|---|:---:|:---:|:---:|
| No arbitrary chunking | ❌ | ✅ | ✅ |
| Hierarchical tree index | ❌ | ✅ | ✅ |
| Reasoning-based retrieval | ❌ | ✅ | ✅ |
| Page-level citations | ❌ | ✅ | ✅ |
| LLM-free retrieval | ✅ | ❌ | ✅ |
| Text-only final inference | ✅ | ❌ | ✅ |
| Multi-question decomposition | ❌ | ❌ | ✅ |
| Zero LLM calls at query time | ✅ | ❌ | ✅ |

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
| Figure understanding | Llama 4 Scout 17B (via [Groq](https://groq.com)) |
| PPTX rendering | LibreOffice |
| Tree indexing | Custom ([PageIndex-inspired](https://github.com/VectifyAI/PageIndex)) |
| Embeddings | [Jina AI](https://github.com/jina-ai) `jina-embeddings-v5-text-small` |
| Vector store | [ChromaDB](https://www.trychroma.com/) |
| LLM inference | Groq (`openai/gpt-oss-120b`) (via [Groq](https://groq.com)) |

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

**[Docling](https://github.com/docling-project/docling)** by IBM Research Zurich. Licensed under MIT.

**[PageIndex](https://github.com/VectifyAI/PageIndex)** by VectifyAI. Licensed under MIT.

Both projects are used in accordance with their respective MIT licenses.

---

## License

DAYA is released under the Apache License 2.0. See the [LICENSE](LICENSE) file for details.

---
