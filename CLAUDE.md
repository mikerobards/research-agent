# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Research Agent built with LangGraph, GPT-4o, RAG (Retrieval-Augmented Generation), Pinecone vector database, ArXiv API, and Google SerpAPI. The entire pipeline lives in a single Jupyter notebook (`project1.ipynb`).

## Setup & Commands

```bash
# Create and activate virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run the notebook
jupyter notebook project1.ipynb
```

Required environment variables in `.env`:
- `OPENAI_API_KEY`
- `PINECONE_API_KEY`
- `SERPAPI_KEY`

## Architecture

The notebook implements a sequential RAG pipeline:

**Data Acquisition** → ArXiv XML API fetches CS/AI papers, downloads PDFs to `files/`, stores metadata in `files/arxiv_dataset.json`

**Chunking** → `RecursiveCharacterTextSplitter` (512 chars, 64 overlap). Each chunk is linked to its neighbors via `prechunk_id`/`postchunk_id` fields using the ID format `{arxiv_id}#{chunk_index}`.

**Embedding & Indexing** → OpenAI `text-embedding-3-small` (1536 dims) embeddings upserted to Pinecone serverless index `langgraph-research-agent` (AWS us-east-1) in batches of 64.

**LangChain Tools** (5 tools decorated with `@tool`):
1. `fetch_arxiv(arxiv_id)` — scrapes abstract from ArXiv HTML
2. `web_search(query)` — Google search via SerpAPI (top 5)
3. `rag_search_filter(query, arxiv_id)` — vector search filtered by paper
4. `rag_search(query)` — general vector search (top 5)
5. `final_answer(...)` — compiles structured research report

**Data flow:** XML → Dict → Pandas DataFrame → Chunked DataFrame → Embeddings → Pinecone → Tool-based retrieval → LLM response

## Key Conventions

- ArXiv IDs are primary keys (e.g., `2012.12104v1`)
- Chunk IDs: `{arxiv_id}#{chunk_index}`
- Paper metadata (title, summary, authors) stored with every chunk for self-contained retrieval
- `format_rag_contexts()` helper standardizes RAG output formatting
- No separate `.py` modules — all logic in the notebook
