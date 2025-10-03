# Backend Documentation

## Overview

The SurfSense backend is a FastAPI-based application that provides a comprehensive API for knowledge management, AI-powered search, and document processing.

**Location**: `surfsense_backend/`

**Key Technologies**:
- Python 3.12+
- FastAPI (web framework)
- PostgreSQL + pgvector (database)
- SQLAlchemy (ORM)
- LangChain & LangGraph (AI workflows)
- LiteLLM (multi-provider LLM interface)

## Directory Structure

```
surfsense_backend/
├── app/
│   ├── agents/              # AI agents (LangGraph-based)
│   │   ├── podcaster/       # Podcast generation agent
│   │   └── researcher/      # Research report agent
│   ├── config/              # Configuration management
│   ├── connectors/          # External service integrations
│   ├── prompts/             # LLM system prompts
│   ├── retriver/            # Search and retrieval logic
│   ├── routes/              # API endpoint definitions
│   ├── schemas/             # Pydantic models for validation
│   ├── services/            # Reusable business logic
│   ├── tasks/               # Background task processing
│   │   ├── connector_indexers/
│   │   └── document_processors/
│   ├── utils/               # Utility functions
│   ├── app.py              # FastAPI application setup
│   ├── db.py               # Database models and setup
│   └── users.py            # Authentication configuration
├── alembic/                 # Database migrations
├── main.py                  # Application entry point
├── pyproject.toml          # Dependencies and project config
└── alembic.ini             # Alembic configuration
```

## Core Components

### 1. Application Setup (`app/app.py`)

The main FastAPI application with CORS, authentication routes, and API routers.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(lifespan=lifespan)

# CORS configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Authentication routes
app.include_router(fastapi_users.get_auth_router(auth_backend), prefix="/auth/jwt")
app.include_router(fastapi_users.get_register_router(UserRead, UserCreate), prefix="/auth")

# API routes
app.include_router(crud_router, prefix="/api/v1")
```

### 2. Database Models (`app/db.py`)

SQLAlchemy models with vector support for semantic search.

**Key Models**:

#### User
```python
class User(SQLAlchemyBaseUserTableUUID, Base):
    """User authentication and profile"""
    # Inherits: id, email, hashed_password, is_active, is_superuser, is_verified
    search_spaces = relationship("SearchSpace", back_populates="user")
```

#### SearchSpace
```python
class SearchSpace(BaseModel, TimestampMixin):
    """Isolated knowledge environment"""
    name = Column(String(100), nullable=False, index=True)
    description = Column(String(500), nullable=True)
    user_id = Column(UUID, ForeignKey("user.id", ondelete="CASCADE"))
    
    # Relationships
    documents = relationship("Document", cascade="all, delete-orphan")
    chats = relationship("Chat", cascade="all, delete-orphan")
    podcasts = relationship("Podcast", cascade="all, delete-orphan")
```

#### Document
```python
class Document(BaseModel, TimestampMixin):
    """Full document with vector embedding"""
    title = Column(String, nullable=False, index=True)
    document_type = Column(SQLAlchemyEnum(DocumentType), nullable=False)
    document_metadata = Column(JSON, nullable=True)
    content = Column(Text, nullable=False)
    content_hash = Column(String, nullable=False, unique=True)
    embedding = Column(Vector(dimension))  # pgvector
    
    search_space_id = Column(Integer, ForeignKey("searchspaces.id", ondelete="CASCADE"))
    chunks = relationship("Chunk", cascade="all, delete-orphan")
```

#### Chunk
```python
class Chunk(BaseModel, TimestampMixin):
    """Document fragment with vector embedding"""
    content = Column(Text, nullable=False)
    embedding = Column(Vector(dimension))  # pgvector
    document_id = Column(Integer, ForeignKey("documents.id", ondelete="CASCADE"))
