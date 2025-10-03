# SurfSense Architecture Documentation

## Table of Contents
- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Core Components](#core-components)
- [Data Flow](#data-flow)
- [Technology Stack](#technology-stack)
- [Key Concepts](#key-concepts)

## Overview

SurfSense is a comprehensive AI-powered knowledge management and research platform that combines the capabilities of tools like NotebookLM and Perplexity with deep integration into your personal knowledge base. The system enables users to:

- Build and manage personal knowledge bases from multiple sources
- Search across saved content using advanced hybrid search
- Chat with documents using RAG (Retrieval Augmented Generation)
- Generate research reports with citations
- Create AI-generated podcasts from conversations
- Integrate with 15+ external data sources

## System Architecture

SurfSense follows a **three-tier architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT TIER                              │
│  ┌────────────────┐    ┌──────────────────────────┐        │
│  │ Web Application│    │  Browser Extension       │        │
│  │  (Next.js 15)  │    │  (Plasmo/Manifest v3)    │        │
│  └────────────────┘    └──────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ (REST API / WebSocket)
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION TIER                           │
│  ┌──────────────────────────────────────────────┐           │
│  │         FastAPI Backend                      │           │
│  │  ┌─────────────┐  ┌────────────────────┐    │           │
│  │  │ API Routes  │  │  AI Agents         │    │           │
│  │  │             │  │  - Researcher      │    │           │
│  │  │ - Auth      │  │  - QNA Agent       │    │           │
│  │  │ - Spaces    │  │  - Podcaster       │    │           │
│  │  │ - Documents │  │                    │    │           │
│  │  │ - Chats     │  └────────────────────┘    │           │
│  │  │ - Podcasts  │                            │           │
│  │  │ - Connectors│  ┌────────────────────┐    │           │
│  │  └─────────────┘  │ Background Tasks   │    │           │
│  │                   │ - Document Proc.   │    │           │
│  │  ┌─────────────┐  │ - Connector Sync   │    │           │
│  │  │  Services   │  │ - Embeddings       │    │           │
│  │  │             │  └────────────────────┘    │           │
│  │  │ - LLM       │                            │           │
│  │  │ - Reranker  │  ┌────────────────────┐    │           │
│  │  │ - TTS       │  │ External APIs      │    │           │
│  │  │ - Connector │  │ - Slack, Notion    │    │           │
│  │  │ - Query     │  │ - GitHub, Linear   │    │           │
│  │  └─────────────┘  │ - Jira, Discord    │    │           │
│  │                   └────────────────────┘    │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     DATA TIER                                │
│  ┌────────────────────────────────────────┐                 │
│  │  PostgreSQL + pgvector                 │                 │
│  │  ┌──────────┐  ┌──────────┐            │                 │
│  │  │  Users   │  │  Search  │            │                 │
│  │  │          │  │  Spaces  │            │                 │
│  │  └──────────┘  └──────────┘            │                 │
│  │  ┌──────────┐  ┌──────────┐            │                 │
│  │  │Documents │  │  Chunks  │            │                 │
│  │  │(+vectors)│  │(+vectors)│            │                 │
│  │  └──────────┘  └──────────┘            │                 │
│  │  ┌──────────┐  ┌──────────┐            │                 │
│  │  │  Chats   │  │ Podcasts │            │                 │
│  │  └──────────┘  └──────────┘            │                 │
│  │  ┌──────────────────────┐              │                 │
│  │  │  Connectors & Logs   │              │                 │
│  │  └──────────────────────┘              │                 │
│  └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Frontend (Next.js Web Application)

**Location**: `surfsense_web/`

The web application provides the user interface for all SurfSense features.

**Key Features**:
- **App Router (Next.js 15)**: Modern routing with server and client components
- **Authentication**: JWT-based auth with OAuth support
- **Search Spaces**: Create and manage isolated knowledge environments
- **Document Management**: Upload, view, and organize documents
- **AI Chat Interface**: Real-time streaming chat with citations
- **Research Agent**: Generate comprehensive reports with multiple depth levels
- **Podcast Generation**: Create audio podcasts from chat conversations
- **Connector Management**: Configure and sync external data sources

**Main Routes**:
- `/` - Homepage and marketing
- `/auth/*` - Authentication (login, register, OAuth callback)
- `/dashboard/searchspaces` - Manage search spaces
- `/dashboard/[search_space_id]/*` - Search space specific features
  - `/researcher` - AI research agent
  - `/chats` - Chat conversations
  - `/documents` - Document management
  - `/podcasts` - Podcast generation
  - `/connectors` - External integrations
  - `/logs` - Task logs

### 2. Backend (FastAPI Application)

**Location**: `surfsense_backend/app/`

A modern Python-based API server handling all business logic, AI operations, and data management.

**Module Structure**:

#### Routes (`app/routes/`)
API endpoints organized by domain:
- `documents_routes.py` - Document CRUD, upload, search
- `chats_routes.py` - Chat conversations and streaming
- `search_spaces_routes.py` - Search space management
- `podcasts_routes.py` - Podcast generation and management
- `search_source_connectors_routes.py` - External connector configuration
- Various connector-specific routes (Slack, Notion, GitHub, etc.)

#### Agents (`app/agents/`)
AI agents built with LangGraph:

**Researcher Agent** (`agents/researcher/`):
- **Purpose**: Generate comprehensive research reports with citations
- **Architecture**: LangGraph-based state machine
- **Modes**: 
  - QNA: Quick question-answering
  - REPORT_GENERAL: Standard research report
  - REPORT_DEEP: In-depth analysis
  - REPORT_DEEPER: Most comprehensive research

**Key Nodes**:
1. `reformulate_user_query` - Clarify and expand the user's question
2. `write_answer_outline` - Create research structure
3. `process_sections` - Generate content for each section
4. `generate_further_questions` - Suggest follow-up questions
5. `handle_qna_workflow` - Direct Q&A without full report

**Podcaster Agent** (`agents/podcaster/`):
- **Purpose**: Convert chat conversations into engaging audio podcasts
- **Features**: 
  - Multiple TTS providers (OpenAI, Kokoro local, Azure, Google)
  - Automatic script generation
  - Fast generation (~20 seconds for 3-minute podcast)

#### Services (`app/services/`)
Reusable business logic:
- `llm_service.py` - LLM integration via LiteLLM (100+ models)
- `reranker_service.py` - Result reranking (FlashRank, Cohere, etc.)
- `query_service.py` - Search and retrieval operations
- `connector_service.py` - External API integrations
- `streaming_service.py` - SSE streaming for real-time responses
- `task_logging_service.py` - Background task logging
- `kokoro_tts_service.py` - Local TTS service
- `docling_service.py` - Document parsing service

#### Tasks (`app/tasks/`)
Background processing:

**Document Processors** (`tasks/document_processors/`):
- Process uploaded files (50+ formats supported)
- Extract text from PDFs, Office docs, images
- Generate embeddings
- Chunk documents for retrieval
- Support for LlamaCloud, Unstructured.io, and Docling

**Connector Indexers** (`tasks/connector_indexers/`):
- Sync data from external sources
- Index Slack messages, Notion pages, GitHub issues, etc.
- Scheduled and manual synchronization

#### Database (`app/db.py`)
SQLAlchemy ORM models and database configuration:
- User management with FastAPI Users
- Vector storage with pgvector extension
- Async operations with asyncpg

#### Retrieval System (`app/retriver/`)
Advanced RAG implementation:
- **Hybrid Search**: Combines semantic (vector) and full-text search
- **Reciprocal Rank Fusion (RRF)**: Merges results from both search methods
- **Two-tier retrieval**: Documents and chunks level
- **Reranking**: FlashRank for improved relevance

### 3. Browser Extension

**Location**: `surfsense_browser_extension/`

A Manifest V3 browser extension built with Plasmo framework.

**Purpose**:
- Save web pages to SurfSense (especially authenticated content)
- Quick access to search functionality
- Context menu integration

**Technology**:
- Plasmo framework (React-based)
- Manifest V3 for modern browser compatibility
- Cross-browser support (Chrome, Firefox, Edge)

### 4. Database Layer

**Technology**: PostgreSQL 14+ with pgvector extension

**Schema Overview**:

```sql
-- Core Tables
- users: Authentication and user profiles
- searchspaces: Isolated knowledge environments per user
- documents: Full documents with content and embeddings
- chunks: Document fragments for granular retrieval
- chats: Conversation history and metadata
- podcasts: Generated audio content
- search_source_connectors: External integration configs
- logs: Background task execution logs

-- Vector Storage
- documents.embedding: Vector(dimension) for semantic search
- chunks.embedding: Vector(dimension) for chunk-level search
```

## Data Flow

### Document Upload and Processing Flow

```
User Uploads File
      │
      ▼
Frontend (upload/page.tsx)
      │ POST /api/v1/documents/fileupload
      ▼
Backend API (documents_routes.py)
      │
      ├─> Validate user ownership
      ├─> Save to temp location
      └─> Queue background task
            │
            ▼
      Document Processor Task
            │
            ├─> Parse file (Docling/Unstructured/LlamaCloud)
            ├─> Extract text and metadata
            ├─> Generate embedding (full document)
            ├─> Create Document record
            │
            ├─> Chunk document (Chonkie LateChunker)
            ├─> Generate embeddings (per chunk)
            └─> Create Chunk records
                  │
                  ▼
            PostgreSQL + pgvector
```

### Chat/Research Flow

```
User Sends Query
      │
      ▼
Frontend (researcher/page.tsx)
      │ POST /chat (streaming)
      ▼
Backend API (chats_routes.py)
      │
      ├─> Validate search space ownership
      └─> Stream connector search results
            │
            ▼
      Researcher Agent (LangGraph)
            │
            ├─> Reformulate query
            ├─> Search documents (hybrid search)
            │     │
            │     ├─> Vector search (pgvector)
            │     ├─> Full-text search (PostgreSQL)
            │     └─> RRF merge + rerank
            │
            ├─> Search external connectors (if enabled)
            │     └─> Query Slack, Notion, GitHub, etc.
            │
            ├─> Generate outline (LLM)
            ├─> Write sections (parallel sub-agents)
            ├─> Generate follow-up questions
            │
            └─> Stream results to frontend
                  │
                  ▼
      Frontend (real-time display)
```

### Connector Sync Flow

```
User Adds Connector
      │
      ▼
Frontend (connectors/add/*/page.tsx)
      │ POST /api/v1/search-source-connectors/
      ▼
Backend API (search_source_connectors_routes.py)
      │
      ├─> Validate credentials
      ├─> Save connector config
      └─> Queue indexing task (if indexable)
            │
            ▼
      Connector Indexer Task
            │
            ├─> Fetch data from external API
            ├─> Transform to document format
            ├─> Process each item:
            │     ├─> Generate embedding
            │     ├─> Create Document
            │     └─> Create Chunks
            │
            └─> Update last_indexed_at
                  │
                  ▼
            PostgreSQL + pgvector
```

## Technology Stack

### Backend Technologies

| Category | Technology | Purpose |
|----------|-----------|---------|
| **Framework** | FastAPI | Modern async Python web framework |
| **Database** | PostgreSQL 14+ | Relational database |
| **Vector DB** | pgvector | Vector similarity search extension |
| **ORM** | SQLAlchemy 2.0 | Database abstraction and migrations |
| **Authentication** | FastAPI Users | JWT and OAuth authentication |
| **AI Framework** | LangChain | LLM application development |
| **AI Orchestration** | LangGraph | Agent workflow state machines |
| **LLM Integration** | LiteLLM | Multi-provider LLM interface (100+ models) |
| **Embeddings** | Chonkie | Document chunking and embeddings |
| **Reranking** | FlashRank/Rerankers | Search result reranking |
| **Document Parsing** | Docling/Unstructured/LlamaCloud | Multi-format file processing |
| **Web Scraping** | Firecrawl | Web page extraction |
| **TTS** | OpenAI/Kokoro/Azure/Google | Text-to-speech synthesis |

### Frontend Technologies

| Category | Technology | Purpose |
|----------|-----------|---------|
| **Framework** | Next.js 15.2.3 | React framework with App Router |
| **Language** | TypeScript | Type-safe JavaScript |
| **UI Library** | React 19.0.0 | Component-based UI |
| **Styling** | Tailwind CSS 4.x | Utility-first CSS |
| **Components** | Shadcn/ui | Headless UI components |
| **Icons** | Lucide React | Icon library |
| **Animation** | Framer Motion | Animation library |
| **Forms** | React Hook Form + Zod | Form management and validation |
| **Notifications** | Sonner | Toast notifications |
| **Tables** | @tanstack/react-table | Data tables |
| **AI SDK** | Vercel AI SDK | Streaming chat interface |

### DevOps & Infrastructure

| Category | Technology | Purpose |
|----------|-----------|---------|
| **Containerization** | Docker | Application containerization |
| **Orchestration** | Docker Compose | Multi-container deployment |
| **Database Admin** | pgAdmin | PostgreSQL web UI |
| **Server** | Uvicorn | ASGI server for FastAPI |
| **Package Manager** | uv (Python), pnpm (Node) | Dependency management |

## Key Concepts

### 1. Search Spaces

**Concept**: Isolated knowledge environments where users can organize documents by topic, project, or any categorization.

**Benefits**:
- Separate personal and work knowledge
- Organize by project or topic
- Fine-grained access control
- Context-specific searches

**Implementation**:
- One-to-many relationship: User → SearchSpaces
- All documents, chats, and podcasts belong to a search space
- Connectors can be shared across search spaces (user-level)

### 2. Hybrid Search (RAG Foundation)

**Concept**: Combine semantic (vector) search with full-text (keyword) search for optimal retrieval.

**How it works**:
1. **Vector Search**: Find semantically similar content using embeddings
2. **Full-text Search**: Find keyword matches using PostgreSQL's text search
3. **RRF (Reciprocal Rank Fusion)**: Merge and rank results from both methods
4. **Reranking**: Use ML model to refine final ranking

**Advantages**:
- Better recall than vector-only search
- Better precision than keyword-only search
- Robust to vocabulary mismatch
- Handles both conceptual and exact matches

### 3. Two-Tier Retrieval

**Document-Level Retrieval**:
- Search across full documents
- Good for getting general context
- Used when broad understanding is needed

**Chunk-Level Retrieval** (Default):
- Search across document fragments
- More precise retrieval
- Better for specific facts and details
- Reduces token usage in LLM context

### 4. AI Agents (LangGraph)

**Researcher Agent**:
- **State Machine**: Manages complex multi-step research workflow
- **Parallel Processing**: Sub-agents work on different sections simultaneously
- **Streaming**: Real-time results to user interface
- **Citations**: Tracks sources for all claims

**QNA Sub-Agent**:
- Specialized for quick question-answering
- Bypasses full report generation
- Faster response time
- Still provides citations

### 5. Embeddings and Chunking

**Embeddings**:
- Vector representations of text
- Captures semantic meaning
- Configurable models (6000+ supported via Chonkie)
- Stored in pgvector for efficient similarity search

**Chunking Strategy**:
- **LateChunker**: Intelligent chunking based on embedding model's max sequence length
- **AutoEmbeddings**: Automatic model selection and configuration
- **Overlap**: Chunks have overlap to preserve context
- **Metadata**: Each chunk maintains reference to parent document

### 6. Background Task Processing

**Async Processing**:
- Document processing doesn't block API responses
- Large file uploads return immediately
- Progress tracked via Logs system

**Task Types**:
- Document parsing and embedding
- Connector synchronization
- Podcast generation
- Batch operations

### 7. Multi-Source Integration

**Indexable Connectors** (Data ingested into database):
- Slack, Notion, GitHub, Linear, Jira, Confluence
- Discord, ClickUp, Gmail, Calendar, Airtable, Luma
- Periodic sync to keep data fresh

**Search-Only Connectors** (Query at runtime):
- Tavily API, LinkUp API
- Real-time web search
- No local indexing required

### 8. LLM Configuration

**Multiple LLM Profiles**:
- **Fast LLM**: Quick responses (default: gpt-4o-mini)
- **Smart LLM**: Comprehensive analysis (default: gpt-4o-mini)
- **Strategic LLM**: Complex reasoning (default: gpt-4o-mini)
- **Long Context LLM**: Large documents (default: gemini-2.0-flash-thinking)

**Provider Support**:
- OpenAI, Anthropic, Google, Azure, AWS Bedrock
- Groq, Together AI, Replicate, Anyscale
- Ollama (local models)
- 100+ models via LiteLLM

### 9. Podcast Generation

**Workflow**:
1. User selects chat conversation
2. LLM generates engaging podcast script
3. TTS synthesizes audio (2 speakers)
4. Audio file saved and made available for download

**TTS Providers**:
- OpenAI (cloud, high quality)
- Kokoro (local, privacy-focused, fast)
- Azure Speech Services
- Google Vertex AI

**Features**:
- Blazingly fast (3-min podcast in ~20 seconds)
- Natural conversation flow
- Multiple voice options
- Transcript included

## Summary

SurfSense is a sophisticated AI-powered platform that combines:
- **Knowledge Management**: Organize and search personal knowledge
- **RAG (Retrieval Augmented Generation)**: Chat with documents using advanced retrieval
- **AI Research**: Generate comprehensive reports with citations
- **Multi-Source Integration**: Connect 15+ external data sources
- **Podcast Generation**: Transform conversations into audio content

The architecture is designed for:
- **Scalability**: Async operations, background tasks, efficient database queries
- **Flexibility**: Configurable LLMs, embeddings, and connectors
- **Privacy**: Self-hostable, local LLM support, no vendor lock-in
- **Extensibility**: Plugin-based connector system, modular agents

All components work together to provide a powerful, privacy-respecting alternative to cloud-only services like NotebookLM and Perplexity, with the added benefit of deep integration into your personal knowledge ecosystem.
