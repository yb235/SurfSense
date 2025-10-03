# SurfSense Workflows and Data Flow

This document explains the key workflows and data flows in SurfSense, helping you understand how different components interact.

## Table of Contents
- [Document Upload and Processing](#document-upload-and-processing)
- [Search and Retrieval](#search-and-retrieval)
- [AI Chat and Research](#ai-chat-and-research)
- [Podcast Generation](#podcast-generation)
- [Connector Synchronization](#connector-synchronization)
- [Authentication Flow](#authentication-flow)

## Document Upload and Processing

### User Flow

1. User selects search space
2. Navigates to "Upload Documents"
3. Drag-and-drops files or clicks to browse
4. Reviews selected files
5. Clicks "Upload"
6. System shows upload progress
7. User is redirected to documents list
8. Processing happens in background

### Technical Flow

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │ 1. POST /api/v1/documents/fileupload
       │    FormData: files[], search_space_id
       ▼
┌─────────────────────────────────────────┐
│         FastAPI Backend                 │
│  ┌───────────────────────────────────┐  │
│  │ documents_routes.py               │  │
│  │                                   │  │
│  │ 1. Validate user authentication   │  │
│  │ 2. Check search space ownership   │  │
│  │ 3. Save files to temp location    │  │
│  │ 4. Queue background task          │  │
│  │ 5. Return success immediately     │  │
│  └───────────────────────────────────┘  │
└─────────────┬───────────────────────────┘
              │
              │ 2. Background Task Queued
              ▼
┌─────────────────────────────────────────┐
│    Background Task Processor            │
│  ┌───────────────────────────────────┐  │
│  │ process_file_in_background        │  │
│  │                                   │  │
│  │ 1. Detect file type               │  │
│  │ 2. Select appropriate parser:     │  │
│  │    - Docling (local, free)        │  │
│  │    - Unstructured (34+ formats)   │  │
│  │    - LlamaCloud (50+ formats)     │  │
│  │ 3. Extract text and metadata      │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              │ 3. Parse Complete         │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Generate Document Embedding       │  │
│  │                                   │  │
│  │ 1. Call embedding API/model       │  │
│  │    - OpenAI text-embedding-ada-002│  │
│  │    - Local sentence-transformers  │  │
│  │    - Other Chonkie-supported     │  │
│  │ 2. Get vector representation      │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              │ 4. Embedding Generated    │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Create Document Record            │  │
│  │                                   │  │
│  │ Document {                        │  │
│  │   title,                          │  │
│  │   content,                        │  │
│  │   embedding (vector),             │  │
│  │   document_type,                  │  │
│  │   document_metadata,              │  │
│  │   search_space_id                 │  │
│  │ }                                 │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              │ 5. Document Saved         │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Chunk Document                    │  │
│  │                                   │  │
│  │ Using Chonkie LateChunker:        │  │
│  │ 1. Split based on embedding model │  │
│  │    max sequence length            │  │
│  │ 2. Add overlap for context        │  │
│  │ 3. Maintain metadata references   │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              │ 6. Chunks Created         │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Generate Chunk Embeddings         │  │
│  │                                   │  │
│  │ For each chunk:                   │  │
│  │ 1. Generate embedding vector      │  │
│  │ 2. Create Chunk record {          │  │
│  │      content,                     │  │
│  │      embedding (vector),          │  │
│  │      document_id                  │  │
│  │    }                              │  │
│  └───────────┬───────────────────────┘  │
└──────────────┼───────────────────────────┘
               │
               │ 7. Save All Chunks
               ▼
┌─────────────────────────────────────────┐
│    PostgreSQL + pgvector                │
│                                         │
│  documents table:                       │
│    - Document with full text            │
│    - Document-level embedding           │
│                                         │
│  chunks table:                          │
│    - Multiple chunks per document       │
│    - Each with its own embedding        │
│    - Foreign key to parent document     │
└─────────────────────────────────────────┘

Log entries created at each step for tracking
```

### Error Handling

- **File too large**: Return 413 error immediately
- **Unsupported format**: Log error, skip file
- **Parsing failure**: Log error, mark document as failed
- **Embedding failure**: Retry with exponential backoff
- **Database error**: Rollback transaction, log error

### Supported File Formats

**Via LlamaCloud** (50+ formats):
- Documents: PDF, DOCX, DOC, RTF, TXT, MD, HTML, EPUB
- Spreadsheets: XLSX, XLS, CSV, ODS
- Presentations: PPTX, PPT, KEY
- Images: JPG, PNG, TIFF, BMP, SVG, WEBP
- And many more...

**Via Unstructured** (34+ formats):
- Core formats: PDF, DOCX, XLSX, PPTX, HTML, MD, TXT
- Images with OCR: JPG, PNG, TIFF, HEIC
- Email: EML, MSG

**Via Docling** (local, free):
- PDF, DOCX, HTML, Images, CSV, XLSX

## Search and Retrieval

### User Flow

1. User types query in search/chat interface
2. Selects search mode (Chunks/Documents)
3. Submits query
4. System performs hybrid search
5. Results displayed with relevance scores

### Technical Flow: Hybrid Search

```
┌─────────────┐
│ User Query  │ "What are the main findings?"
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│     Query Processing                    │
│  1. Clean and normalize query           │
│  2. Generate query embedding            │
└──────┬──────────────────────────────────┘
       │
       ├────────────────────┬──────────────────────┐
       │                    │                      │
       ▼                    ▼                      ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ Vector Search    │ │ Full-Text Search │ │ Metadata Filter  │
│                  │ │                  │ │                  │
│ SELECT *         │ │ SELECT *         │ │ Filter by:       │
│ FROM chunks      │ │ FROM chunks      │ │ - search_space   │
│ WHERE            │ │ WHERE            │ │ - document_type  │
│  search_space_id │ │  search_space_id │ │ - date range     │
│ ORDER BY         │ │  AND             │ │                  │
│  embedding       │ │  content @@      │ │                  │
│  <=>             │ │  to_tsquery()    │ │                  │
│  query_embedding │ │ ORDER BY         │ │                  │
│ LIMIT 20         │ │  ts_rank         │ │                  │
│                  │ │ LIMIT 20         │ │                  │
└────────┬─────────┘ └────────┬─────────┘ └──────────────────┘
         │                    │
         │   Results Set A    │   Results Set B
         │                    │
         └────────┬───────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│ Reciprocal Rank Fusion (RRF)           │
│                                         │
│ For each result in both sets:          │
│   score = 1/(rank_A + k) +             │
│           1/(rank_B + k)               │
│   where k = 60 (constant)              │
│                                         │
│ Sort by combined score                 │
└──────┬──────────────────────────────────┘
       │
       │ Merged Results (top 20)
       ▼
┌─────────────────────────────────────────┐
│ Reranking (FlashRank)                  │
│                                         │
│ 1. Pass query + results to reranker    │
│ 2. ML model scores semantic relevance  │
│ 3. Reorder by new scores               │
│ 4. Return top K results                │
└──────┬──────────────────────────────────┘
       │
       │ Final Ranked Results
       ▼
┌─────────────────────────────────────────┐
│ Return to User                          │
│                                         │
│ Results with:                           │
│ - Relevance scores                      │
│ - Document references                   │
│ - Chunk content                         │
│ - Metadata                              │
└─────────────────────────────────────────┘
```

### Why Hybrid Search?

**Vector Search Alone**:
- ✅ Finds semantically similar content
- ❌ May miss exact keyword matches
- ❌ Vocabulary mismatch issues

**Full-Text Search Alone**:
- ✅ Finds exact keyword matches
- ❌ Misses semantic similarity
- ❌ Requires exact word matches

**Hybrid Search** (Vector + Full-Text):
- ✅ Best of both worlds
- ✅ Better recall and precision
- ✅ Handles both semantic and exact matches

## AI Chat and Research

### User Flow

1. User enters question in researcher interface
2. Selects research mode (QNA/General/Deep/Deeper)
3. Optionally selects external connectors
4. Submits query
5. Watch real-time streaming response
6. Review citations
7. Explore follow-up questions

### Technical Flow: Research Agent

```
┌─────────────┐
│ User Query  │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Frontend: POST /chat                   │
│                                         │
│  {                                      │
│    messages: [...],                     │
│    data: {                              │
│      search_space_id,                   │
│      research_mode: "REPORT_DEEP",      │
│      selected_connectors: [...]         │
│    }                                    │
│  }                                      │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Backend: Stream Response (SSE)         │
│  stream_connector_search_results()      │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  LangGraph Researcher Agent             │
│  ┌───────────────────────────────────┐  │
│  │ Node 1: Reformulate Query        │  │
│  │                                   │  │
│  │ Input: "What are the findings?"   │  │
│  │ Output: "Provide comprehensive    │  │
│  │          overview of key findings │  │
│  │          and conclusions from     │  │
│  │          the reports"             │  │
│  │                                   │  │
│  │ Stream: {type: "reformulated"}    │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Conditional Routing               │  │
│  │                                   │  │
│  │ If QNA mode:                      │  │
│  │   → handle_qna_workflow           │  │
│  │ Else:                             │  │
│  │   → write_answer_outline          │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼ (Report Mode)             │
│  ┌───────────────────────────────────┐  │
│  │ Node 2: Write Answer Outline      │  │
│  │                                   │  │
│  │ 1. Retrieve relevant documents    │  │
│  │    via hybrid search              │  │
│  │                                   │  │
│  │ 2. Query external connectors      │  │
│  │    (if selected)                  │  │
│  │                                   │  │
│  │ 3. Generate report outline:       │  │
│  │    - Introduction                 │  │
│  │    - Main Findings                │  │
│  │    - Analysis                     │  │
│  │    - Conclusions                  │  │
│  │                                   │  │
│  │ Stream: {type: "outline", ...}    │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Node 3: Process Sections          │  │
│  │                                   │  │
│  │ For each section in outline:      │  │
│  │                                   │  │
│  │ ┌─────────────────────────────┐   │  │
│  │ │ Sub-Agent: Write Section    │   │  │
│  │ │                             │   │  │
│  │ │ 1. Re-retrieve docs for     │   │  │
│  │ │    this specific section    │   │  │
│  │ │                             │   │  │
│  │ │ 2. Generate section content │   │  │
│  │ │                             │   │  │
│  │ │ 3. Track citations          │   │  │
│  │ │                             │   │  │
│  │ │ 4. Stream incrementally     │   │  │
│  │ └─────────────────────────────┘   │  │
│  │                                   │  │
│  │ Sections processed in parallel    │  │
│  │                                   │  │
│  │ Stream for each section:          │  │
│  │   {type: "section_title"}         │  │
│  │   {type: "section_content"}       │  │
│  │   {type: "citation"}              │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Node 4: Generate Follow-up Qs     │  │
│  │                                   │  │
│  │ Based on full report + sources:   │  │
│  │ - "How do these findings compare  │  │
│  │    to previous quarters?"         │  │
│  │ - "What are the implications?"    │  │
│  │ - "What data supports this?"      │  │
│  │                                   │  │
│  │ Stream: {type: "followup_questions"}│ │
│  └───────────┬───────────────────────┘  │
└──────────────┼───────────────────────────┘
               │
               │ All events streamed
               ▼
┌─────────────────────────────────────────┐
│  Frontend: Real-time Display           │
│                                         │
│  1. Receive SSE events                  │
│  2. Update UI incrementally:            │
│     - Show sections as they arrive      │
│     - Add citations dynamically         │
│     - Display follow-up questions       │
│  3. Provide smooth user experience      │
└─────────────────────────────────────────┘
```

### QNA Workflow (Simpler Path)

```
User Query
    │
    ▼
Reformulate Query
    │
    ▼
Retrieve Documents (Hybrid Search)
    │
    ▼
Generate Direct Answer
    │  - Use LLM with retrieved context
    │  - Include citations
    │  - Generate follow-up questions
    ▼
Stream to User
```

## Podcast Generation

### User Flow

1. User navigates to Podcasts
2. Clicks "Create New Podcast"
3. Selects source chat conversation
4. Enters podcast title
5. Clicks "Generate"
6. Waits ~20 seconds
7. Podcast ready to play/download

### Technical Flow

```
┌─────────────┐
│   User      │ Create Podcast from Chat #123
└──────┬──────┘
       │
       │ POST /api/v1/podcasts/
       │ {search_space_id, title, chat_id}
       ▼
┌─────────────────────────────────────────┐
│  Backend: Podcast Route                 │
│                                         │
│  1. Validate ownership                  │
│  2. Fetch chat conversation             │
│  3. Queue podcast generation task       │
│  4. Return podcast record               │
└──────┬──────────────────────────────────┘
       │
       │ Background Task
       ▼
┌─────────────────────────────────────────┐
│  Podcaster Agent                        │
│  ┌───────────────────────────────────┐  │
│  │ Step 1: Generate Script           │  │
│  │                                   │  │
│  │ Input: Chat messages              │  │
│  │                                   │  │
│  │ Prompt LLM to create engaging     │  │
│  │ dialogue between 2 speakers:      │  │
│  │                                   │  │
│  │ Output:                           │  │
│  │ {                                 │  │
│  │   speakers: ["Alex", "Sam"],      │  │
│  │   dialogue: [                     │  │
│  │     {                             │  │
│  │       speaker: "Alex",            │  │
│  │       text: "Welcome to..."       │  │
│  │     },                            │  │
│  │     {                             │  │
│  │       speaker: "Sam",             │  │
│  │       text: "Thanks..."           │  │
│  │     }                             │  │
│  │   ]                               │  │
│  │ }                                 │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 2: Text-to-Speech            │  │
│  │                                   │  │
│  │ For each dialogue segment:        │  │
│  │                                   │  │
│  │ Voice Assignment:                 │  │
│  │ - Alex → Voice 1 (e.g., alloy)    │  │
│  │ - Sam → Voice 2 (e.g., nova)      │  │
│  │                                   │  │
│  │ TTS Provider (configurable):      │  │
│  │ ┌─────────────────────────────┐   │  │
│  │ │ OpenAI TTS                  │   │  │
│  │ │ - High quality             │   │  │
│  │ │ - Multiple voices          │   │  │
│  │ │ - Cloud-based              │   │  │
│  │ └─────────────────────────────┘   │  │
│  │ ┌─────────────────────────────┐   │  │
│  │ │ Kokoro TTS (local)         │   │  │
│  │ │ - Privacy-focused          │   │  │
│  │ │ - Fast (runs locally)      │   │  │
│  │ │ - Free                     │   │  │
│  │ └─────────────────────────────┘   │  │
│  │ ┌─────────────────────────────┐   │  │
│  │ │ Azure/Google TTS           │   │  │
│  │ │ - Enterprise options       │   │  │
│  │ └─────────────────────────────┘   │  │
│  │                                   │  │
│  │ Generate audio for each segment   │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 3: Audio Concatenation       │  │
│  │                                   │  │
│  │ 1. Combine all audio segments     │  │
│  │ 2. Add pauses between speakers    │  │
│  │ 3. Normalize audio levels         │  │
│  │ 4. Export as MP3                  │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 4: Save Results              │  │
│  │                                   │  │
│  │ 1. Save audio file                │  │
│  │ 2. Update podcast record:         │  │
│  │    - file_location                │  │
│  │    - podcast_transcript           │  │
│  │ 3. Mark as complete               │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  User Can Now:                          │
│  - Play podcast in browser              │
│  - Download MP3 file                    │
│  - Read transcript                      │
└─────────────────────────────────────────┘
```

### Performance

- **3-minute podcast**: ~20 seconds generation time
- **Parallel processing**: TTS segments processed concurrently
- **Caching**: Script generation cached if regenerating

## Connector Synchronization

### User Flow

1. User adds connector (e.g., Slack)
2. Provides credentials and configuration
3. System validates connection
4. User enables indexing
5. System syncs data in background
6. Data becomes searchable

### Technical Flow

```
┌─────────────┐
│   User      │ Add Slack Connector
└──────┬──────┘
       │
       │ POST /api/v1/search-source-connectors/
       │ {
       │   name: "Work Slack",
       │   connector_type: "SLACK_CONNECTOR",
       │   is_indexable: true,
       │   config: {token, channels}
       │ }
       ▼
┌─────────────────────────────────────────┐
│  Backend: Connector Route               │
│                                         │
│  1. Validate connector config           │
│  2. Test connection to Slack API        │
│  3. Save connector record               │
│  4. If indexable, queue indexing task   │
└──────┬──────────────────────────────────┘
       │
       │ Background Task
       ▼
┌─────────────────────────────────────────┐
│  Connector Indexer                      │
│  ┌───────────────────────────────────┐  │
│  │ Step 1: Fetch Data                │  │
│  │                                   │  │
│  │ For Slack:                        │  │
│  │ - conversations.history()         │  │
│  │ - For each configured channel     │  │
│  │ - Since last_indexed_at           │  │
│  │                                   │  │
│  │ For Notion:                       │  │
│  │ - Search pages                    │  │
│  │ - Get page content                │  │
│  │                                   │  │
│  │ For GitHub:                       │  │
│  │ - List issues/PRs                 │  │
│  │ - Get comments                    │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 2: Transform Data            │  │
│  │                                   │  │
│  │ Convert external format to        │  │
│  │ SurfSense document format:        │  │
│  │                                   │  │
│  │ {                                 │  │
│  │   title: "Message in #general",   │  │
│  │   content: "Message text...",     │  │
│  │   document_type: "SLACK_CONNECTOR"│  │
│  │   document_metadata: {            │  │
│  │     channel, timestamp, author    │  │
│  │   }                               │  │
│  │ }                                 │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 3: Process Documents         │  │
│  │                                   │  │
│  │ For each transformed document:    │  │
│  │                                   │  │
│  │ 1. Check for duplicates           │  │
│  │    (content_hash)                 │  │
│  │                                   │  │
│  │ 2. Generate embedding             │  │
│  │                                   │  │
│  │ 3. Create Document record         │  │
│  │                                   │  │
│  │ 4. Chunk content                  │  │
│  │                                   │  │
│  │ 5. Generate chunk embeddings      │  │
│  │                                   │  │
│  │ 6. Create Chunk records           │  │
│  └───────────┬───────────────────────┘  │
│              │                           │
│              ▼                           │
│  ┌───────────────────────────────────┐  │
│  │ Step 4: Update Connector          │  │
│  │                                   │  │
│  │ - Set last_indexed_at = now()     │  │
│  │ - Log success/failure             │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Data Now Searchable                    │
│  - Appears in document lists            │
│  - Included in search results           │
│  - Available for chat/research          │
└─────────────────────────────────────────┘
```

### Incremental Sync

- Only fetch new/updated items since last_indexed_at
- Prevents redundant processing
- Scheduled periodic syncs (configurable)

### Error Handling

- API failures: Retry with exponential backoff
- Invalid credentials: Notify user, pause syncing
- Rate limits: Queue for later, respect limits
- Partial failures: Log which items failed, continue with others

## Authentication Flow

### Registration

```
User → Register Form → POST /auth/register
                              │
                              ▼
                       Create User Record
                              │
                              ▼
                       Send Verification Email (if configured)
                              │
                              ▼
                       Return User Data
                              │
                              ▼
                       Auto-login or Redirect to Login
```

### Login (JWT)

```
User → Login Form → POST /auth/jwt/login
                          │
                          ▼
                   Verify Credentials
                          │
                          ▼
                   Generate JWT Token
                          │
                          ▼
                   Return {access_token, token_type}
                          │
                          ▼
Frontend Stores Token in localStorage
                          │
                          ▼
All Subsequent Requests Include:
Authorization: Bearer {token}
```

### OAuth (Google)

```
User → Click "Login with Google"
         │
         ▼
Redirect to /auth/google/authorize
         │
         ▼
Google OAuth Consent Screen
         │
         ▼
User Approves
         │
         ▼
Google → /auth/google/callback?code=...
         │
         ▼
Backend Exchanges Code for User Info
         │
         ▼
Create/Update User Record
         │
         ▼
Generate JWT Token
         │
         ▼
Redirect to Frontend with Token
         │
         ▼
Frontend Stores Token
         │
         ▼
User Authenticated
```

## Summary

These workflows demonstrate how SurfSense components work together to provide:

1. **Document Processing**: Async, fault-tolerant, multi-format support
2. **Advanced Search**: Hybrid approach for best results
3. **AI Research**: Streaming, citation-aware, multi-mode
4. **Podcast Generation**: Fast, high-quality, customizable
5. **Data Integration**: Automatic syncing from 15+ sources
6. **Security**: JWT/OAuth, ownership validation, isolated data

Each workflow is designed for:
- **Performance**: Async operations, parallel processing
- **Reliability**: Error handling, retries, logging
- **Scalability**: Background tasks, efficient database queries
- **User Experience**: Real-time feedback, progress tracking

For more details on specific components, see:
- [Architecture Documentation](ARCHITECTURE.md)
- [Backend Documentation](backend/README.md)
- [Frontend Documentation](frontend/README.md)
- [API Reference](API_REFERENCE.md)
