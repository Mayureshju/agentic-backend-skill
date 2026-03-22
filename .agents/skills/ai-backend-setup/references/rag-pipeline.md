# RAG Pipeline Reference

## Table of Contents
1. [Pipeline Architecture](#pipeline-architecture)
2. [Document Ingestion](#document-ingestion)
3. [Chunking Strategies](#chunking-strategies)
4. [Embedding](#embedding)
5. [Retrieval](#retrieval)
6. [Reranking](#reranking)
7. [Context Assembly & Generation](#context-assembly--generation)
8. [Framework Integration](#framework-integration)
9. [Pipeline API Endpoints](#pipeline-api-endpoints)

---

## Pipeline Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              INGESTION PIPELINE              │
                    │  Upload → Parse → Chunk → Embed → Store     │
                    └──────────────────┬──────────────────────────┘
                                       │ (async / background)
                                       ▼
┌──────────────┐    ┌─────────────────────────────────────────────┐
│  User Query  │───▶│            RETRIEVAL PIPELINE               │
└──────────────┘    │  Embed Query → Search → Rerank → Filter     │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │            GENERATION PIPELINE               │
                    │  Assemble Context → Build Prompt → LLM      │
                    │  → Stream Response → Log Trace               │
                    └─────────────────────────────────────────────┘
```

### Pipeline Module Structure (Python)

```
app/
├── pipeline/
│   ├── __init__.py
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── parser.py          # Document parsing (PDF, text, markdown, HTML)
│   │   ├── chunker.py         # Chunking strategies
│   │   └── processor.py       # Orchestrates parse → chunk → embed → store
│   │
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── searcher.py        # Search strategy (vector/fulltext/hybrid)
│   │   ├── reranker.py        # Cross-encoder reranking
│   │   └── retriever.py       # Orchestrates search → rerank → filter
│   │
│   ├── generation/
│   │   ├── __init__.py
│   │   ├── context.py         # Context window assembly
│   │   ├── prompts.py         # System prompts, prompt templates
│   │   └── generator.py       # LLM call with streaming
│   │
│   └── rag.py                 # End-to-end RAG orchestrator
```

### Pipeline Module Structure (TypeScript)

```
src/
├── pipeline/
│   ├── ingestion/
│   │   ├── parser.ts
│   │   ├── chunker.ts
│   │   └── processor.ts
│   ├── retrieval/
│   │   ├── searcher.ts
│   │   ├── reranker.ts
│   │   └── retriever.ts
│   ├── generation/
│   │   ├── context.ts
│   │   ├── prompts.ts
│   │   └── generator.ts
│   └── rag.ts
```

---

## Document Ingestion

### Document Parser (Python)

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from pathlib import Path

@dataclass
class ParsedDocument:
    content: str
    metadata: dict = field(default_factory=dict)
    source: str = ""
    pages: int = 0

class DocumentParser(ABC):
    @abstractmethod
    async def parse(self, file_path: Path | None = None, content: bytes | None = None) -> ParsedDocument: ...

class TextParser(DocumentParser):
    async def parse(self, file_path=None, content=None) -> ParsedDocument:
        if file_path:
            text = file_path.read_text(encoding="utf-8")
        elif content:
            text = content.decode("utf-8")
        else:
            raise ValueError("Provide file_path or content")
        return ParsedDocument(content=text, source=str(file_path or "upload"))

class PDFParser(DocumentParser):
    """Uses pymupdf (fitz) for PDF parsing."""
    async def parse(self, file_path=None, content=None) -> ParsedDocument:
        import fitz  # pymupdf
        if file_path:
            doc = fitz.open(str(file_path))
        elif content:
            doc = fitz.open(stream=content, filetype="pdf")
        else:
            raise ValueError("Provide file_path or content")

        pages_text = []
        for page in doc:
            pages_text.append(page.get_text())

        return ParsedDocument(
            content="\n\n".join(pages_text),
            metadata={"total_pages": len(doc), "title": doc.metadata.get("title", "")},
            source=str(file_path or "upload"),
            pages=len(doc),
        )

class MarkdownParser(DocumentParser):
    async def parse(self, file_path=None, content=None) -> ParsedDocument:
        if file_path:
            text = file_path.read_text(encoding="utf-8")
        elif content:
            text = content.decode("utf-8")
        else:
            raise ValueError("Provide file_path or content")
        return ParsedDocument(content=text, source=str(file_path or "upload"), metadata={"format": "markdown"})

def get_parser(file_extension: str) -> DocumentParser:
    parsers = {
        ".txt": TextParser(),
        ".md": MarkdownParser(),
        ".pdf": PDFParser(),
    }
    parser = parsers.get(file_extension.lower())
    if not parser:
        raise ValueError(f"Unsupported file type: {file_extension}")
    return parser
```

### Ingestion Processor (Python)

```python
from dataclasses import dataclass
from app.pipeline.ingestion.parser import get_parser, ParsedDocument
from app.pipeline.ingestion.chunker import Chunker
from app.services.embedding_service import EmbeddingProvider

@dataclass
class IngestedChunk:
    content: str
    embedding: list[float]
    metadata: dict
    chunk_index: int
    document_id: str

class IngestionProcessor:
    def __init__(self, chunker: Chunker, embedder: EmbeddingProvider):
        self.chunker = chunker
        self.embedder = embedder

    async def process(self, file_path=None, content=None, file_ext=".txt", document_id="", extra_metadata=None):
        # 1. Parse
        parser = get_parser(file_ext)
        parsed = await parser.parse(file_path=file_path, content=content)

        # 2. Chunk
        chunks = self.chunker.chunk(parsed.content)

        # 3. Embed (batch)
        embeddings = await self.embedder.embed_batch([c.text for c in chunks])

        # 4. Return structured chunks ready for storage
        return [
            IngestedChunk(
                content=chunk.text,
                embedding=embedding,
                metadata={**parsed.metadata, **(extra_metadata or {}), "chunk_index": i, "source": parsed.source},
                chunk_index=i,
                document_id=document_id,
            )
            for i, (chunk, embedding) in enumerate(zip(chunks, embeddings))
        ]
```

---

## Chunking Strategies

### Python Implementation

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class Chunk:
    text: str
    start_index: int
    end_index: int
    metadata: dict | None = None

class Chunker(ABC):
    @abstractmethod
    def chunk(self, text: str) -> list[Chunk]: ...

class RecursiveCharacterChunker(Chunker):
    """
    Benchmark-validated default: 512 tokens (~2048 chars) with 10-20% overlap.
    Splits by paragraphs → sentences → characters, keeping semantic units together.
    """
    def __init__(self, chunk_size: int = 2000, chunk_overlap: int = 200):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.separators = ["\n\n", "\n", ". ", " ", ""]

    def chunk(self, text: str) -> list[Chunk]:
        return self._split(text, self.separators)

    def _split(self, text: str, separators: list[str]) -> list[Chunk]:
        if not text:
            return []

        separator = separators[0]
        remaining_separators = separators[1:]

        splits = text.split(separator) if separator else list(text)
        chunks = []
        current = ""
        start = 0

        for split in splits:
            piece = split + separator if separator else split
            if len(current) + len(piece) > self.chunk_size and current:
                chunks.append(Chunk(text=current.strip(), start_index=start, end_index=start + len(current)))
                # Overlap: keep the tail of the current chunk
                overlap_text = current[-self.chunk_overlap:] if self.chunk_overlap else ""
                start = start + len(current) - len(overlap_text)
                current = overlap_text + piece
            else:
                current += piece

        if current.strip():
            chunks.append(Chunk(text=current.strip(), start_index=start, end_index=start + len(current)))

        # If any chunk is still too large and we have more separators, split further
        if remaining_separators:
            final = []
            for chunk in chunks:
                if len(chunk.text) > self.chunk_size:
                    final.extend(self._split(chunk.text, remaining_separators))
                else:
                    final.append(chunk)
            return final

        return chunks

class SemanticChunker(Chunker):
    """
    Groups sentences by embedding similarity. Requires an embedding provider.
    Higher quality but higher cost — embeds every sentence during ingestion.
    """
    def __init__(self, embedder, similarity_threshold: float = 0.75, max_chunk_size: int = 2000):
        self.embedder = embedder
        self.similarity_threshold = similarity_threshold
        self.max_chunk_size = max_chunk_size

    def chunk(self, text: str) -> list[Chunk]:
        # This is a sync wrapper; the actual embedding happens in the async process method
        # For the sync interface, fall back to recursive splitting
        return RecursiveCharacterChunker().chunk(text)

    async def chunk_async(self, text: str) -> list[Chunk]:
        import re
        import numpy as np
        sentences = re.split(r'(?<=[.!?])\s+', text)
        if len(sentences) <= 1:
            return [Chunk(text=text, start_index=0, end_index=len(text))]

        embeddings = await self.embedder.embed_batch(sentences)

        chunks = []
        current_sentences = [sentences[0]]
        start = 0

        for i in range(1, len(sentences)):
            sim = np.dot(embeddings[i], embeddings[i - 1]) / (
                np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[i - 1])
            )
            combined = " ".join(current_sentences + [sentences[i]])
            if sim >= self.similarity_threshold and len(combined) <= self.max_chunk_size:
                current_sentences.append(sentences[i])
            else:
                chunk_text = " ".join(current_sentences)
                chunks.append(Chunk(text=chunk_text, start_index=start, end_index=start + len(chunk_text)))
                start += len(chunk_text) + 1
                current_sentences = [sentences[i]]

        if current_sentences:
            chunk_text = " ".join(current_sentences)
            chunks.append(Chunk(text=chunk_text, start_index=start, end_index=start + len(chunk_text)))

        return chunks

class DocumentAwareChunker(Chunker):
    """
    Respects document structure: headings, sections, page breaks.
    Best for structured docs (PDFs, academic papers, legal docs).
    """
    def __init__(self, chunk_size: int = 2000, chunk_overlap: int = 200):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.fallback = RecursiveCharacterChunker(chunk_size, chunk_overlap)

    def chunk(self, text: str) -> list[Chunk]:
        import re
        # Split on markdown-style headings and page breaks
        sections = re.split(r'(^#{1,3}\s+.+$|\f)', text, flags=re.MULTILINE)

        chunks = []
        current = ""
        start = 0

        for section in sections:
            if len(current) + len(section) > self.chunk_size and current.strip():
                chunks.append(Chunk(text=current.strip(), start_index=start, end_index=start + len(current)))
                start += len(current)
                current = section
            else:
                current += section

        if current.strip():
            chunks.append(Chunk(text=current.strip(), start_index=start, end_index=start + len(current)))

        # Split any oversized chunks with fallback
        final = []
        for chunk in chunks:
            if len(chunk.text) > self.chunk_size * 1.5:
                final.extend(self.fallback.chunk(chunk.text))
            else:
                final.append(chunk)

        return final

def create_chunker(strategy: str, **kwargs) -> Chunker:
    strategies = {
        "recursive": RecursiveCharacterChunker,
        "semantic": SemanticChunker,
        "document_aware": DocumentAwareChunker,
    }
    chunker_cls = strategies.get(strategy)
    if not chunker_cls:
        raise ValueError(f"Unknown chunking strategy: {strategy}. Options: {list(strategies.keys())}")
    return chunker_cls(**kwargs)
```

### TypeScript Implementation

```typescript
export interface Chunk {
  text: string;
  startIndex: number;
  endIndex: number;
  metadata?: Record<string, unknown>;
}

export interface Chunker {
  chunk(text: string): Chunk[];
}

export class RecursiveCharacterChunker implements Chunker {
  constructor(
    private chunkSize = 2000,
    private chunkOverlap = 200,
    private separators = ["\n\n", "\n", ". ", " ", ""],
  ) {}

  chunk(text: string): Chunk[] {
    return this.split(text, this.separators);
  }

  private split(text: string, separators: string[]): Chunk[] {
    if (!text) return [];
    const [separator, ...rest] = separators;
    const splits = separator ? text.split(separator) : [...text];
    const chunks: Chunk[] = [];
    let current = "";
    let start = 0;

    for (const piece of splits) {
      const segment = separator ? piece + separator : piece;
      if (current.length + segment.length > this.chunkSize && current) {
        chunks.push({ text: current.trim(), startIndex: start, endIndex: start + current.length });
        const overlap = this.chunkOverlap ? current.slice(-this.chunkOverlap) : "";
        start += current.length - overlap.length;
        current = overlap + segment;
      } else {
        current += segment;
      }
    }
    if (current.trim()) {
      chunks.push({ text: current.trim(), startIndex: start, endIndex: start + current.length });
    }

    if (rest.length) {
      return chunks.flatMap((c) => (c.text.length > this.chunkSize ? this.split(c.text, rest) : [c]));
    }
    return chunks;
  }
}
```

---

## Embedding

See the stack-specific reference files (`python-sql.md`, `js-sql.md`, etc.) for embedding provider implementations. Key points:

- Use `text-embedding-3-small` (1536 dims) as default — good balance of quality and cost
- Batch embeddings during ingestion (max 2048 inputs per API call for OpenAI)
- Add retry logic with exponential backoff for API failures
- Cache embeddings — don't re-embed unchanged documents

**Additional dependency for PDF parsing:**
```
# Python
pymupdf>=1.25.0

# JavaScript
pdf-parse: ^1.1.1
```

---

## Retrieval

See the stack-specific reference files for full search implementations (vector, fulltext, hybrid). The retrieval module wraps the search with additional logic:

```python
class Retriever:
    def __init__(self, searcher, reranker=None, max_results: int = 10):
        self.searcher = searcher
        self.reranker = reranker
        self.max_results = max_results

    async def retrieve(self, query: str, query_embedding: list[float], filters: dict | None = None) -> list[dict]:
        # 1. Initial retrieval (fetch more than needed for reranking)
        fetch_k = self.max_results * 3 if self.reranker else self.max_results
        results = await self.searcher.search(query, query_embedding, limit=fetch_k, filters=filters)

        # 2. Rerank (if configured)
        if self.reranker and results:
            results = await self.reranker.rerank(query, results, top_k=self.max_results)

        return results[:self.max_results]
```

---

## Reranking

### Cross-Encoder Reranking (Python)

```python
from abc import ABC, abstractmethod

class Reranker(ABC):
    @abstractmethod
    async def rerank(self, query: str, documents: list[dict], top_k: int = 10) -> list[dict]: ...

class CohereReranker(Reranker):
    """Uses Cohere Rerank API. ~$1/1k searches, adds ~50-100ms latency."""
    def __init__(self, api_key: str, model: str = "rerank-v3.5"):
        import cohere
        self.client = cohere.AsyncClientV2(api_key=api_key)
        self.model = model

    async def rerank(self, query: str, documents: list[dict], top_k: int = 10) -> list[dict]:
        doc_texts = [d.get("content", d.get("text", "")) for d in documents]
        response = await self.client.rerank(
            query=query,
            documents=doc_texts,
            model=self.model,
            top_n=top_k,
        )
        return [
            {**documents[r.index], "rerank_score": r.relevance_score}
            for r in response.results
        ]

class CrossEncoderReranker(Reranker):
    """Uses a local cross-encoder model (e.g., BAAI/bge-reranker-v2-m3). No API cost, GPU recommended."""
    def __init__(self, model_name: str = "BAAI/bge-reranker-v2-m3"):
        from sentence_transformers import CrossEncoder
        self.model = CrossEncoder(model_name)

    async def rerank(self, query: str, documents: list[dict], top_k: int = 10) -> list[dict]:
        import asyncio
        doc_texts = [d.get("content", d.get("text", "")) for d in documents]
        pairs = [(query, text) for text in doc_texts]

        # Run sync model in thread pool
        loop = asyncio.get_event_loop()
        scores = await loop.run_in_executor(None, lambda: self.model.predict(pairs))

        scored = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
        return [{**doc, "rerank_score": float(score)} for doc, score in scored[:top_k]]

class NoReranker(Reranker):
    """Pass-through for when reranking is disabled."""
    async def rerank(self, query: str, documents: list[dict], top_k: int = 10) -> list[dict]:
        return documents[:top_k]

def create_reranker(strategy: str, **kwargs) -> Reranker:
    if strategy == "cohere":
        return CohereReranker(**kwargs)
    elif strategy == "cross_encoder":
        return CrossEncoderReranker(**kwargs)
    elif strategy == "none":
        return NoReranker()
    raise ValueError(f"Unknown reranker: {strategy}")
```

### Reranking (TypeScript)

```typescript
export interface Reranker {
  rerank(query: string, documents: Array<Record<string, unknown>>, topK?: number): Promise<Array<Record<string, unknown>>>;
}

export class CohereReranker implements Reranker {
  constructor(private apiKey: string, private model = "rerank-v3.5") {}

  async rerank(query: string, documents: Array<Record<string, unknown>>, topK = 10) {
    const response = await fetch("https://api.cohere.ai/v2/rerank", {
      method: "POST",
      headers: { Authorization: `Bearer ${this.apiKey}`, "Content-Type": "application/json" },
      body: JSON.stringify({
        query,
        documents: documents.map((d) => (d.content ?? d.text) as string),
        model: this.model,
        top_n: topK,
      }),
    });
    const data = await response.json();
    return data.results.map((r: { index: number; relevance_score: number }) => ({
      ...documents[r.index],
      rerankScore: r.relevance_score,
    }));
  }
}
```

**Dependencies for reranking:**
```
# Python (if using Cohere)
cohere>=5.13.0

# Python (if using local cross-encoder)
sentence-transformers>=3.4.0

# TypeScript (Cohere uses fetch — no extra dependency)
```

---

## Context Assembly & Generation

### Context Builder (Python)

```python
@dataclass
class RAGContext:
    chunks: list[dict]
    query: str
    system_prompt: str
    context_text: str
    sources: list[dict]

class ContextAssembler:
    def __init__(self, max_context_tokens: int = 4000):
        self.max_context_tokens = max_context_tokens

    def assemble(self, query: str, chunks: list[dict], system_prompt: str = "") -> RAGContext:
        context_parts = []
        sources = []
        token_count = 0

        for i, chunk in enumerate(chunks):
            chunk_text = chunk.get("content", chunk.get("text", ""))
            # Rough token estimate: 1 token ≈ 4 chars
            chunk_tokens = len(chunk_text) // 4
            if token_count + chunk_tokens > self.max_context_tokens:
                break
            context_parts.append(f"[Source {i+1}]: {chunk_text}")
            sources.append({
                "index": i + 1,
                "document_id": chunk.get("document_id", ""),
                "title": chunk.get("title", ""),
                "chunk_index": chunk.get("chunk_index", 0),
                "score": chunk.get("rerank_score", chunk.get("similarity", 0)),
            })
            token_count += chunk_tokens

        context_text = "\n\n".join(context_parts)

        default_system = (
            "You are a helpful assistant. Answer the user's question based on the provided context. "
            "If the context doesn't contain enough information to answer, say so. "
            "Cite your sources using [Source N] notation."
        )

        return RAGContext(
            chunks=chunks[:len(sources)],
            query=query,
            system_prompt=system_prompt or default_system,
            context_text=context_text,
            sources=sources,
        )
```

### RAG Orchestrator (Python)

```python
class RAGPipeline:
    def __init__(self, embedder, retriever, context_assembler, llm_provider):
        self.embedder = embedder
        self.retriever = retriever
        self.context_assembler = context_assembler
        self.llm = llm_provider

    async def query(self, question: str, filters: dict | None = None, stream: bool = False):
        # 1. Embed the query
        query_embedding = await self.embedder.embed(question)

        # 2. Retrieve relevant chunks
        chunks = await self.retriever.retrieve(question, query_embedding, filters=filters)

        # 3. Assemble context
        context = self.context_assembler.assemble(question, chunks)

        # 4. Generate response
        messages = [
            LLMMessage(role="system", content=context.system_prompt),
            LLMMessage(role="user", content=f"Context:\n{context.context_text}\n\nQuestion: {question}"),
        ]

        if stream:
            return self.llm.chat_stream(messages), context.sources
        else:
            response = await self.llm.chat(messages)
            return {"answer": response.content, "sources": context.sources, "usage": response.usage}
```

---

## Framework Integration

### LangChain RAG Pipeline

When the user selects LangChain, use LangChain's components instead of custom implementations:

```python
# Key LangChain components to use:
from langchain_community.document_loaders import PyMuPDFLoader, TextLoader, UnstructuredMarkdownLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI  # or Anthropic/OpenRouter equivalents
from langchain_postgres import PGVector  # or langchain_mongodb
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate

# Additional dependencies:
# langchain>=0.3.15
# langchain-openai>=0.3.0
# langchain-anthropic>=0.3.0
# langchain-community>=0.3.15
# langchain-postgres>=0.0.12   (for pgvector)
# langchain-mongodb>=0.3.0     (for MongoDB)
```

### LlamaIndex RAG Pipeline

When the user selects LlamaIndex:

```python
# Key LlamaIndex components:
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader, Settings, StorageContext
from llama_index.core.node_parser import SentenceSplitter, SemanticSplitterNodeParser
from llama_index.vector_stores.postgres import PGVectorStore  # or MongoDBAtlasVectorSearch
from llama_index.llms.openai import OpenAI  # or Anthropic
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.core.postprocessor import SentenceTransformerRerank

# Additional dependencies:
# llama-index>=0.12.0
# llama-index-vector-stores-postgres>=0.4.0  (for pgvector)
# llama-index-vector-stores-mongodb>=0.4.0   (for MongoDB)
# llama-index-llms-openai>=0.4.0
# llama-index-llms-anthropic>=0.4.0
# llama-index-embeddings-openai>=0.4.0
```

### No Framework (Direct)

When the user selects no framework, use the custom implementations in this reference file directly. The pipeline module structure above is designed for this path.

---

## Pipeline API Endpoints

```
# Ingestion
POST   /api/v1/documents/upload        # Upload and ingest a document (multipart/form-data)
POST   /api/v1/documents/ingest-text   # Ingest raw text content
GET    /api/v1/documents/:id/chunks    # List chunks for a document
DELETE /api/v1/documents/:id           # Delete document and its chunks/embeddings

# Querying
POST   /api/v1/query                   # RAG query (returns answer + sources)
POST   /api/v1/query/stream            # Streaming RAG query (SSE)
POST   /api/v1/search                  # Search only (no LLM generation, returns chunks)

# Conversation (if chat-style)
POST   /api/v1/conversations                    # Create conversation
POST   /api/v1/conversations/:id/messages       # Send message (triggers RAG pipeline)
GET    /api/v1/conversations/:id/messages       # Get conversation history
```
