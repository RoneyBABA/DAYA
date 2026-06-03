# DAYA — Setup Guide

DAYA comes in two flavours. Pick the one that fits your environment.

```
daya/
├── colab/          # Google Colab notebook, zero local setup
└── offline/        # Local Python environment, IDE-friendly modules
```

---

## 🚀 Quick Trial

Get started instantly without any local configuration:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1mMr9ORZ9daOPTSF5DiBHLFLD1p8npDjY?usp=sharing)

> **Note:** You can also download this code and run it locally in **VS Code**. Simply install the **Jupyter extension**, open the file, and use the Colab/Local kernel to execute your code.

---

## Colab (Quickstart)

No installation needed. Open the notebook in Google Colab, add your API keys as Colab Secrets, and run all cells.

Required secrets (left sidebar → 🔑 Secrets):

| Secret Name | Where to get it |
|---|---|
| `VLM_API_KEY` | [console.groq.com](https://console.groq.com) |
| `LLM_API_KEY` | [console.groq.com](https://console.groq.com) |
| `EMBEDDING_API_KEY` | [jina.ai](https://jina.ai) — prefix with `Bearer ` |

---

## Swapping Components

Every external service in DAYA is a free tier and swappable. Nothing is hardcoded to a vendor.

| Component | Default | Drop-in alternatives |
|---|---|---|
| LLM inference | Groq (free tier) | OpenAI, Ollama, Anthropic, Gemini |
| VLM (figure description) | Llama 4 Scout via Groq | GPT-4o, Gemini Flash, LLaVA (local) |
| Embeddings | Jina AI (free tier) | OpenAI, Cohere, `sentence-transformers` (local) |
| Vector store | ChromaDB (in-memory) | Qdrant, Weaviate, FAISS, Milvus |
| Document parsing | Docling | — (core dependency, not swappable) |

---

## Functions to be Swapped

Since all the services used here are of free tier, swap it out for paid keys. If the code is hitting rate limits Search for "time.sleep" and un comment all the lines of code

### Swapping Groq (LLM)

Groq is used in two functions: `llm_call()` and `ask_question()`. Both follow the same pattern — find these lines and replace them:

```python
# llm_call() and ask_question() — replace these three lines:
client = Groq(api_key=LLM_API_KEY)
client.chat.completions.create(model="openai/gpt-oss-120b", ...)

# OpenAI example:
from openai import OpenAI
client = OpenAI(api_key=LLM_API_KEY)
client.chat.completions.create(model="gpt-4o", ...)

# Ollama (local, no key needed):
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
client.chat.completions.create(model="llama3.2", ...)
```

---

### Swapping Groq (VLM — figure descriptions)

The VLM client is initialised at the top of the layout extraction section. Change these two lines:

```python
# Original — find and replace:
vlm_client = Groq(api_key=VLM_API_KEY)
VISION_MODEL = "meta-llama/llama-4-scout-17b-16e-instruct"

# OpenAI example:
from openai import OpenAI
vlm_client = OpenAI(api_key=VLM_API_KEY)
VISION_MODEL = "gpt-4o"
```

The actual call uses `vlm_client.chat.completions.create(...)` — the call signature is identical across OpenAI-compatible providers so no further changes are needed.

---

### Swapping Jina AI (Embeddings)

All embedding logic lives in `get_embeddings()`. Replace the entire function body:

```python
# Original — Jina AI REST call:
MODEL_URL = "https://api.jina.ai/v1/embeddings"
embedding_model = "jina-embeddings-v5-text-small"

def get_embeddings(texts: list[str]) -> list[list[float]]:
    payload = {"model": embedding_model, "task": "text-matching", ...}
    response = requests.post(MODEL_URL, headers={..., "Authorization": EMBEDDING_API_KEY}, json=payload)
    ...

# sentence-transformers (fully local, no API key):
from sentence_transformers import SentenceTransformer
_model = SentenceTransformer("all-MiniLM-L6-v2")

def get_embeddings(texts: list[str]) -> list[list[float]]:
    return _model.encode(texts, normalize_embeddings=True).tolist()

# OpenAI embeddings:
def get_embeddings(texts: list[str]) -> list[list[float]]:
    response = openai_client.embeddings.create(model="text-embedding-3-small", input=texts)
    return [item.embedding for item in response.data]
```

`get_embeddings()` is called in three places: `embed()`, `llm_call()`, and `display_distance()` — but since the function signature `get_embeddings(texts: list[str]) -> list[list[float]]` stays the same, swapping only the internals is enough.

---

### Swapping ChromaDB (Vector Store)

Two functions handle the vector store: `innit_vector_db()` (creates the collection) and `embed()` (inserts vectors). A third pattern repeats in `llm_call()` and `display_distance()` for querying.

```python
# Original — in-memory ChromaDB:
def innit_vector_db():
    clientdb = chromadb.Client()
    collection = clientdb.get_or_create_collection(name="demo")
    return collection

# Persistent ChromaDB (survives restarts):
def innit_vector_db():
    clientdb = chromadb.PersistentClient(path="./daya_db")
    collection = clientdb.get_or_create_collection(name="demo")
    return collection
```

For a full swap to FAISS or Qdrant, you need to replace `innit_vector_db()`, the `collection.upsert(...)` call inside `embed()`, and the `collection.query(...)` calls inside `llm_call()` and `display_distance()`. The logic stays the same — initialise, insert with metadata, query by embedding distance.

---

## Acknowledgements

DAYA is built on top of outstanding open-source work:

- **[Docling](https://github.com/docling-project/docling)** — document parsing and layout understanding (MIT, IBM Research)
- **[PageIndex](https://github.com/VectifyAI/PageIndex)** — inspiration for the hierarchical tree indexing approach (MIT, VectifyAI)
- **[ChromaDB](https://github.com/chroma-core/chroma)** — vector store (Apache 2.0)
- **[Jina AI Embeddings](https://jina.ai)** — text embeddings
- **[Groq](https://groq.com)** — LLM and VLM inference
- **[python-pptx](https://github.com/scanny/python-pptx)** — PPTX parsing (MIT)
- **[EasyOCR](https://github.com/JaidedAI/EasyOCR)** — OCR support (Apache 2.0)

---

Licensed under the Apache License 2.0. See [LICENSE](./LICENSE) for details.
