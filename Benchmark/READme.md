# DAYA vs PageIndex — Benchmarking Report

This folder contains benchmarking results comparing **[DAYA](https://github.com/RoneyBABA/DAYA)** against **[PageIndex](https://github.com/VectifyAI/PageIndex)**.

Both systems use hierarchical tree indexing and reject arbitrary chunking. The benchmarks are designed to surface where they diverge and shed the light on the quality of hierarchical trees created.

The benchmarking was evaluated **Vision-based Vectorless RAG notebook** provided in **[PageIndex Repository](https://github.com/VectifyAI/PageIndex/tree/main/cookbook)** 

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/VectifyAI/PageIndex/blob/main/cookbook/vision_RAG_pageindex.ipynb)
---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Comparison](#architecture-comparison)
3. [Pipeline Deep-Dive](#pipeline-deep-dive)
   - [DAYA Pipeline](#daya-pipeline)
   - [PageIndex Pipeline](#pageindex-pipeline)
4. [Key Architectural Differentiator: Image Annotation at Tree-Build Time](#key-architectural-differentiator-image-annotation-at-tree-build-time)
5. [Retrieval Logic: Smart Context Stitching](#retrieval-logic-smart-context-stitching)
6. [Reproducing the Benchmark](#reproducing-the-benchmark)
7. [Caveats and Limitations](#caveats-and-limitations)
8. [Acknowledgements](#acknowledgements)

---

## Overview

This document benchmarks **DAYA** against **[PageIndex](https://github.com/VectifyAI/PageIndex)** — one of the most principled open-source RAG alternatives available — across a range of document types, query styles, and evaluation criteria.

Both systems reject the traditional chunk-embed-search paradigm in favour of **hierarchical tree indexing**. The benchmarking is designed to surface where they diverge: visual document understanding, retrieval precision, inference cost, and contextual coherence.

| Property | DAYA | PageIndex |
|---|---|---|
| Core philosophy | Structure-preserved, vision-augmented RAG | Reasoning-based, vectorless RAG |
| Chunking | None | None |
| Visual content | ✅ Annotated at index time (text-converted) | ❌ Passed raw to VLM at query time |
| Retrieval mechanism | Embedding + distance/frequency filtering | LLM reasoning over tree at every query |
| LLM calls at retrieval | 0 | 1 per query |
| Final inference modality | **Text only** | Multimodal (image + text) |

---

## Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DAYA PIPELINE                                │
│                                                                     │
│  Document ──► Docling Layout Engine ──► VLM Figure Annotation       │
│                      │                        │                     │
│                       └──────────┬────────────┘                     │
│                                  ▼                                  │
│                         Tree Builder (JSON)                         │
│                    [figures → text descriptions]                    │
│                                  │                                  │
│                                  ▼                                  │
│                    Jina Embeddings → ChromaDB                       │
│                                  │                                  │
│                    ┌─────────────┘                                  │
│                    ▼                                                │
│           Query Decomposition (LLM)                                 │
│                    │                                                │
│                    ▼                                                │
│         Distance + Frequency Filtered Retrieval                     │
│                    │                                                │
│                    ▼                                                │
│         Smart Tree-Logical Context Stitching                        │
│                    │                                                │
│                    ▼                                                │
│              LLM Inference (TEXT ONLY) ──► Cited Answer             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     PAGEINDEX PIPELINE                              │
│                                                                     │
│  Document ──► PageIndex API ──► Hierarchical Tree (text + summary)  │
│                                           │                         │
│              PDF → Page Images (local)    │                         │
│                    │                      │                         │
│                    └──────────┬───────────┘                         │
│                               ▼                                     │
│              LLM Reasons over Tree Structure (per query)            │
│                               │                                     │
│                               ▼                                     │
│              Page Images of Retrieved Nodes                         │
│                               │                                     │
│                               ▼                                     │
│              VLM Inference (IMAGE + TEXT) ──► Answer                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Deep-Dive

### DAYA Pipeline

**Stage 1 — Layout Extraction (Docling + VLM)**
Docling extracts section text, Markdown tables, and figures. Every `PictureItem` is passed once to Llama 4 Scout (Groq), which produces a structured text description (chart type, axes, data series, visible text). This is the only VLM call in the pipeline — it happens at index time, never again.

**Stage 2 — Tree Building**
Content is assembled into a hierarchical JSON tree following the document's actual section structure. Each node carries `node_id`, `page_index`, `root_page`, `curr_index`, `next_index`, and full section text. Figure descriptions from Stage 1 are written in as text nodes — every image in the document is converted to text here. Nothing visual is deferred to query time.

**Stage 3 — Embedding and Storage**
Each node is embedded with `jina-embeddings-v5-text-small` and upserted into ChromaDB with `node_id`, `curr_index`, `next_index`, `root_index`, and `filename` as metadata.

**Stage 4 — Retrieval and Inference**
Queries are decomposed into sub-questions, each embedded and matched against ChromaDB via distance thresholding (≤ 0.85) and frequency-based filtering. The final LLM call receives text only — context-stitched node content with a citation-enforcing system prompt.

---

### PageIndex Pipeline

**Stage 1 — Tree Generation**
The PageIndex API builds a hierarchical tree with node titles, summaries, and page ranges. Images are not annotated; the tree is text metadata only.

**Stage 2 — Visual Asset Preparation**
PDF pages are rasterized locally via PyMuPDF (2× zoom → JPEG), keyed by page number. These are not part of the index.

**Stage 3 — Retrieval (LLM-as-Reasoner)**
At every query, the tree (titles + summaries) is serialised into a prompt. The LLM returns a reasoning chain and a list of relevant `node_id`s — one LLM call per retrieval.

**Stage 4 — Multimodal Inference**
Retrieved node page ranges are mapped to JPEGs, base64-encoded, and sent to a VLM alongside the query. The model reads raw page images and produces the final answer.

---


## **Key Architectural Differentiator: Image Annotation at Tree-Build Time**

This is the most significant design divergence between the two systems.

### PageIndex approach — visual content deferred
PageIndex builds its tree from text only. Visual content (figures, diagrams, charts) is handled by passing raw page images to a VLM **at query time**. The tree has no awareness of visual content during indexing.

Consequence: the tree cannot reason about whether a node's page contains a relevant figure. The VLM at query time must infer both the relevance and the content of images simultaneously.

### DAYA approach — visual content collapsed to text at index time
DAYA annotates every figure at **Stage 1** (tree build time) using a VLM. The annotation produces a structured textual description:

- Chart type and title
- Axis labels and scale
- Data series names and approximate values
- UI element states (for screenshots)
- All visible text verbatim

This description is written into the tree node as text. By the time the tree is embedded and stored, **there are no images in the index** — only rich, queryable text representations of what those images contained.

**Downstream consequence — text-only inference:**

Because all visual content has been converted to text during indexing, the final retrieval and answer generation stage operates entirely on text. This means:

- Standard embedding models (not multimodal) can match visual content
- LLM inference at query time is text-only: lower latency, lower cost, no VLM dependency per query
- The answer generator does not need vision capability
- Retrieved context is fully inspectable and debuggable as plain text

The trade-off is a richer, more expensive index-time pass — paid once per document — versus a cheaper but vision-dependent query-time pass paid on every question.

---

## Reproducing the Benchmark

Run DAYA at **[Google colab](https://colab.research.google.com/drive/1mMr9ORZ9daOPTSF5DiBHLFLD1p8npDjY?usp=sharing)** to retrieve DAYA_tree.

Run the following code in **Google colab** to obtain a PageIndex_tree.

Get your free `PAGEINDEX_API_KEY` | [PageIndex.ai](https://dash.pageindex.ai/api-keys) |
```bash

   #1 : Intialize
   !pip install pageindex
   import os
   from google.colab import files
   from google.colab import userdata
   from pageindex import PageIndexClient
   import pageindex.utils as utils

   PAGEINDEX_API_KEY = userdata.get('PAGEINDEX')
   pi_client = PageIndexClient(api_key=PAGEINDEX_API_KEY)


   #2 : Call PageIndex API
   import requests

   api_key = PAGEINDEX_API_KEY  # or hardcode temporarily
   uploaded = files.upload()

   if len(uploaded) > 1:
      raise ValueError(f"Please upload only one file. You uploaded {len(uploaded)} files.")

   file_path = list(uploaded.keys())[0]
   print(f"File loaded: {file_path}")

   with open(file_path, "rb") as file:
      response = requests.post(
         "https://api.pageindex.ai/doc/",
         headers={"api_key": api_key},
         files={"file": file},
         timeout=60
      )

   print(response.status_code)
   print(response.json())

  #The doc ID will be the output of the following function. Use it in the next step to obtain it's tree.

 #3: Obtain the json tree
   doc_id = #"pi-cmpwwj0uj00"
   tree = pi_client.get_tree(doc_id, node_summary=False)['result'] #Can set node_summary = True to summarize the content of each node
   print('Simplified Tree Structure of the Document:')
   utils.print_tree(tree)

   # Save tree to a JSON file, then pass the path to make_pdf
   res_path = "tree_output.json"
   with open(res_path, "w") as f:
      json.dump(tree, f, indent=4)
   files.download("tree_output.json")
```

---

## Caveats and Limitations

**PageIndex tree quality is API-opaque.** The quality of the PageIndex tree depends on the closed API's internal LLM and retry logic. Variability in tree structure across runs may affect retrieval consistency.

**VLM annotation quality is prompt-sensitive.** DAYA's figure annotations are only as good as the Llama 4 Scout extraction prompt. Highly complex figures (3D plots, dense schematics) may be partially described.

**DAYA index time is higher.** VLM annotation at index time adds latency that PageIndex avoids. For very large document corpora, this is a meaningful operational cost.

**PageIndex is vectorless by design.** Comparing retrieval recall between a vector-search system (DAYA) and a reasoning-based system (PageIndex) is not fully apples-to-apples. PageIndex's "retrieval" is closer to reading comprehension over a summary; DAYA's is nearest-neighbour search over embeddings. The benchmarking metrics reflect *outcome quality* rather than mechanism equivalence.

---

## Acknowledgements

DAYA benchmarks against **[PageIndex by VectifyAI](https://github.com/VectifyAI/PageIndex)** — a genuinely innovative project that rethought RAG retrieval from first principles. Both systems are built on the premise that chunking is the wrong abstraction for structured documents.

DAYA's layout extraction is powered by **[Docling by IBM Research Zurich](https://github.com/DS4SD/docling)**, licensed under MIT.

---

## License

DAYA is released under the **Apache License 2.0**. See [LICENSE](./LICENSE) for details.  
PageIndex is released under the **MIT License**.

---

<div align="center">

# DAYA
### Document Aware hYbrid Architecture

**The RAG pipeline built for precision, by eliminating *chunking strategies* for illustrated documents.**

</div>