```

#### Chat
```python
class Chat(BaseModel, TimestampMixin):
    """Conversation history"""
    type = Column(SQLAlchemyEnum(ChatType), nullable=False)
    title = Column(String, nullable=False)
    initial_connectors = Column(ARRAY(String), nullable=True)
    messages = Column(JSON, nullable=False)
    search_space_id = Column(Integer, ForeignKey("searchspaces.id", ondelete="CASCADE"))
```

### 3. API Routes (`app/routes/`)

FastAPI routers organized by domain.

#### Documents Routes (`documents_routes.py`)

**Key Endpoints**:
- `POST /documents/` - Create documents programmatically
- `POST /documents/fileupload` - Upload files for processing
- `GET /documents/` - List documents with pagination
- `GET /documents/search/` - Search documents by title
- `GET /documents/{document_id}` - Get single document
- `GET /documents/by-chunk/{chunk_id}` - Get document by chunk ID
- `DELETE /documents/{document_id}` - Delete document

**File Upload Flow**:
```python
@router.post("/documents/fileupload")
async def create_documents_file_upload(
    files: list[UploadFile],
    search_space_id: int = Form(...),
    session: AsyncSession = Depends(get_async_session),
    user: User = Depends(current_active_user),
    fastapi_background_tasks: BackgroundTasks = BackgroundTasks(),
):
    # 1. Validate ownership
    await check_ownership(session, SearchSpace, search_space_id, user)
    
    # 2. Save to temp location
    for file in files:
        with tempfile.NamedTemporaryFile(delete=False) as temp_file:
            temp_path = temp_file.name
        
        content = await file.read()
        with open(temp_path, "wb") as f:
            f.write(content)
        
        # 3. Queue background processing
        fastapi_background_tasks.add_task(
            process_file_in_background_with_new_session,
            temp_path,
            file.filename,
            search_space_id,
            str(user.id),
        )
```

#### Chats Routes (`chats_routes.py`)

**Key Endpoints**:
- `POST /chat` - Stream chat responses (SSE)
- `POST /chats/` - Create chat
- `GET /chats/` - List chats
- `GET /chats/{chat_id}` - Get chat
- `PUT /chats/{chat_id}` - Update chat
- `DELETE /chats/{chat_id}` - Delete chat

**Streaming Chat**:
```python
@router.post("/chat")
async def handle_chat_data(
    request: AISDKChatRequest,
    session: AsyncSession = Depends(get_async_session),
    user: User = Depends(current_active_user),
):
    # Extract parameters
    user_query = messages[-1]["content"]
    search_space_id = request.data.get("search_space_id")
    research_mode = request.data.get("research_mode")
    
    # Stream results
    response = StreamingResponse(
        stream_connector_search_results(
            user_query,
            user.id,
            search_space_id,
            session,
            research_mode,
            selected_connectors,
            langchain_chat_history,
            search_mode_str,
            document_ids_to_add_in_context,
        )
    )
    
    response.headers["x-vercel-ai-data-stream"] = "v1"
    return response
```

#### Search Spaces Routes (`search_spaces_routes.py`)

**Key Endpoints**:
- `POST /searchspaces/` - Create search space
- `GET /searchspaces/` - List user's search spaces
- `GET /searchspaces/{id}` - Get search space
- `PUT /searchspaces/{id}` - Update search space
- `DELETE /searchspaces/{id}` - Delete search space

### 4. AI Agents (`app/agents/`)

LangGraph-based agents for complex AI workflows.

#### Researcher Agent (`agents/researcher/`)

**Purpose**: Generate comprehensive research reports with citations.

**Architecture**: LangGraph state machine with conditional routing.

**Files**:
- `graph.py` - Workflow definition
- `state.py` - State management
- `nodes.py` - Individual processing nodes
- `configuration.py` - Agent configuration
- `prompts.py` - System prompts
- `utils.py` - Helper functions

**Workflow** (`graph.py`):
```python
def build_graph():
    workflow = StateGraph(State, config_schema=Configuration)
    
    # Add nodes
    workflow.add_node("reformulate_user_query", reformulate_user_query)
    workflow.add_node("handle_qna_workflow", handle_qna_workflow)
    workflow.add_node("write_answer_outline", write_answer_outline)
    workflow.add_node("process_sections", process_sections)
    workflow.add_node("generate_further_questions", generate_further_questions)
    
    # Conditional routing based on research mode
    workflow.add_conditional_edges(
        "reformulate_user_query",
        route_after_reformulate,
        {
            "handle_qna_workflow": "handle_qna_workflow",  # QNA mode
            "write_answer_outline": "write_answer_outline", # Report modes
        },
    )
    
    return workflow.compile()
