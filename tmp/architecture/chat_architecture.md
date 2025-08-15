# Chat System Architecture with MCP and RAG Integration

## System Overview

This architecture demonstrates a modern chat system that leverages MCP (Model Context Protocol) as the central orchestrator for routing requests to appropriate services and tools, including RAG (Retrieval-Augmented Generation) capabilities.

## Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        UI[Chat UI/Frontend]
        API[API Gateway]
    end

    subgraph "Orchestration Layer"
        MCP[MCP Orchestrator<br/>Decision Engine]
        Router[Request Router]
        Context[Context Manager]
    end

    subgraph "RAG Pipeline"
        Embedder[Text Embedder]
        VectorDB[(Vector Database<br/>Pinecone/Weaviate)]
        Retriever[Document Retriever]
        Reranker[Result Reranker]
    end

    subgraph "Tool Services"
        CodeTool[Code Analysis Tool]
        SearchTool[Web Search Tool]
        DBTool[Database Query Tool]
        FileTool[File System Tool]
        CalcTool[Calculator Tool]
        ImageTool[Image Processing Tool]
    end

    subgraph "Knowledge Sources"
        DocStore[(Document Store)]
        KnowledgeBase[(Knowledge Base)]
        CodeRepo[(Code Repository)]
        APIEndpoints[External APIs]
    end

    subgraph "LLM Services"
        MainLLM[Primary LLM<br/>GPT-4/Claude]
        SpecializedLLM[Specialized LLMs<br/>Code/Math/Vision]
        LocalLLM[Local LLM<br/>Llama/Mistral]
    end

    subgraph "Response Processing"
        ResponseGen[Response Generator]
        Validator[Response Validator]
        Cache[(Response Cache)]
    end

    %% Client to Orchestration
    UI --> API
    API --> MCP
    MCP --> Router
    Router --> Context

    %% MCP Decision Flow
    Context --> |Query Analysis| MCP
    MCP -->|RAG Required| Retriever
    MCP -->|Tool Required| CodeTool
    MCP -->|Tool Required| SearchTool
    MCP -->|Tool Required| DBTool
    MCP -->|Tool Required| FileTool
    MCP -->|Tool Required| CalcTool
    MCP -->|Tool Required| ImageTool

    %% RAG Pipeline Flow
    Retriever --> Embedder
    Embedder --> VectorDB
    VectorDB --> Reranker
    Reranker --> MainLLM

    %% Knowledge Source Connections
    DocStore --> Retriever
    KnowledgeBase --> Retriever
    CodeRepo --> CodeTool
    APIEndpoints --> SearchTool

    %% Tool to LLM Flow
    CodeTool --> SpecializedLLM
    SearchTool --> MainLLM
    DBTool --> MainLLM
    FileTool --> LocalLLM
    CalcTool --> SpecializedLLM
    ImageTool --> SpecializedLLM

    %% LLM to Response
    MainLLM --> ResponseGen
    SpecializedLLM --> ResponseGen
    LocalLLM --> ResponseGen
    ResponseGen --> Validator
    Validator --> Cache
    Cache --> API

    %% Styling
    classDef clientStyle fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef orchStyle fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef ragStyle fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef toolStyle fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px
    classDef knowledgeStyle fill:#fce4ec,stroke:#880e4f,stroke-width:2px
    classDef llmStyle fill:#f1f8e9,stroke:#33691e,stroke-width:2px
    classDef responseStyle fill:#e0f2f1,stroke:#004d40,stroke-width:2px

    class UI,API clientStyle
    class MCP,Router,Context orchStyle
    class Embedder,VectorDB,Retriever,Reranker ragStyle
    class CodeTool,SearchTool,DBTool,FileTool,CalcTool,ImageTool toolStyle
    class DocStore,KnowledgeBase,CodeRepo,APIEndpoints knowledgeStyle
    class MainLLM,SpecializedLLM,LocalLLM llmStyle
    class ResponseGen,Validator,Cache responseStyle
```

## Component Descriptions

### MCP Orchestrator
The central decision engine that:
- Analyzes incoming queries
- Determines which tools/services to invoke
- Manages the flow between RAG and tool-based responses
- Coordinates multi-step workflows

### RAG Pipeline
- **Text Embedder**: Converts queries and documents to vector representations
- **Vector Database**: Stores and retrieves document embeddings
- **Document Retriever**: Fetches relevant documents based on similarity
- **Result Reranker**: Optimizes retrieved results for relevance

### Tool Services
Specialized tools that MCP can invoke:
- **Code Analysis**: Analyzes and generates code
- **Web Search**: Retrieves real-time information
- **Database Query**: Executes structured data queries
- **File System**: Manages local file operations
- **Calculator**: Performs complex calculations
- **Image Processing**: Handles vision-related tasks

### Response Flow
1. User query enters through Chat UI
2. API Gateway forwards to MCP Orchestrator
3. MCP analyzes query and decides on approach:
   - RAG for knowledge-based queries
   - Tools for specific actions
   - Direct LLM for general conversation
4. Selected services process the request
5. Response is generated, validated, and cached
6. Final response returned to user

## Key Features
- **Intelligent Routing**: MCP decides optimal path for each query
- **Hybrid Approach**: Combines RAG with tool-based responses
- **Multi-LLM Support**: Different LLMs for different tasks
- **Caching**: Improves response time for repeated queries
- **Extensible**: Easy to add new tools and services