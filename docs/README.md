# SurfSense Documentation

Welcome to the comprehensive documentation for SurfSense - your personal AI research assistant and knowledge management platform.

## 📚 Documentation Index

### For First-Time Users

**[Getting Started Guide](GETTING_STARTED.md)** - Start here!
- What is SurfSense and how does it work?
- Core concepts explained for beginners
- Step-by-step tutorials
- Common use cases
- Tips and best practices
- FAQ and troubleshooting

### Understanding the System

**[Architecture Documentation](ARCHITECTURE.md)** - Deep dive into the system
- System architecture overview
- Core components explained
- Technology stack details
- Data flow diagrams
- Key concepts (Search Spaces, Hybrid Search, RAG, Embeddings)

**[Workflows and Data Flow](WORKFLOWS.md)** - See how it all works together
- Document upload and processing flow
- Search and retrieval workflow
- AI chat and research workflow
- Podcast generation workflow
- Connector synchronization workflow
- Authentication flow

### For Developers

**[API Reference](API_REFERENCE.md)** - Complete API documentation
- Authentication endpoints
- Search Spaces API
- Documents API
- Chats API
- Podcasts API
- Connectors API
- Code examples in Python, JavaScript, and cURL

**[Backend Documentation](backend/README.md)** - Backend deep dive
- FastAPI application structure
- Database models and schemas
- AI agents (LangGraph)
- Services and utilities
- Background tasks
- Retrieval system
- Configuration and deployment

**[Frontend Documentation](frontend/README.md)** - Frontend deep dive
- Next.js App Router structure
- React components and hooks
- UI/UX implementation
- State management
- API integration
- Styling with Tailwind CSS

## 🚀 Quick Links

### Installation and Setup

- **[Docker Installation](../DOCKER_SETUP.md)** - Easiest way to get started
- **[Manual Installation](../README.md#how-to-get-started)** - For advanced users
- **[Deployment Guide](../DEPLOYMENT_GUIDE.md)** - Production deployment

### Contributing

- **[Contributing Guidelines](../CONTRIBUTING.md)** - How to contribute
- **[Code of Conduct](../CODE_OF_CONDUCT.md)** - Community guidelines
- **[Pre-commit Hooks](../PRE_COMMIT.md)** - Code quality tools

## 📖 Documentation Structure

```
docs/
├── README.md                    # This file - documentation index
├── GETTING_STARTED.md           # Beginner's guide
├── ARCHITECTURE.md              # System architecture
├── WORKFLOWS.md                 # Data flows and workflows
├── API_REFERENCE.md             # Complete API documentation
├── backend/
│   └── README.md               # Backend documentation
├── frontend/
│   └── README.md               # Frontend documentation
└── diagrams/                    # Architecture diagrams (future)
```

## 🎯 What Should I Read?

### I'm brand new to SurfSense
→ Start with **[Getting Started Guide](GETTING_STARTED.md)**

### I want to understand how it works
→ Read **[Architecture Documentation](ARCHITECTURE.md)** and **[Workflows](WORKFLOWS.md)**

### I want to build integrations
→ Check **[API Reference](API_REFERENCE.md)**

### I want to contribute to the backend
→ Read **[Backend Documentation](backend/README.md)**

### I want to contribute to the frontend
→ Read **[Frontend Documentation](frontend/README.md)**

### I need to deploy SurfSense
→ See **[Deployment Guide](../DEPLOYMENT_GUIDE.md)** and **[Docker Setup](../DOCKER_SETUP.md)**

## 🔑 Key Concepts at a Glance

### Search Spaces
Isolated knowledge environments where you organize documents by topic, project, or context. Like folders, but smarter.

### Hybrid Search
Combines semantic (meaning-based) search with full-text (keyword) search for optimal results. Uses Reciprocal Rank Fusion (RRF) to merge results.

### RAG (Retrieval Augmented Generation)
AI generates answers based on YOUR documents, not just its training data. Reduces hallucinations and provides citations.

### Embeddings
Mathematical representations of text that capture semantic meaning. Enables similarity search and semantic understanding.

### Two-Tier Retrieval
Search at both document level (broad context) and chunk level (precise facts) for flexibility.

### AI Agents
LangGraph-based agents that orchestrate complex workflows:
- **Researcher Agent**: Generates comprehensive reports with citations
- **QNA Agent**: Quick question-answering
- **Podcaster Agent**: Converts conversations to audio

### Connectors
Integrations with external services (Slack, Notion, GitHub, etc.) that automatically sync data into SurfSense.

## 🛠️ Technology Stack Summary

### Backend
- **Framework**: FastAPI (Python 3.12+)
- **Database**: PostgreSQL + pgvector
- **AI**: LangChain, LangGraph, LiteLLM (100+ models)
- **Search**: Hybrid search with FlashRank reranking
- **Embeddings**: Chonkie (6000+ models supported)

### Frontend
- **Framework**: Next.js 15.2.3 (App Router)
- **Language**: TypeScript
- **UI**: React 19, Tailwind CSS 4.x, Shadcn/ui
- **Streaming**: Vercel AI SDK

### Infrastructure
- **Containerization**: Docker & Docker Compose
- **Database Admin**: pgAdmin
- **Server**: Uvicorn (ASGI)

## 📊 Feature Highlights

- ✅ **50+ File Formats**: PDF, DOCX, images, videos, and more
- ✅ **Advanced RAG**: Hybrid search + reranking
- ✅ **100+ LLM Models**: Via LiteLLM
- ✅ **6000+ Embedding Models**: Via Chonkie
- ✅ **15+ Connectors**: Slack, Notion, GitHub, Jira, and more
- ✅ **Podcast Generation**: 3-min podcast in ~20 seconds
- ✅ **Self-Hosted**: Complete data ownership
- ✅ **Privacy-Focused**: Local LLM support (Ollama)
- ✅ **Citations**: Always know the source
- ✅ **Streaming Responses**: Real-time feedback

## 🤝 Community

- **Discord**: [Join our community](https://discord.gg/ejRNvftDp9)
- **GitHub**: [Star and contribute](https://github.com/yb235/SurfSense)
- **Roadmap**: [View planned features](https://github.com/users/MODSetter/projects/2)

## 📝 License

SurfSense is licensed under the Apache License 2.0. See [LICENSE](../LICENSE) for details.

## 🙏 Acknowledgments

SurfSense builds on the shoulders of giants:
- FastAPI, LangChain, LangGraph
- Next.js, React, Tailwind CSS
- PostgreSQL, pgvector
- And many more open-source projects

---

**Ready to get started?** 

Begin with the **[Getting Started Guide](GETTING_STARTED.md)** and explore from there!

For questions or issues:
- Check the [FAQ in Getting Started](GETTING_STARTED.md#faq)
- Visit our [Discord community](https://discord.gg/ejRNvftDp9)
- Open an issue on [GitHub](https://github.com/yb235/SurfSense/issues)

Happy researching! 🚀
