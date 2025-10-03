# SurfSense API Reference

## Table of Contents
- [Authentication](#authentication)
- [Search Spaces](#search-spaces)
- [Documents](#documents)
- [Chats](#chats)
- [Podcasts](#podcasts)
- [Connectors](#connectors)
- [Logs](#logs)
- [Common Patterns](#common-patterns)

## Base URL

```
Local Development: http://localhost:8000
Production: Your deployed backend URL
```

All API endpoints are prefixed with `/api/v1` except authentication endpoints.

## Authentication

### JWT Authentication

**Register**
```http
POST /auth/register
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "secure_password"
}
```

**Response**
```json
{
  "id": "uuid-string",
  "email": "user@example.com",
  "is_active": true,
  "is_superuser": false,
  "is_verified": false
}
```

**Login**
```http
POST /auth/jwt/login
Content-Type: application/x-www-form-urlencoded

username=user@example.com&password=secure_password
```

**Response**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "token_type": "bearer"
}
```

**Verify Token**
```http
GET /verify-token
Authorization: Bearer {access_token}
```

**Response**
```json
{
  "message": "Token is valid"
}
```

### OAuth (Google)

**Authorize**
```http
GET /auth/google/authorize
```
Redirects to Google OAuth consent screen.

**Callback** (handled automatically by FastAPI Users)
```http
GET /auth/google/callback?code={auth_code}&state={state}
```

## Search Spaces

Search Spaces are isolated knowledge environments where users organize their documents.

### Create Search Space

```http
POST /api/v1/searchspaces/
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "name": "My Work Projects",
  "description": "Documents related to work projects"
}
```

**Response**
```json
{
  "id": 1,
  "name": "My Work Projects",
  "description": "Documents related to work projects",
  "user_id": "uuid-string",
  "created_at": "2024-01-01T00:00:00Z"
}
```

### List Search Spaces

```http
GET /api/v1/searchspaces/?skip=0&limit=200
Authorization: Bearer {access_token}
```

**Response**
```json
[
  {
    "id": 1,
    "name": "My Work Projects",
    "description": "Documents related to work projects",
    "user_id": "uuid-string",
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

### Get Search Space

```http
GET /api/v1/searchspaces/{search_space_id}
Authorization: Bearer {access_token}
```

### Update Search Space

```http
PUT /api/v1/searchspaces/{search_space_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "name": "Updated Name",
  "description": "Updated description"
}
```

### Delete Search Space

```http
DELETE /api/v1/searchspaces/{search_space_id}
Authorization: Bearer {access_token}
```

**Note**: Deleting a search space cascades to all associated documents, chunks, chats, and podcasts.

## Documents

Documents are the core content units in SurfSense. They can come from various sources.

### Document Types

```typescript
type DocumentType = 
  | "EXTENSION"              // From browser extension
  | "CRAWLED_URL"            // From web crawler
  | "FILE"                   // Uploaded file
  | "SLACK_CONNECTOR"        // From Slack
  | "NOTION_CONNECTOR"       // From Notion
  | "YOUTUBE_VIDEO"          // From YouTube
  | "GITHUB_CONNECTOR"       // From GitHub
  | "LINEAR_CONNECTOR"       // From Linear
  | "DISCORD_CONNECTOR"      // From Discord
  | "JIRA_CONNECTOR"         // From Jira
  | "CONFLUENCE_CONNECTOR"   // From Confluence
  | "CLICKUP_CONNECTOR"      // From ClickUp
  | "GOOGLE_CALENDAR_CONNECTOR"
  | "GOOGLE_GMAIL_CONNECTOR"
  | "AIRTABLE_CONNECTOR"
  | "LUMA_CONNECTOR"
```

### Upload Files

```http
POST /api/v1/documents/fileupload
Authorization: Bearer {access_token}
Content-Type: multipart/form-data

files: [File, File, ...]
search_space_id: 1
```

**Response**
```json
{
  "message": "Files uploaded for processing"
}
```

**Processing**: Files are processed asynchronously. Check logs for status.

**Supported Formats**: 50+ formats including:
- Documents: PDF, DOCX, DOC, ODT, RTF, TXT, MD, HTML
- Spreadsheets: XLSX, XLS, CSV
- Presentations: PPTX, PPT
- Images: JPG, PNG, TIFF, BMP
- Audio/Video: MP3, MP4, WAV, M4A

### Create Documents (Programmatic)

```http
POST /api/v1/documents/
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "search_space_id": 1,
  "document_type": "CRAWLED_URL",
  "content": ["https://example.com/article"]
}
```

**Document Types**:
- `EXTENSION`: Array of extension-saved pages
- `CRAWLED_URL`: Array of URLs to crawl
- `YOUTUBE_VIDEO`: Array of YouTube URLs

### List Documents

```http
GET /api/v1/documents/?search_space_id=1&page=1&page_size=50
Authorization: Bearer {access_token}
```

**Query Parameters**:
- `search_space_id` (required): Filter by search space
- `page` (optional): Page number (default: 1)
- `page_size` (optional): Items per page (default: 50, max: 100)

**Response**
```json
{
  "items": [
    {
      "id": 123,
      "title": "Document Title",
      "document_type": "FILE",
      "document_metadata": {
        "filename": "report.pdf",
        "file_size": 1024000
      },
      "content": "Document content...",
      "created_at": "2024-01-01T00:00:00Z",
      "search_space_id": 1
    }
  ],
  "total": 150
}
```

### Search Documents by Title

```http
GET /api/v1/documents/search/?search_space_id=1&title=project
Authorization: Bearer {access_token}
```

**Query Parameters**:
- `search_space_id` (required)
- `title` (required): Search term for document titles
- `page`, `page_size`: Pagination

### Get Document by ID

```http
GET /api/v1/documents/{document_id}
Authorization: Bearer {access_token}
```

**Response**
```json
{
  "id": 123,
  "title": "Document Title",
  "document_type": "FILE",
  "document_metadata": {...},
  "content": "Full document content...",
  "created_at": "2024-01-01T00:00:00Z",
  "search_space_id": 1
}
```

### Get Document by Chunk ID

```http
GET /api/v1/documents/by-chunk/{chunk_id}
Authorization: Bearer {access_token}
```

**Response**
```json
{
  "document": {
    "id": 123,
    "title": "Document Title",
    ...
  },
  "chunks": [
    {
      "id": 456,
      "content": "Chunk content...",
      "document_id": 123,
      "created_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

### Delete Document

```http
DELETE /api/v1/documents/{document_id}
Authorization: Bearer {access_token}
```

**Note**: Deleting a document also deletes all associated chunks.

## Chats

Chats enable conversational AI interactions with your knowledge base.

### Chat Types

```typescript
type ChatType = 
  | "QNA"              // Quick question-answering
  | "REPORT_GENERAL"   // Standard research report
  | "REPORT_DEEP"      // In-depth analysis
  | "REPORT_DEEPER"    // Most comprehensive research
```

### Stream Chat (Real-time)

```http
POST /chat
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "messages": [
    {
      "role": "user",
      "content": "What are the main findings in the Q4 report?"
    }
  ],
  "data": {
    "search_space_id": 1,
    "research_mode": "QNA",
    "search_mode": "CHUNKS",
    "selected_connectors": ["NOTION_CONNECTOR"],
    "document_ids_to_add_in_context": [123, 456]
  }
}
```

**Request Fields**:
- `messages`: Array of conversation messages
  - `role`: "user" or "assistant"
  - `content`: Message text
- `data`:
  - `search_space_id`: Search space to query
  - `research_mode`: "QNA" | "REPORT_GENERAL" | "REPORT_DEEP" | "REPORT_DEEPER"
  - `search_mode`: "CHUNKS" | "DOCUMENTS" (retrieval granularity)
  - `selected_connectors`: Array of connector types to search
  - `document_ids_to_add_in_context`: Specific docs to include in context

**Response**: Server-Sent Events (SSE) stream

**Stream Events**:
```
data: {"type": "reformulated_query", "query": "..."}

data: {"type": "section_title", "content": "Introduction"}

data: {"type": "section_content", "content": "Based on the documents..."}

data: {"type": "citation", "number": 1, "title": "Q4 Report", "document_id": 123}

data: {"type": "followup_questions", "questions": ["...", "..."]}

data: [DONE]
```

### Create Chat (Save Conversation)

```http
POST /api/v1/chats/
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "search_space_id": 1,
  "type": "QNA",
  "title": "Q4 Report Discussion",
  "initial_connectors": ["NOTION_CONNECTOR"],
  "messages": [
    {
      "role": "user",
      "content": "What are the main findings?"
    },
    {
      "role": "assistant",
      "content": "Based on the Q4 report..."
    }
  ]
}
```

### List Chats

```http
GET /api/v1/chats/?search_space_id=1&skip=0&limit=50
Authorization: Bearer {access_token}
```

**Response**
```json
[
  {
    "id": 789,
    "search_space_id": 1,
    "type": "QNA",
    "title": "Q4 Report Discussion",
    "created_at": "2024-01-01T00:00:00Z"
  }
]
```

### Get Chat

```http
GET /api/v1/chats/{chat_id}
Authorization: Bearer {access_token}
```

**Response**
```json
{
  "id": 789,
  "search_space_id": 1,
  "type": "QNA",
  "title": "Q4 Report Discussion",
  "initial_connectors": ["NOTION_CONNECTOR"],
  "messages": [...],
  "created_at": "2024-01-01T00:00:00Z"
}
```

### Update Chat

```http
PUT /api/v1/chats/{chat_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "title": "Updated Title",
  "messages": [...]
}
```

### Delete Chat

```http
DELETE /api/v1/chats/{chat_id}
Authorization: Bearer {access_token}
```

## Podcasts

Generate audio podcasts from chat conversations.

### Create Podcast

```http
POST /api/v1/podcasts/
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "search_space_id": 1,
  "title": "Q4 Report Discussion Podcast",
  "chat_id": 789
}
```

**Response**
```json
{
  "id": 111,
  "search_space_id": 1,
  "title": "Q4 Report Discussion Podcast",
  "podcast_transcript": {
    "speakers": ["Alex", "Sam"],
    "dialogue": [
      {"speaker": "Alex", "text": "Welcome to..."},
      {"speaker": "Sam", "text": "Thanks for having me..."}
    ]
  },
  "file_location": "/path/to/podcast.mp3",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**Processing Time**: ~20 seconds for a 3-minute podcast

### List Podcasts

```http
GET /api/v1/podcasts/?search_space_id=1&skip=0&limit=50
Authorization: Bearer {access_token}
```

### Get Podcast

```http
GET /api/v1/podcasts/{podcast_id}
Authorization: Bearer {access_token}
```

### Download Podcast

```http
GET /api/v1/podcasts/{podcast_id}/download
Authorization: Bearer {access_token}
```

**Response**: Binary audio file (MP3)

### Delete Podcast

```http
DELETE /api/v1/podcasts/{podcast_id}
Authorization: Bearer {access_token}
```

## Connectors

External data source integrations.

### Connector Types

```typescript
type ConnectorType = 
  | "TAVILY_API"           // Web search
  | "LINKUP_API"           // Web search
  | "SLACK_CONNECTOR"      // Indexable
  | "NOTION_CONNECTOR"     // Indexable
  | "GITHUB_CONNECTOR"     // Indexable
  | "LINEAR_CONNECTOR"     // Indexable
  | "DISCORD_CONNECTOR"    // Indexable
  | "JIRA_CONNECTOR"       // Indexable
  | "CONFLUENCE_CONNECTOR" // Indexable
  | "CLICKUP_CONNECTOR"    // Indexable
  | "GOOGLE_CALENDAR_CONNECTOR"
  | "GOOGLE_GMAIL_CONNECTOR"
  | "AIRTABLE_CONNECTOR"
  | "LUMA_CONNECTOR"
```

### Create Connector

**General Pattern**
```http
POST /api/v1/search-source-connectors/
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "name": "My Slack Workspace",
  "connector_type": "SLACK_CONNECTOR",
  "is_indexable": true,
  "config": {
    // Connector-specific configuration
  }
}
```

**Slack Example**
```json
{
  "name": "Work Slack",
  "connector_type": "SLACK_CONNECTOR",
  "is_indexable": true,
  "config": {
    "token": "xoxb-...",
    "channels": ["general", "engineering"]
  }
}
```

**Notion Example**
```json
{
  "name": "Personal Notion",
  "connector_type": "NOTION_CONNECTOR",
  "is_indexable": true,
  "config": {
    "token": "secret_...",
    "page_ids": ["page-id-1", "page-id-2"]
  }
}
```

**Tavily API Example**
```json
{
  "name": "Tavily Search",
  "connector_type": "TAVILY_API",
  "is_indexable": false,
  "config": {
    "api_key": "tvly-..."
  }
}
```

### List Connectors

```http
GET /api/v1/search-source-connectors/?skip=0&limit=50
Authorization: Bearer {access_token}
```

**Response**
```json
[
  {
    "id": 1,
    "name": "Work Slack",
    "connector_type": "SLACK_CONNECTOR",
    "is_indexable": true,
    "last_indexed_at": "2024-01-01T12:00:00Z",
    "config": {...},
    "created_at": "2024-01-01T00:00:00Z",
    "user_id": "uuid-string"
  }
]
```

### Get Connector

```http
GET /api/v1/search-source-connectors/{connector_id}
Authorization: Bearer {access_token}
```

### Update Connector

```http
PUT /api/v1/search-source-connectors/{connector_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "name": "Updated Name",
  "config": {...}
}
```

### Delete Connector

```http
DELETE /api/v1/search-source-connectors/{connector_id}
Authorization: Bearer {access_token}
```

### Trigger Indexing

For indexable connectors, manually trigger a sync:

```http
POST /api/v1/search-source-connectors/{connector_id}/index
Authorization: Bearer {access_token}
```

**Response**
```json
{
  "message": "Indexing started"
}
```

## Logs

Track background task execution.

### Log Levels

```typescript
type LogLevel = "DEBUG" | "INFO" | "WARNING" | "ERROR" | "CRITICAL"
```

### Log Status

```typescript
type LogStatus = "IN_PROGRESS" | "SUCCESS" | "FAILED"
```

### List Logs

```http
GET /api/v1/logs/?search_space_id=1&level=ERROR&status=FAILED&page=1&page_size=50
Authorization: Bearer {access_token}
```

**Query Parameters**:
- `search_space_id` (optional): Filter by search space
- `level` (optional): Filter by log level
- `status` (optional): Filter by status
- `page`, `page_size`: Pagination

**Response**
```json
{
  "items": [
    {
      "id": 1,
      "search_space_id": 1,
      "level": "ERROR",
      "status": "FAILED",
      "message": "Failed to process document",
      "metadata": {
        "document_id": 123,
        "error": "Unsupported file format"
      },
      "created_at": "2024-01-01T00:00:00Z"
    }
  ],
  "total": 5
}
```

## Common Patterns

### Pagination

Most list endpoints support pagination:

```
?page=1&page_size=50
```

Or offset-based:

```
?skip=0&limit=50
```

### Error Responses

All errors follow this format:

```json
{
  "detail": "Error message describing what went wrong"
}
```

Common HTTP status codes:
- `400`: Bad Request - Invalid input
- `401`: Unauthorized - Missing or invalid token
- `403`: Forbidden - No access to resource
- `404`: Not Found - Resource doesn't exist
- `422`: Unprocessable Entity - Validation error
- `500`: Internal Server Error - Server-side error

### Ownership Validation

All resources (search spaces, documents, chats, etc.) are validated to ensure the authenticated user has access.

Example error:
```json
{
  "detail": "You don't have access to this search space"
}
```

### Background Processing

Operations that take time (file upload, connector indexing) return immediately and process in the background:

1. API returns success response
2. Task is queued for background processing
3. Check logs for task status
4. Results appear when processing completes

### Streaming Responses

Chat endpoint uses Server-Sent Events (SSE):

```javascript
const response = await fetch('/chat', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer token',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({...})
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  
  const text = decoder.decode(value);
  const lines = text.split('\n');
  
  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = line.slice(6);
      if (data === '[DONE]') {
        // Stream complete
      } else {
        const event = JSON.parse(data);
        // Process event
      }
    }
  }
}
```

### Rate Limiting

Currently, there are no explicit rate limits, but this may change in production deployments. Follow best practices:
- Don't make excessive concurrent requests
- Use pagination for large datasets
- Cache responses when appropriate

### CORS

The backend allows all origins in development:

```python
allow_origins=["*"]
```

For production, configure specific allowed origins in your deployment.

## API Client Examples

### Python

```python
import requests

BASE_URL = "http://localhost:8000"

# Login
response = requests.post(
    f"{BASE_URL}/auth/jwt/login",
    data={
        "username": "user@example.com",
        "password": "password"
    }
)
token = response.json()["access_token"]

# List documents
response = requests.get(
    f"{BASE_URL}/api/v1/documents/",
    params={"search_space_id": 1},
    headers={"Authorization": f"Bearer {token}"}
)
documents = response.json()["items"]
```

### JavaScript/TypeScript

```typescript
const BASE_URL = "http://localhost:8000";

// Login
const loginResponse = await fetch(`${BASE_URL}/auth/jwt/login`, {
  method: "POST",
  headers: {
    "Content-Type": "application/x-www-form-urlencoded",
  },
  body: new URLSearchParams({
    username: "user@example.com",
    password: "password",
  }),
});
const { access_token } = await loginResponse.json();

// List documents
const docsResponse = await fetch(
  `${BASE_URL}/api/v1/documents/?search_space_id=1`,
  {
    headers: {
      Authorization: `Bearer ${access_token}`,
    },
  }
);
const { items: documents } = await docsResponse.json();
```

### cURL

```bash
# Login
curl -X POST http://localhost:8000/auth/jwt/login \
  -d "username=user@example.com&password=password"

# Save token
TOKEN="eyJ0eXAiOiJKV1QiLCJhbGc..."

# List documents
curl http://localhost:8000/api/v1/documents/?search_space_id=1 \
  -H "Authorization: Bearer $TOKEN"
```

## WebSocket Support

Currently, real-time updates use HTTP streaming (SSE). WebSocket support may be added in future versions for:
- Real-time log updates
- Document processing status
- Collaborative features

## API Versioning

The API is currently at version 1 (`/api/v1`). Future versions will:
- Maintain backward compatibility when possible
- Provide migration guides for breaking changes
- Support multiple versions simultaneously during transition periods

---

For more information, see:
- [Architecture Documentation](ARCHITECTURE.md)
- [Backend Documentation](backend/README.md)
- [Frontend Documentation](frontend/README.md)