```

**State** (`state.py`):
```python
class State(TypedDict):
    # Input
    user_query: str
    search_space_id: int
    
    # Intermediate
    reformulated_query: str
    answer_outline: dict
    sections: list[dict]
    reranked_documents: list[Document]
    
    # Output
    final_written_report: str
    followup_questions: list[str]
```

**Key Nodes** (`nodes.py`):

1. **reformulate_user_query**: Clarify and expand user's question
```python
async def reformulate_user_query(state: State, config):
    """Expand user query for better retrieval"""
    query = state["user_query"]
    llm = get_llm(config)
    
    prompt = f"Reformulate this query for better search: {query}"
    response = await llm.ainvoke(prompt)
    
    return {"reformulated_query": response.content}
```

2. **write_answer_outline**: Create research structure
```python
async def write_answer_outline(state: State, config):
    """Generate report outline with sections"""
    query = state["reformulated_query"]
    
    # Retrieve relevant documents
    docs = await retrieve_documents(query, state["search_space_id"])
    
    # Generate outline
    outline = await generate_outline(query, docs, config)
    
    return {"answer_outline": outline}
```

3. **process_sections**: Generate content for each section
```python
async def process_sections(state: State, config):
    """Write content for each outline section in parallel"""
    outline = state["answer_outline"]
    
    # Use sub-agents to write sections in parallel
    tasks = [
        write_section(section, state, config)
        for section in outline["sections"]
    ]
    sections = await asyncio.gather(*tasks)
    
    return {"sections": sections}
```

4. **generate_further_questions**: Suggest follow-up questions
```python
async def generate_further_questions(state: State, config):
    """Generate relevant follow-up questions"""
    report = state["final_written_report"]
    
    questions = await llm.ainvoke(
        f"Based on this report, suggest 3 follow-up questions:\n{report}"
    )
    
    return {"followup_questions": questions}
```

#### Podcaster Agent (`agents/podcaster/`)

**Purpose**: Convert chat conversations into engaging audio podcasts.

**Workflow**:
1. Extract conversation from chat
2. Generate podcast script (dialogue format)
3. Synthesize audio with TTS (multiple speakers)
4. Save audio file and transcript

**TTS Providers**:
- OpenAI TTS
- Kokoro (local, fast)
- Azure Speech Services
- Google Vertex AI

### 5. Services (`app/services/`)

Reusable business logic extracted from routes and agents.

#### LLM Service (`llm_service.py`)

**Purpose**: Unified interface to multiple LLM providers via LiteLLM.

```python
from litellm import acompletion

async def get_llm_response(
    messages: list[dict],
    model: str = "gpt-4o-mini",
    temperature: float = 0.7,
    stream: bool = False,
):
    """Get response from any LLM provider"""
    response = await acompletion(
        model=model,
        messages=messages,
        temperature=temperature,
        stream=stream,
    )
    return response
```

**Supported Providers**:
- OpenAI, Anthropic, Google, Azure
- Groq, Together AI, Replicate
- Ollama (local models)
- 100+ total models

#### Reranker Service (`reranker_service.py`)

**Purpose**: Rerank search results for improved relevance.

```python
from rerankers import Reranker

