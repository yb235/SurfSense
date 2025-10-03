# SurfSense Beginner's Guide

Welcome to SurfSense! This guide will help you understand what SurfSense is, how it works, and how to use it effectively.

## Table of Contents
- [What is SurfSense?](#what-is-surfsense)
- [Key Features for Beginners](#key-features-for-beginners)
- [Core Concepts Explained](#core-concepts-explained)
- [Getting Started](#getting-started)
- [Common Use Cases](#common-use-cases)
- [Step-by-Step Tutorials](#step-by-step-tutorials)
- [Tips and Best Practices](#tips-and-best-practices)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)

## What is SurfSense?

SurfSense is your **personal AI research assistant** that helps you:
- 📚 **Save and organize** information from multiple sources
- 🔍 **Search** through your saved content instantly
- 💬 **Chat** with your documents to get answers
- 📊 **Generate reports** with proper citations
- 🎙️ **Create podcasts** from your conversations

Think of it as having your own private version of tools like NotebookLM or Perplexity, but:
- ✅ **Self-hosted**: You control your data
- ✅ **Integrated**: Connects to Slack, Notion, GitHub, and more
- ✅ **Customizable**: Use any AI model you prefer
- ✅ **Privacy-focused**: Works with local AI models

## Key Features for Beginners

### 1. Search Spaces (Your Knowledge Folders)

Search Spaces are like folders for organizing your knowledge. Examples:
- 📁 "Work Projects" - All work-related documents
- 📁 "Personal Learning" - Articles and tutorials you're studying
- 📁 "Research Papers" - Academic papers for a thesis

**Why use them?**
- Keep different topics separate
- Search within specific contexts
- Better organization

### 2. Document Upload

Upload almost any file type:
- 📄 PDFs, Word documents, PowerPoint presentations
- 📊 Excel spreadsheets, CSV files
- 🖼️ Images (SurfSense can extract text from images!)
- 🎥 Videos and 🎵 Audio files (transcribed automatically)

**Supports 50+ file formats!**

### 3. AI Chat

Ask questions about your documents in natural language:
- "What are the main points in the Q4 report?"
- "Summarize all meeting notes from last week"
- "Find information about the new product launch"

**Get answers with citations** - know exactly where information came from!

### 4. Research Reports

Generate comprehensive reports with different depth levels:
- ⚡ **Quick (QNA)**: Fast answers to simple questions
- 📝 **General**: Standard research report
- 🔬 **Deep**: Detailed analysis
- 🎓 **Deeper**: Most comprehensive research

### 5. Podcast Generation

Turn your chat conversations into audio podcasts:
- 🎙️ Two AI speakers discuss your topic
- ⚡ Generated in ~20 seconds
- 📝 Includes transcript

Perfect for learning on the go!

### 6. External Integrations (Connectors)

Connect SurfSense to your existing tools:
- 💼 **Work**: Slack, Jira, Confluence, Linear
- 📝 **Notes**: Notion
- 👨‍💻 **Development**: GitHub
- 📧 **Email**: Gmail
- 📅 **Calendar**: Google Calendar
- And many more!

## Core Concepts Explained

### What Happens When You Upload a Document?

1. **Upload**: You select a file and upload it
2. **Processing**: SurfSense extracts the text
3. **Chunking**: The document is split into smaller pieces
4. **Embedding**: Each piece gets a "vector" (mathematical representation)
5. **Storage**: Everything is saved to the database
6. **Ready**: You can now search and chat with the document!

**How long does it take?** Most documents process in seconds to minutes, depending on size.

### How Does Search Work?

SurfSense uses **"Hybrid Search"** - the best of both worlds:

1. **Semantic Search**: Understands meaning
   - Question: "How do I improve sales?"
   - Finds: "Strategies to boost revenue" (different words, same meaning!)

2. **Keyword Search**: Finds exact matches
   - Question: "Project Alpha timeline"
   - Finds: Documents with those exact words

3. **Combined**: Results are merged and ranked for relevance

**Result**: You get the best matches, whether they use the same words or just similar concepts!

### What are Embeddings?

**Simple explanation**: Embeddings convert text into numbers that capture meaning.

**Example**:
- "cat" and "kitten" have similar embeddings (similar meanings)
- "cat" and "car" have different embeddings (different meanings)

This allows the AI to understand that "How to train puppies" is similar to "Dog training tips" even though they use different words.

**You don't need to understand the math** - just know that embeddings make smart search possible!

### What is RAG (Retrieval Augmented Generation)?

**RAG** = Retrieval + AI Generation

1. **Retrieval**: Find relevant information from your documents
2. **Augmented**: Add this information to the AI's context
3. **Generation**: AI generates an answer based on YOUR documents

**Why is this important?**
- ✅ AI answers are based on YOUR information
- ✅ Reduces "hallucinations" (AI making things up)
- ✅ Provides citations (you can verify answers)

## Getting Started

### Step 1: Installation

Choose your installation method:

**Option A: Docker (Easiest)**
1. Install Docker and Docker Compose
2. Clone the repository
3. Set up environment variables
4. Run `docker compose up`

**Option B: Manual Installation**
1. Install PostgreSQL with pgvector
2. Set up Python backend
3. Set up Node.js frontend
4. Configure environment variables
5. Start both servers

See the [main README](../README.md) for detailed instructions.

### Step 2: Create an Account

1. Navigate to `http://localhost:3000`
2. Click "Register"
3. Enter your email and password
4. Verify your email (if configured)

### Step 3: Create Your First Search Space

1. Click "Create Search Space"
2. Give it a name (e.g., "My First Space")
3. Add a description (optional)
4. Click "Create"

### Step 4: Upload Your First Document

1. Select your search space
2. Click "Documents" → "Upload Files"
3. Drag and drop files or click to browse
4. Wait for processing to complete
5. Your documents are now searchable!

### Step 5: Start Chatting

1. Go to "Researcher" or "Chats"
2. Type your question
3. Select search mode (Quick/General/Deep)
4. Click "Send"
5. Watch as AI generates an answer with citations!

## Common Use Cases

### Use Case 1: Research Assistant for Students

**Scenario**: You're writing a thesis and have dozens of research papers.

**How SurfSense Helps**:
1. Upload all PDFs to a "Thesis Research" search space
2. Ask questions like:
   - "What are the main theories about X?"
   - "Which papers discuss methodology Y?"
   - "Summarize the findings of Smith et al."
3. Generate a comprehensive report
4. All answers include citations to specific papers!

### Use Case 2: Knowledge Management for Developers

**Scenario**: You need to keep track of documentation, issues, and discussions.

**How SurfSense Helps**:
1. Connect GitHub connector to pull issues and PRs
2. Upload documentation files
3. Connect Slack for team discussions
4. Ask questions like:
   - "What's the status of the authentication feature?"
   - "Show me all bugs related to the payment system"
   - "Summarize the discussion about the database migration"

### Use Case 3: Meeting Notes Management

**Scenario**: You have meeting notes scattered across Notion, Google Docs, and email.

**How SurfSense Helps**:
1. Connect Notion and Gmail
2. Upload any standalone documents
3. Ask questions like:
   - "What decisions were made in last week's meetings?"
   - "Show me all action items assigned to me"
   - "What was discussed about the Q4 budget?"

### Use Case 4: Content Creation

**Scenario**: You're creating content and need to reference multiple sources.

**How SurfSense Helps**:
1. Upload source materials (articles, videos, transcripts)
2. Generate a comprehensive report on your topic
3. Convert the chat to a podcast for review
4. All with proper citations for your content!

## Step-by-Step Tutorials

### Tutorial 1: Upload and Search Documents

**Goal**: Upload documents and find specific information.

1. **Create a search space**:
   - Click "Search Spaces" → "Create New"
   - Name: "Tutorial Space"
   - Click "Create"

2. **Upload documents**:
   - Click "Documents" → "Upload Files"
   - Select 2-3 PDF or Word documents
   - Click "Upload"
   - Wait for processing (check "Logs" for status)

3. **Search by title**:
   - Click "Documents"
   - Use the search box to filter by title
   - View results

4. **Semantic search (via chat)**:
   - Click "Researcher"
   - Type: "What are the main topics covered?"
   - Click "Send"
   - Review the answer and citations

### Tutorial 2: Generate a Research Report

**Goal**: Create a comprehensive report from your documents.

1. **Ensure you have documents uploaded** (see Tutorial 1)

2. **Start a research session**:
   - Click "Researcher"
   - Enter your question: "Write a comprehensive overview of [your topic]"

3. **Select research depth**:
   - Choose "General" for standard report
   - Choose "Deep" for detailed analysis
   - Choose "Deeper" for most comprehensive

4. **Configure search**:
   - Search mode: "Chunks" (default, more precise)
   - Select connectors if you have them

5. **Submit and wait**:
   - Click "Send"
   - Watch as the report is generated section by section
   - Review citations

6. **Save the chat**:
   - The conversation is automatically saved
   - Find it in "Chats" for future reference

### Tutorial 3: Create a Podcast

**Goal**: Turn a chat conversation into an audio podcast.

1. **Complete Tutorial 2** (you need a chat first)

2. **Navigate to podcasts**:
   - Click "Podcasts"
   - Click "Create New Podcast"

3. **Select source**:
   - Choose the chat conversation from Tutorial 2
   - Enter a title for the podcast

4. **Generate**:
   - Click "Generate"
   - Wait ~20-30 seconds

5. **Listen**:
   - Click "Play" to listen in browser
   - Or click "Download" to save the MP3

### Tutorial 4: Connect External Sources

**Goal**: Integrate Notion pages into SurfSense.

1. **Get Notion API key**:
   - Go to https://www.notion.so/my-integrations
   - Create a new integration
   - Copy the API key
   - Share your pages with the integration

2. **Add connector in SurfSense**:
   - Click "Connectors" → "Add Connector"
   - Select "Notion"
   - Enter your API key
   - Enter page IDs you want to index

3. **Index content**:
   - Click "Index Now" (or wait for automatic sync)
   - Check "Logs" for progress

4. **Use in searches**:
   - Go to "Researcher"
   - Your Notion content is now searchable!
   - In search settings, enable "Notion Connector"

## Tips and Best Practices

### Organization Tips

1. **Use descriptive search space names**
   - ✅ "ML Research Papers 2024"
   - ❌ "Space 1"

2. **Separate contexts**
   - Create different spaces for work, personal, learning
   - Don't mix unrelated topics

3. **Regular cleanup**
   - Delete outdated documents
   - Archive completed projects

### Search Tips

1. **Be specific with questions**
   - ✅ "What were the Q4 revenue projections for Product X?"
   - ❌ "Tell me about money"

2. **Use different search modes**
   - Quick (QNA): For simple facts
   - Deep/Deeper: For comprehensive analysis

3. **Leverage citations**
   - Click on citations to see source documents
   - Verify important information

### Performance Tips

1. **Optimize document uploads**
   - Remove unnecessary files
   - Compress large files before uploading
   - Upload in batches, not all at once

2. **Choose appropriate search modes**
   - Chunks: Faster, more precise (default)
   - Documents: Broader context, slower

3. **Monitor logs**
   - Check logs for failed uploads
   - Address errors promptly

### Privacy Tips

1. **Sensitive information**
   - SurfSense is self-hosted, so your data stays with you
   - Use local LLMs (Ollama) for maximum privacy
   - Don't share API keys

2. **User management**
   - Each user's data is isolated
   - Search spaces are private to the owner

## Troubleshooting

### Document Upload Issues

**Problem**: Document upload fails

**Solutions**:
1. Check file format is supported
2. Verify file isn't corrupted
3. Check file size (very large files may timeout)
4. Review logs for specific error
5. Try a different ETL service (Docling vs Unstructured vs LlamaCloud)

**Problem**: Document processes but no content

**Solutions**:
1. Some formats may not be supported by your ETL service
2. Check if file is password-protected
3. Try converting to PDF first

### Search Issues

**Problem**: Search returns no results

**Solutions**:
1. Verify documents have finished processing (check logs)
2. Try different keywords
3. Check you're searching the correct search space
4. Ensure embeddings were generated successfully

**Problem**: Search returns irrelevant results

**Solutions**:
1. Try more specific questions
2. Use exact phrases in quotes
3. Use Deep/Deeper search modes for better quality
4. Consider adjusting reranking settings

### Chat Issues

**Problem**: Chat responses are slow

**Solutions**:
1. Check your LLM provider's status
2. Use Fast LLM for quicker responses
3. Reduce number of connectors searched
4. Use Chunks mode instead of Documents mode

**Problem**: Chat generates incorrect information

**Solutions**:
1. Verify source documents contain the information
2. Check citations - AI might be making inferences
3. Try more specific questions
4. Use Deep/Deeper modes for better accuracy

### Connection Issues

**Problem**: Cannot connect to external service (Slack, Notion, etc.)

**Solutions**:
1. Verify API key/token is correct
2. Check permissions granted to integration
3. Ensure service is accessible from your network
4. Review logs for specific error messages

## FAQ

### General Questions

**Q: Is my data safe?**
A: Yes! SurfSense is self-hosted, meaning your data never leaves your infrastructure. You have complete control.

**Q: Do I need a GPU?**
A: No, but it can help with:
- Local embedding models
- Local LLMs (Ollama)
For cloud-based services (OpenAI, etc.), no GPU needed.

**Q: How much does it cost?**
A: SurfSense is free and open-source. You only pay for:
- Infrastructure (if cloud-hosted)
- API keys (OpenAI, LlamaCloud, etc.)
- Can run 100% free with local models!

**Q: Can I use it offline?**
A: Partially:
- ✅ With local LLMs (Ollama) and local embeddings
- ❌ External connectors require internet
- ❌ Cloud LLM providers require internet

### Technical Questions

**Q: What's the difference between search modes (Chunks vs Documents)?**
A: 
- **Chunks**: Search small pieces of documents. More precise, faster.
- **Documents**: Search full documents. More context, slower.
Recommendation: Use Chunks (default) for most cases.

**Q: What embedding model should I use?**
A: SurfSense supports 6000+ models via Chonkie. Recommendations:
- Default: text-embedding-ada-002 (OpenAI)
- Local: sentence-transformers models
- See Chonkie documentation for options

**Q: How many documents can I upload?**
A: No hard limit, but consider:
- Database size
- Search performance (degrades with millions of documents)
- Storage space
For best performance, keep search spaces under 10,000 documents each.

**Q: Can multiple users share a search space?**
A: Currently, search spaces are private to each user. Multi-user collaboration may be added in future versions.

### Feature Questions

**Q: Can I edit documents after upload?**
A: No, documents are read-only. To update:
1. Delete old document
2. Upload new version

**Q: Can I download generated reports?**
A: Reports are shown in chat interface. You can:
- Copy the text
- Save the chat
- Take screenshots
Export features may be added in future.

**Q: Can I customize AI personas?**
A: Currently uses default system prompts. Custom personas may be added in future versions.

**Q: Does it support languages other than English?**
A: Yes! SurfSense supports:
- Multi-language documents
- Multi-language queries
- Depends on your embedding model and LLM

### Troubleshooting Questions

**Q: Why is processing slow?**
A: Possible reasons:
- Large files
- ETL service rate limits
- Slow LLM provider
- Limited system resources

**Q: How do I reset everything?**
A: To start fresh:
1. Delete all search spaces
2. Or reset database (see deployment guide)
⚠️ This deletes all data!

**Q: Where can I get help?**
A: 
- GitHub Issues: Report bugs
- Discord: Community support
- Documentation: This guide and others

## Next Steps

Now that you understand the basics:

1. **Read the Architecture Guide** ([ARCHITECTURE.md](ARCHITECTURE.md))
   - Understand how SurfSense works under the hood
   - Learn about the technology stack

2. **Explore API Reference** ([API_REFERENCE.md](API_REFERENCE.md))
   - If you want to build integrations
   - Programmatic access to SurfSense

3. **Check Deployment Guide** ([../DEPLOYMENT_GUIDE.md](../DEPLOYMENT_GUIDE.md))
   - Production deployment tips
   - Performance optimization

4. **Join the Community**
   - Discord: https://discord.gg/ejRNvftDp9
   - GitHub: https://github.com/yb235/SurfSense

## Summary

SurfSense is your personal AI research assistant that:
- 📚 Organizes your knowledge
- 🔍 Makes it searchable
- 💬 Lets you chat with your documents
- 📊 Generates reports with citations
- 🎙️ Creates podcasts
- 🔗 Integrates with your tools

**Key Takeaways**:
- Start with Search Spaces to organize
- Upload documents in supported formats
- Use Hybrid Search for best results
- Choose appropriate research depth
- Leverage external integrations
- Keep your data private and secure

Welcome to SurfSense! Happy researching! 🚀
