# Grounding AI in Reality: How We Made Joule Answer From Our Own Documents

There's a moment in most AI projects where you realize the model confidently gave you a completely wrong answer — and you have no idea how to fix it. You can't retrain it, you can't shout at it, and you definitely can't ship it to users like that.

That was roughly where we were when we started experimenting with **Document Grounding on SAP BTP**. We wanted Joule (SAP's AI assistant) to answer questions from a specific equipment manual — not from its general training data, but from our actual PDF. This post is about how we got there.

---

## What We Were Trying to Do

The goal was simple on paper: take a technical PDF (an operator manual for a loader), upload it to SAP's Document Grounding service, and let Joule search through it when users ask questions. Instead of guessing or hallucinating, Joule would pull actual passages from the document and answer based on those.

The underlying pattern is called **RAG** — Retrieval-Augmented Generation. You store your documents as vectors (mathematical representations of meaning), and when a question comes in, you find the most semantically relevant chunks and feed them to the LLM as context. It's become the standard way to make AI useful on private knowledge bases.

SAP BTP has its own managed infrastructure for this through the **Document Grounding service**, and there's a Python library called `langchain-rage` that wraps its API in a LangChain-compatible interface.

---

## Prerequisites

Before you try to replicate this, you'll need:

- **An SAP BTP account** with the Document Grounding service instance provisioned
- **Identity service** set up with X.509 certificate support (this is what handles authentication)
- **Service keys** generated from both the Document Grounding service and the Identity service — you'll need these as JSON files locally
- **Python 3.11+** — we used 3.11.14
- **A PDF you want to search** — for our test we used a 114-page equipment manual
- `cf` CLI access to pull fresh service keys if needed
- Joule Studio access if you want to test the end-to-end conversational integration

> **Note on the stage environment**: We were working against eu12 (canary/stage). This environment can be flaky — timeouts happen, uploads occasionally fail midway. Plan your ingestion scripts with that in mind from the start.

---

## How the Pieces Fit Together

Before diving into setup, here's the high-level picture:

```
Your PDF
   ↓
Load pages with PyMuPDF
   ↓
Split into ~1000-character chunks (LangChain splitter)
   ↓
Upload to BTP Document Grounding (vector store)
   ↓
BTP auto-generates embeddings for each chunk
   ↓
User asks a question in Joule or your app
   ↓
Semantic search finds the most relevant chunks
   ↓
LLM generates a grounded, cited answer
```

The service handles the embedding generation — you don't need to bring your own embedding model or manage any of that infrastructure. That's honestly one of the nicer aspects of this approach.

---

## Setting Up the Environment

```bash
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install langchain-community langchain-openai pymupdf requests-toolbelt

# Install the SAP RAGe LangChain integration (from internal git)
pip install git+https://<your-internal-repo>/langchain-rage.git
```

Place your service key files in the project root:
- `doc-grounding-service-key-fresh.json` — from your Document Grounding service instance
- `identity-key-x509-fresh.json` — from your Identity service instance (must include X.509 certs)

> If your service keys are more than a day or two old, regenerate them via CF CLI. Stale keys were a source of several confusing authentication failures for us.

---

## Step 1: Create the Collection

A "collection" is the vector store bucket that holds your document chunks. The key thing to get right is the metadata — specifically setting `type=custom`. Without this, Joule won't discover the collection when grounding responses.

```python
from langchain_rage.rage_clients.vector.client import VectorClient

vector_client = VectorClient.create_from_service_keys(
    path_document_grounding_key="doc-grounding-service-key-fresh.json",
    path_identity_key="identity-key-x509-fresh.json"
)

# Create the collection
collection = vector_client.create_collection(
    title="my-document-collection",
    metadata={"type": "custom"}  # Required for Joule integration
)

collection_id = collection.id
print(f"Collection created: {collection_id}")
```

Save that collection ID somewhere — you'll reference it in every subsequent operation.

---

## Step 2: Ingest Your PDF

This is where most of the work happens. We load the PDF with PyMuPDF (which preserves page numbers in metadata — important for citations), split into chunks, convert to the RAGe format, and upload.

```python
from langchain_community.document_loaders import DirectoryLoader, PyMuPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_rage.rage_clients.vector.model import (
    Chunk as RAGeChunk,
    Document as RAGeDocument,
    MetaData as RAGeMetaData
)
import time

# Load
loader = DirectoryLoader(path=".", glob="*.pdf", loader_cls=PyMuPDFLoader)
raw_documents = loader.load()

# Split
chunker = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100
)

# Convert and upload in batches
batch_size = 50
all_ids = []

for raw_doc in raw_documents:
    chunks = chunker.split_documents([raw_doc])
    rage_docs = []

    for chunk in chunks:
        rage_chunk = RAGeChunk(
            content=chunk.page_content,
            metadata=[
                RAGeMetaData(key=k, value=[str(v)])
                for k, v in chunk.metadata.items()
            ]
        )
        rage_doc = RAGeDocument(
            chunks=[rage_chunk],
            metadata=[RAGeMetaData(key="title", value=[raw_doc.metadata["source"]])]
        )
        rage_docs.append(rage_doc)

    # Upload in batches
    for i in range(0, len(rage_docs), batch_size):
        batch = rage_docs[i:i + batch_size]
        try:
            ids = vector_client.create_documents(
                collection_id=collection_id,
                documents=batch
            )
            all_ids.extend(ids)
            time.sleep(1)  # Be nice to the stage environment
        except Exception as e:
            print(f"Batch failed: {e}")
            continue

print(f"Uploaded {len(all_ids)} chunks")
```

**About chunk sizing**: We settled on 1000 characters with 100 character overlap. Smaller chunks give more precise retrieval but miss broader context; larger chunks are the opposite. 1000 works well for technical documentation with a mix of lists, tables, and prose. You'll likely need to tune this for your content type.

**About batching**: If you're on stage/canary, do not try to upload everything in one shot. We lost entire uploads that way. Batches of 50 with a 1-second delay between them worked reliably. If you're resuming a partial upload, drop to batches of 10 and add a 3-second delay.

For a 114-page PDF, we ended up with 233 chunks total. The whole upload took a few minutes with batching.

---

## Step 3: Verify the Collection

Before moving on, it's worth checking that the upload actually worked:

```python
status = vector_client.get_collection(collection_id=collection_id)
print(f"Collection status: {status}")
```

You can also run a quick semantic query to verify the chunks are searchable:

```python
from langchain_rage.rage_clients.retrieval.client import RetrievalClient
from langchain_rage.rage_clients.retrieval.model import Search, SearchFilter, SearchConfiguration

retrieval_client = RetrievalClient.create_from_service_keys(
    path_document_grounding_key="doc-grounding-service-key-fresh.json",
    path_identity_key="identity-key-x509-fresh.json"
)

search = Search(
    query="maintenance intervals",
    filters=[SearchFilter(
        dataRepositories=[collection_id],
        dataRepositoryType="vector",
        searchConfiguration=SearchConfiguration(maxChunkCount=3)
    )]
)

results = retrieval_client.query(search).json()
```

If you get chunks back with relevance scores in the 0.4–0.6 range, you're in good shape. Scores above 0.5 on a meaningful query indicate solid embedding quality.

---

## Step 4: Joule Integration

Once your collection is live, connecting it to Joule requires a deployment through Joule Studio. The key configuration is the capability definition — specifically making sure the capability points to your collection and is marked with the right metadata so Joule's grounding layer can discover it.

The deployment manifest structure looks roughly like:

```yaml
# da.sapdas.yaml (simplified)
apiVersion: sapdas/v1
kind: DeploymentAssistant
metadata:
  name: my-document-grounding-capability
spec:
  capabilities:
    - path: a2a/
```

```yaml
# a2a/capability.sapdas.yaml (simplified)
apiVersion: sapdas/v3
kind: Capability
metadata:
  name: document-search
spec:
  grounding:
    collections:
      - id: "<your-collection-id>"
        type: custom
```

After deploying via Joule Studio's CLI, Joule will automatically use your collection when users ask questions that match its scope. The grounding is transparent — users just see answers, and Joule cites the source pages.

---

## What It Looks Like in Practice

Once everything is connected, the experience is straightforward. A user asks Joule something like "what's the maintenance schedule for the first 100 operating hours?" — Joule runs a semantic search against the collection, finds the relevant passages (in our case from pages 62 and 76–77 of the manual), and generates a response grounded in those chunks.

The relevance scores we saw in testing:

| Query | Score Range | Source Pages |
|-------|------------|--------------|
| "maintenance intervals" | 0.50 – 0.56 | 62, 76–77 |
| "safety requirements before operation" | 0.45 – 0.50 | 18 |
| "engine oil change procedure" | 0.50 – 0.55 | 76–77 |
| "technical specifications" | 0.48 – 0.54 | 30 |

Not all queries will score high — if a topic isn't covered in your document, scores drop noticeably (below 0.3) and Joule will typically indicate it couldn't find relevant information. That failure mode is actually what you want: grounded AI that knows what it doesn't know.

---

## A Few Things We'd Do Differently

**Start with a resume-capable ingestion script.** We wrote `ingest_pdf.py` first (one big upload), then `ingest_pdf_batched.py` (batched), then `ingest_pdf_resume.py` (resume from checkpoint). We should have started with the third one. Save your uploaded document IDs to a file as you go, and on retry, skip chunks that already made it through.

**Test semantic search before deploying to Joule.** We spent time debugging the Joule integration when the actual problem was that some chunks didn't upload. A simple retrieval test catches this early.

**Keep your service keys fresh.** X.509 certificates have expiry dates. If you hit sudden authentication failures days later, regenerating keys is the first thing to try.

**Name your collections descriptively.** We used `joule-document-test` which was fine for a PoC but would get confusing fast in a real environment with multiple document sets.

---

## Wrapping Up

Document grounding is one of those features where the demo works immediately and then you discover all the small things that make it production-ready: batch handling, error recovery, chunk size tuning, collection discoverability. The SAP BTP Document Grounding service handles the hard parts (embedding generation, vector storage, similarity search) reasonably well — the `langchain-rage` integration makes it approachable from Python without dealing with raw API calls.

If you're building anything where you need AI to answer reliably from a known corpus — internal documentation, product manuals, compliance documents — this pattern is worth understanding. It's not magic, but it's close enough.

---

*Setup based on SAP BTP Document Grounding service (eu12 region, stage environment). The `langchain-rage` library is an internal SAP integration library for LangChain. Joule Studio is required for the conversational integration step.*