class RerankerService:
    def __init__(self, model_name: str = "ms-marco-MiniLM-L-12-v2"):
        self.reranker = Reranker(model_name, model_type="flashrank")
    
    async def rerank(
        self,
        query: str,
        documents: list[str],
        top_k: int = 10,
    ) -> list[int]:
        """Rerank documents by relevance to query"""
        results = self.reranker.rank(
            query=query,
            docs=documents,
            top_k=top_k,
        )
        return [r.doc_id for r in results]
```

#### Query Service (`query_service.py`)

**Purpose**: Handle search queries with hybrid search.

```python
class QueryService:
    async def hybrid_search(
        self,
        query: str,
        search_space_id: int,
        mode: str = "CHUNKS",
        limit: int = 10,
    ) -> list[Document | Chunk]:
        """Perform hybrid search combining vector + full-text"""
        
        # 1. Vector search
        vector_results = await self._vector_search(query, search_space_id, mode)
        
        # 2. Full-text search
        fulltext_results = await self._fulltext_search(query, search_space_id, mode)
        
        # 3. Merge with RRF (Reciprocal Rank Fusion)
        merged = self._reciprocal_rank_fusion(vector_results, fulltext_results)
        
        # 4. Rerank for final ordering
        reranked = await self.reranker.rerank(query, merged, top_k=limit)
        
        return reranked
```

#### Connector Service (`connector_service.py`)

**Purpose**: Interface with external APIs (Slack, Notion, GitHub, etc.).

```python
class ConnectorService:
    async def fetch_slack_messages(
        self,
        token: str,
        channels: list[str],
        since: datetime,
    ) -> list[dict]:
        """Fetch messages from Slack channels"""
        # Implementation
        
    async def fetch_notion_pages(
        self,
        token: str,
        page_ids: list[str],
    ) -> list[dict]:
        """Fetch pages from Notion"""
        # Implementation
```

### 6. Background Tasks (`app/tasks/`)

Asynchronous processing for long-running operations.

#### Document Processors (`tasks/document_processors/`)

**Purpose**: Process uploaded files and extract content.

**Workflow**:
```python
async def process_file_in_background_with_new_session(
    file_path: str,
    filename: str,
    search_space_id: int,
    user_id: str,
):
    """Process uploaded file"""
    
    # 1. Parse file (Docling/Unstructured/LlamaCloud)
    content = await parse_file(file_path, filename)
    
    # 2. Generate document embedding
    embedding = await generate_embedding(content)
    
    # 3. Create document record
    document = await create_document(
        title=filename,
        content=content,
        embedding=embedding,
        search_space_id=search_space_id,
    )
    
    # 4. Chunk document
    chunks = await chunk_document(content)
    
    # 5. Generate chunk embeddings
    for chunk in chunks:
        chunk_embedding = await generate_embedding(chunk.content)
        await create_chunk(
            content=chunk.content,
            embedding=chunk_embedding,
            document_id=document.id,
        )
```

**Supported Parsers**:
- **Docling**: PDF, DOCX, images (local, free)
- **Unstructured.io**: 34+ formats (API key required)
- **LlamaCloud**: 50+ formats (API key required)

#### Connector Indexers (`tasks/connector_indexers/`)

**Purpose**: Sync data from external sources.

```python
async def index_slack_connector(
    connector_id: int,
    user_id: str,
):
    """Index messages from Slack"""
    
    # 1. Get connector config
    connector = await get_connector(connector_id)
    
    # 2. Fetch messages
    messages = await fetch_slack_messages(
        token=connector.config["token"],
        channels=connector.config["channels"],
        since=connector.last_indexed_at,
    )
    
    # 3. Process each message
    for message in messages:
        # Create document from message
        await process_connector_document(
            content=message["text"],
            metadata=message,
            connector_type="SLACK_CONNECTOR",
            search_space_id=connector.search_space_id,
        )
    
    # 4. Update last indexed time
    await update_connector_indexed_at(connector_id)
```

### 7. Retrieval System (`app/retriver/`)

Advanced RAG implementation with hybrid search.

#### Hybrid Search (`chunks_hybrid_search.py`, `documents_hybrid_search.py`)

**Purpose**: Combine vector and full-text search for optimal retrieval.

```python
class ChucksHybridSearchRetriever:
    """Hybrid search at chunk level"""
    
    async def retrieve(
        self,
        query: str,
        search_space_id: int,
        limit: int = 10,
    ) -> list[Chunk]:
        """
        Hybrid search combining:
        1. Vector similarity (pgvector)
        2. Full-text search (PostgreSQL tsvector)
        3. RRF merge
        4. Reranking
        """
        
        # Generate query embedding
        query_embedding = await self.embed_query(query)
        
        # Vector search
        vector_results = await self.session.execute(
            select(Chunk)
            .join(Document)
            .where(Document.search_space_id == search_space_id)
            .order_by(Chunk.embedding.cosine_distance(query_embedding))
            .limit(limit * 2)
        )
        
        # Full-text search
        fulltext_results = await self.session.execute(
            select(Chunk)
            .join(Document)
            .where(Document.search_space_id == search_space_id)
            .where(Chunk.content.match(query))
            .limit(limit * 2)
        )
        
        # RRF merge
        merged = reciprocal_rank_fusion(
            vector_results.scalars().all(),
            fulltext_results.scalars().all(),
        )
        
        # Rerank
        reranked = await self.reranker.rerank(
            query,
            [chunk.content for chunk in merged],
            top_k=limit,
        )
        
        return [merged[idx] for idx in reranked]
```

**Reciprocal Rank Fusion (RRF)**:
```python
def reciprocal_rank_fusion(
    results1: list,
    results2: list,
    k: int = 60,
) -> list:
    """Merge two ranked lists using RRF"""
    scores = {}
    
    for rank, item in enumerate(results1):
        scores[item] = scores.get(item, 0) + 1 / (rank + k)
    
    for rank, item in enumerate(results2):
        scores[item] = scores.get(item, 0) + 1 / (rank + k)
    
    return sorted(scores.keys(), key=lambda x: scores[x], reverse=True)
```

## Configuration

### Environment Variables

Key environment variables (`.env` file):

```bash
# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost/surfsense

# Authentication
SECRET=your-secret-key-here
AUTH_TYPE=GOOGLE  # or JWT

# LLM Configuration
FAST_LLM_MODEL=gpt-4o-mini
SMART_LLM_MODEL=gpt-4o-mini
STRATEGIC_LLM_MODEL=gpt-4o-mini
LONG_CONTEXT_LLM_MODEL=gemini-2.0-flash-thinking

# Embedding Model
EMBEDDING_MODEL=text-embedding-ada-002
EMBEDDING_DIM=1536

# ETL Service (choose one)
UNSTRUCTURED_API_KEY=your-key
LLAMA_CLOUD_API_KEY=your-key
# or use Docling (no key needed)

# External APIs
OPENAI_API_KEY=your-key
TAVILY_API_KEY=your-key
FIRECRAWL_API_KEY=your-key

# Optional
LANGSMITH_API_KEY=your-key  # For tracing
```

### Config Module (`app/config/`)

Centralized configuration management:

```python
from pydantic_settings import BaseSettings

class Config(BaseSettings):
    DATABASE_URL: str
    SECRET: str
    AUTH_TYPE: str = "JWT"
    
    # LLM Models
    FAST_LLM_MODEL: str = "gpt-4o-mini"
    SMART_LLM_MODEL: str = "gpt-4o-mini"
    
    # Embeddings
    EMBEDDING_MODEL: str = "text-embedding-ada-002"
    EMBEDDING_DIM: int = 1536
    
    class Config:
        env_file = ".env"

config = Config()
```

## Database Migrations

SurfSense uses Alembic for database migrations.

**Create migration**:
```bash
alembic revision --autogenerate -m "description"
```

**Run migrations**:
```bash
alembic upgrade head
```

**Rollback**:
```bash
alembic downgrade -1
```

## Testing

Currently, SurfSense doesn't have extensive test coverage. This is a great area for contribution!

**Recommended test structure**:
```
tests/
├── test_routes/
│   ├── test_documents.py
│   ├── test_chats.py
│   └── test_search_spaces.py
├── test_services/
│   ├── test_llm_service.py
│   └── test_query_service.py
└── test_agents/
    └── test_researcher.py
```

## Performance Considerations

### Database Optimization

1. **Indexes**: All foreign keys and frequently queried fields are indexed
2. **Vector indexes**: Consider using IVFFlat or HNSW for large datasets
3. **Connection pooling**: Configured in SQLAlchemy

### Async Operations

All I/O operations use async/await:
- Database queries (asyncpg)
- HTTP requests (aiohttp)
- LLM calls (async clients)

### Caching

Consider adding caching for:
- Embeddings (if same text is embedded multiple times)
- Search results (short TTL)
- External API responses

### Background Tasks

Long-running operations use FastAPI BackgroundTasks:
- Document processing
- Connector indexing
- Podcast generation

## Security

### Authentication

- JWT tokens with expiration
- OAuth 2.0 support (Google)
- Password hashing with bcrypt
- FastAPI Users handles all auth logic

### Authorization

All resources are protected with ownership checks:

```python
async def check_ownership(
    session: AsyncSession,
    model: Type[BaseModel],
    resource_id: int,
    user: User,
):
    """Ensure user owns the resource"""
    result = await session.execute(
        select(model).where(
            model.id == resource_id,
            model.user_id == user.id,
        )
    )
    resource = result.scalar_one_or_none()
    
    if not resource:
        raise HTTPException(status_code=404, detail="Resource not found")
    
    return resource
```

### Data Protection

- All user data isolated by user_id
- CASCADE deletes prevent orphaned data
- SQL injection protection via SQLAlchemy ORM
- CORS configurable for production

## Deployment

See [DEPLOYMENT_GUIDE.md](../DEPLOYMENT_GUIDE.md) for detailed instructions.

**Quick start**:

1. **Set environment variables**
2. **Run migrations**: `alembic upgrade head`
3. **Start server**: `python main.py`

**Production recommendations**:
- Use proper secret management
- Configure CORS for specific origins
- Set up reverse proxy (nginx)
- Use managed PostgreSQL
- Monitor with LangSmith/Sentry
- Set up regular database backups

## Extending the Backend

### Adding a New Connector

1. **Create connector config schema** (`app/schemas/connectors/`)
2. **Implement fetcher** (`app/connectors/`)
3. **Create indexer** (`app/tasks/connector_indexers/`)
4. **Add route** (`app/routes/`)
5. **Update DocumentType enum**

### Adding a New LLM Provider

LiteLLM handles most providers automatically. Just set environment variables:

```bash
# For Anthropic
ANTHROPIC_API_KEY=your-key

# For Azure
AZURE_API_KEY=your-key
AZURE_API_BASE=https://your-instance.openai.azure.com
AZURE_API_VERSION=2023-05-15
```

### Customizing Prompts

Prompts are in `app/prompts/`. Edit to customize AI behavior:

```python
RESEARCHER_SYSTEM_PROMPT = """
You are an expert researcher...
[Customize this]
"""
```

## Troubleshooting

### Common Issues

**Issue**: Database connection errors
**Solution**: Check DATABASE_URL, ensure PostgreSQL is running with pgvector extension

**Issue**: Embedding dimension mismatch
**Solution**: Ensure EMBEDDING_DIM matches your embedding model's actual dimension

**Issue**: LLM rate limits
**Solution**: Implement retry logic with exponential backoff, or use different LLM tiers

**Issue**: Slow document processing
**Solution**: Check ETL service (Docling is local and faster than cloud services)

## Further Reading

- [Architecture Documentation](../ARCHITECTURE.md)
- [API Reference](../API_REFERENCE.md)
- [Frontend Documentation](../frontend/README.md)
- [LangChain Documentation](https://python.langchain.com/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
