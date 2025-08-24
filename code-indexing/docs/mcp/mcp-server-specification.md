# CodeIndex MCP Server Technical Specification

## Overview

This document provides detailed technical specifications for implementing a Model Context Protocol (MCP) server that extracts and adapts the indexing and search capabilities from VS Code Copilot Chat for use by individual developers and OSS contributors.

## System Architecture

### High-Level Architecture

```mermaid
graph TB
    subgraph "MCP Client (Claude, Cursor, etc.)"
        A[AI Assistant]
        B[MCP Client]
    end
    
    subgraph "CodeIndex MCP Server"
        C[MCP Protocol Handler]
        D[Resource Handlers]
        E[Tool Handlers]
        F[Search Service]
        G[Indexing Service]
        H[Cache Service]
        I[Config Service]
        J[Workspace Service]
    end
    
    subgraph "Storage Layer"
        K[SQLite Database]
        L[File System Cache]
        M[Configuration Files]
    end
    
    subgraph "External Services"
        N[Embedding Providers]
        O[GitHub API (Optional)]
    end
    
    A --> B
    B <--> C
    C --> D
    C --> E
    D --> F
    E --> F
    F --> G
    F --> H
    G --> H
    G --> I
    G --> J
    H --> K
    H --> L
    I --> M
    G --> N
    G --> O
```

### Core Services

#### MCP Protocol Handler
- **Purpose**: Handles MCP protocol communication
- **Technology**: TypeScript with @modelcontextprotocol/sdk
- **Responsibilities**:
  - Protocol message parsing and validation
  - Resource and tool request routing
  - Error handling and response formatting
  - Connection lifecycle management

#### Search Service
- **Purpose**: Orchestrates search operations across multiple strategies
- **Responsibilities**:
  - Query preprocessing and optimization
  - Strategy selection based on workspace size and configuration
  - Result ranking and relevance scoring
  - Search result caching and deduplication

#### Indexing Service
- **Purpose**: Manages file discovery, processing, and embedding generation
- **Responsibilities**:
  - File system scanning with sophisticated filtering
  - Text chunking using NaiveChunker algorithm
  - Embedding generation and storage
  - Incremental index updates via file watchers

#### Cache Service
- **Purpose**: Multi-level caching for performance optimization
- **Responsibilities**:
  - Memory-based LRU cache for recent results
  - SQLite-based persistent cache for embeddings and metadata
  - Cache invalidation on file changes
  - Cache size management and cleanup

#### Configuration Service
- **Purpose**: Manages system configuration and settings
- **Responsibilities**:
  - Configuration file parsing and validation
  - Runtime configuration updates
  - Default value management
  - Environment variable integration

#### Workspace Service
- **Purpose**: Handles workspace detection and management
- **Responsibilities**:
  - Workspace root detection (git, package.json, etc.)
  - Multi-workspace support
  - Path normalization and validation
  - File system permission checks

## MCP Interface Specification

### Resources

#### `codeindex://workspace/status`
**Description**: Current workspace indexing status and statistics

**Response Schema**:
```json
{
  "workspace": {
    "path": "/path/to/workspace",
    "name": "project-name",
    "lastIndexed": "2025-01-15T10:30:00Z"
  },
  "indexing": {
    "status": "ready|indexing|error",
    "totalFiles": 15420,
    "indexedFiles": 15420,
    "skippedFiles": 245,
    "totalChunks": 87650,
    "embeddedChunks": 87650,
    "progress": 1.0
  },
  "strategies": {
    "available": ["embeddings", "tfidf", "fullworkspace"],
    "preferred": "embeddings",
    "fallback": ["tfidf"]
  },
  "cache": {
    "memoryUsage": "245MB",
    "diskUsage": "1.2GB",
    "hitRate": 0.85,
    "lastCleanup": "2025-01-15T09:00:00Z"
  }
}
```

#### `codeindex://search/{query}`
**Description**: Search results for natural language query

**Parameters**:
- `query` (string): Natural language search query
- `maxResults` (number, optional): Maximum number of results (default: 50)
- `strategy` (string, optional): Preferred search strategy
- `threshold` (number, optional): Relevance threshold (0.0-1.0)

**Response Schema**:
```json
{
  "query": "user search query",
  "strategy": "embeddings",
  "executionTime": 245,
  "totalResults": 42,
  "results": [
    {
      "file": "/path/to/file.ts",
      "fileName": "file.ts",
      "chunk": {
        "text": "relevant code snippet...",
        "startLine": 125,
        "endLine": 140,
        "tokens": 230
      },
      "relevance": 0.87,
      "context": {
        "function": "processUserInput",
        "class": "UserService",
        "language": "typescript"
      }
    }
  ],
  "metadata": {
    "searchedFiles": 15420,
    "searchedChunks": 87650,
    "cacheHit": true
  }
}
```

#### `codeindex://file/{path}/chunks`
**Description**: Individual file chunks with metadata

**Parameters**:
- `path` (string): Relative file path within workspace

**Response Schema**:
```json
{
  "file": "/path/to/file.ts",
  "language": "typescript",
  "size": 8542,
  "lastModified": "2025-01-15T10:15:00Z",
  "chunks": [
    {
      "index": 0,
      "text": "chunk content...",
      "startLine": 1,
      "endLine": 25,
      "tokens": 250,
      "embedding": {
        "provider": "openai",
        "model": "text-embedding-3-small",
        "generated": "2025-01-15T10:16:00Z"
      }
    }
  ]
}
```

#### `codeindex://workspace/stats`
**Description**: Comprehensive workspace statistics and analytics

**Response Schema**:
```json
{
  "files": {
    "total": 15420,
    "byLanguage": {
      "typescript": 8540,
      "javascript": 3250,
      "python": 2140,
      "other": 1490
    },
    "bySize": {
      "small": 12340,
      "medium": 2850,
      "large": 230
    }
  },
  "chunks": {
    "total": 87650,
    "averageSize": 185,
    "embedded": 87650,
    "tfidfIndexed": 87650
  },
  "performance": {
    "averageSearchTime": 245,
    "indexingTime": 1850,
    "cacheHitRate": 0.85,
    "memoryUsage": "245MB"
  }
}
```

### Tools

#### `search_code`
**Description**: Perform semantic search across workspace

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "Natural language search query"
    },
    "maxResults": {
      "type": "number",
      "description": "Maximum number of results",
      "default": 50,
      "minimum": 1,
      "maximum": 200
    },
    "strategy": {
      "type": "string",
      "enum": ["auto", "embeddings", "tfidf", "fullworkspace"],
      "description": "Search strategy preference",
      "default": "auto"
    },
    "threshold": {
      "type": "number",
      "description": "Relevance threshold (0.0-1.0)",
      "default": 0.65,
      "minimum": 0.0,
      "maximum": 1.0
    },
    "fileTypes": {
      "type": "array",
      "items": {"type": "string"},
      "description": "Filter by file extensions",
      "examples": [["ts", "js"], ["py"], ["*"]]
    }
  },
  "required": ["query"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "results": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": {"type": "string"},
          "chunk": {
            "type": "object",
            "properties": {
              "text": {"type": "string"},
              "startLine": {"type": "number"},
              "endLine": {"type": "number"}
            }
          },
          "relevance": {"type": "number"},
          "context": {"type": "object"}
        }
      }
    },
    "metadata": {
      "type": "object",
      "properties": {
        "strategy": {"type": "string"},
        "executionTime": {"type": "number"},
        "totalResults": {"type": "number"}
      }
    }
  }
}
```

#### `reindex_workspace`
**Description**: Trigger workspace reindexing

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "force": {
      "type": "boolean",
      "description": "Force full reindex even if files haven't changed",
      "default": false
    },
    "path": {
      "type": "string",
      "description": "Specific path to reindex (relative to workspace root)",
      "default": null
    },
    "strategy": {
      "type": "string",
      "enum": ["embeddings", "tfidf", "both"],
      "description": "Which indexing strategies to rebuild",
      "default": "both"
    }
  }
}
```

#### `update_config`
**Description**: Update runtime configuration

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "settings": {
      "type": "object",
      "description": "Configuration settings to update",
      "properties": {
        "chunkSize": {"type": "number", "minimum": 50, "maximum": 1000},
        "embeddingProvider": {"type": "string", "enum": ["openai", "ollama", "local"]},
        "maxFiles": {"type": "number", "minimum": 100, "maximum": 100000},
        "cacheSize": {"type": "string", "pattern": "^\\d+[KMGT]B$"}
      }
    },
    "restart": {
      "type": "boolean",
      "description": "Whether to restart indexing after config change",
      "default": false
    }
  },
  "required": ["settings"]
}
```

#### `get_file_context`
**Description**: Get specific file context with surrounding code

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "description": "File path relative to workspace root"
    },
    "lineRange": {
      "type": "object",
      "description": "Specific line range to focus on",
      "properties": {
        "start": {"type": "number", "minimum": 1},
        "end": {"type": "number", "minimum": 1}
      }
    },
    "contextLines": {
      "type": "number",
      "description": "Number of lines to include before/after range",
      "default": 10,
      "minimum": 0,
      "maximum": 100
    }
  },
  "required": ["path"]
}
```

#### `find_similar_code`
**Description**: Find code similar to provided snippet

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "code": {
      "type": "string",
      "description": "Code snippet to find similar patterns for"
    },
    "language": {
      "type": "string",
      "description": "Programming language of the snippet",
      "examples": ["typescript", "python", "javascript", "go"]
    },
    "maxResults": {
      "type": "number",
      "description": "Maximum number of similar code snippets to return",
      "default": 20,
      "minimum": 1,
      "maximum": 100
    }
  },
  "required": ["code"]
}
```

### Prompts

#### `code-search`
**Description**: Template for effective semantic code search

**Arguments**:
- `intent` (string, required): What you're trying to accomplish
- `context` (string, optional): Additional context about the codebase
- `language` (string, optional): Preferred programming language

**Template**:
```
Search the codebase for: {{intent}}

{{#if context}}
Additional context: {{context}}
{{/if}}

{{#if language}}
Focus on {{language}} code.
{{/if}}

Please provide relevant code snippets that help with this task.
```

#### `workspace-overview`
**Description**: Generate comprehensive workspace analysis

**Arguments**:
- `focus` (string, optional): Specific area to focus on (architecture, patterns, etc.)

**Template**:
```
Analyze this workspace and provide:

1. **Project Structure**: Main directories and their purposes
2. **Architecture Patterns**: Key architectural decisions and patterns used
3. **Technologies**: Programming languages, frameworks, and tools
4. **Entry Points**: Main files and key components
{{#if focus}}
5. **{{focus}} Analysis**: Detailed analysis of {{focus}}
{{/if}}

Use the workspace indexing to understand the codebase comprehensively.
```

## Configuration Schema

### Primary Configuration File (`codeindex.config.json`)

```json
{
  "$schema": "https://schemas.codeindex.dev/config/v1.json",
  "workspace": {
    "paths": ["/path/to/workspace"],
    "excludePatterns": [
      "node_modules/**",
      ".git/**", 
      "dist/**",
      "build/**",
      "*.log",
      "*.tmp"
    ],
    "includePatterns": ["**/*"],
    "maxFileSize": "1.5MB",
    "maxFiles": 25000,
    "watchFiles": true
  },
  "indexing": {
    "chunkSize": 250,
    "chunkOverlap": 50,
    "strategies": ["embeddings", "tfidf"],
    "embeddingProvider": "openai",
    "updateInterval": "60s",
    "batchSize": 100
  },
  "search": {
    "defaultMaxResults": 50,
    "relevanceThreshold": 0.65,
    "timeout": "30s",
    "enableCaching": true,
    "cacheSize": "500MB"
  },
  "providers": {
    "openai": {
      "apiKey": "sk-...",
      "model": "text-embedding-3-small",
      "batchSize": 100
    },
    "ollama": {
      "baseUrl": "http://localhost:11434",
      "model": "nomic-embed-text",
      "timeout": "30s"
    }
  },
  "server": {
    "name": "CodeIndex MCP Server",
    "version": "1.0.0",
    "transport": "stdio",
    "logLevel": "info",
    "enableTelemetry": false
  }
}
```

### Environment Variables

```bash
# Workspace Configuration
CODEINDEX_WORKSPACE_PATH="/path/to/workspace"
CODEINDEX_MAX_FILES=25000
CODEINDEX_MAX_FILE_SIZE="1.5MB"

# Indexing Configuration  
CODEINDEX_CHUNK_SIZE=250
CODEINDEX_EMBEDDING_PROVIDER="openai"
CODEINDEX_UPDATE_INTERVAL="60s"

# Provider Configuration
CODEINDEX_OPENAI_API_KEY="sk-..."
CODEINDEX_OLLAMA_BASE_URL="http://localhost:11434"

# Server Configuration
CODEINDEX_LOG_LEVEL="info"
CODEINDEX_CACHE_SIZE="500MB"
CODEINDEX_ENABLE_TELEMETRY="false"
```

## Database Schema

### SQLite Database Structure

```sql
-- Workspace metadata
CREATE TABLE workspaces (
    id INTEGER PRIMARY KEY,
    path TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    last_indexed DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- File tracking
CREATE TABLE files (
    id INTEGER PRIMARY KEY,
    workspace_id INTEGER NOT NULL,
    relative_path TEXT NOT NULL,
    absolute_path TEXT NOT NULL,
    file_hash TEXT NOT NULL,
    file_size INTEGER NOT NULL,
    language TEXT,
    last_modified DATETIME,
    indexed_at DATETIME,
    chunk_count INTEGER DEFAULT 0,
    FOREIGN KEY (workspace_id) REFERENCES workspaces (id),
    UNIQUE (workspace_id, relative_path)
);

-- Text chunks
CREATE TABLE chunks (
    id INTEGER PRIMARY KEY,
    file_id INTEGER NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    content_hash TEXT NOT NULL,
    start_line INTEGER NOT NULL,
    end_line INTEGER NOT NULL,
    token_count INTEGER NOT NULL,
    language TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (file_id) REFERENCES files (id),
    UNIQUE (file_id, chunk_index)
);

-- Embeddings storage
CREATE TABLE embeddings (
    id INTEGER PRIMARY KEY,
    chunk_id INTEGER NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    embedding BLOB NOT NULL,
    dimensions INTEGER NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (chunk_id) REFERENCES chunks (id),
    UNIQUE (chunk_id, provider, model)
);

-- TF-IDF index
CREATE TABLE tfidf_terms (
    id INTEGER PRIMARY KEY,
    term TEXT UNIQUE NOT NULL,
    document_frequency INTEGER DEFAULT 0
);

CREATE TABLE tfidf_chunk_terms (
    chunk_id INTEGER NOT NULL,
    term_id INTEGER NOT NULL,
    frequency INTEGER NOT NULL,
    tf_idf_score REAL NOT NULL,
    FOREIGN KEY (chunk_id) REFERENCES chunks (id),
    FOREIGN KEY (term_id) REFERENCES tfidf_terms (id),
    PRIMARY KEY (chunk_id, term_id)
);

-- Cache management
CREATE TABLE cache_entries (
    key TEXT PRIMARY KEY,
    value BLOB NOT NULL,
    expiry DATETIME,
    size INTEGER NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    accessed_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Configuration storage
CREATE TABLE config (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    type TEXT NOT NULL DEFAULT 'string',
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Performance indexes
CREATE INDEX idx_files_workspace_path ON files (workspace_id, relative_path);
CREATE INDEX idx_files_hash ON files (file_hash);
CREATE INDEX idx_chunks_file ON chunks (file_id);
CREATE INDEX idx_chunks_content_hash ON chunks (content_hash);
CREATE INDEX idx_embeddings_chunk ON embeddings (chunk_id);
CREATE INDEX idx_tfidf_terms_term ON tfidf_terms (term);
CREATE INDEX idx_tfidf_chunk_terms_chunk ON tfidf_chunk_terms (chunk_id);
CREATE INDEX idx_tfidf_chunk_terms_score ON tfidf_chunk_terms (tf_idf_score DESC);
CREATE INDEX idx_cache_expiry ON cache_entries (expiry);
```

## Performance Specifications

### Target Performance Metrics

| Operation | Target Time | Maximum Memory | Scalability |
|-----------|-------------|----------------|-------------|
| Initial Indexing (1K files) | <30 seconds | <100MB | Linear |
| Initial Indexing (10K files) | <5 minutes | <500MB | Sub-linear |
| Search Query | <500ms | <50MB | Constant |
| Incremental Update | <5 seconds | <50MB | Proportional |
| File Watch Response | <1 second | <10MB | Constant |

### Resource Limits

```json
{
  "memory": {
    "heap": "1GB",
    "cache": "500MB", 
    "embeddings": "300MB",
    "buffer": "200MB"
  },
  "disk": {
    "database": "10GB",
    "cache": "5GB",
    "logs": "1GB"
  },
  "network": {
    "embedding_timeout": "30s",
    "batch_size": 100,
    "concurrent_requests": 10
  },
  "processing": {
    "max_files": 50000,
    "max_file_size": "5MB",
    "chunk_timeout": "10s",
    "index_timeout": "30m"
  }
}
```

## Error Handling Strategy

### Error Categories

1. **Configuration Errors**: Invalid config, missing required settings
2. **File System Errors**: Permission issues, file not found, disk space
3. **Network Errors**: Embedding provider timeouts, API rate limits
4. **Processing Errors**: Chunking failures, embedding generation issues
5. **Resource Errors**: Memory exhaustion, disk space, CPU overload

### Error Response Format

```json
{
  "error": {
    "code": "INDEX_TIMEOUT",
    "message": "Indexing operation timed out after 30 minutes",
    "category": "processing",
    "severity": "error",
    "retryable": true,
    "context": {
      "operation": "initial_indexing",
      "workspace": "/path/to/workspace",
      "files_processed": 15420,
      "files_remaining": 2340
    },
    "suggestions": [
      "Try reducing maxFiles configuration",
      "Check available system memory",
      "Consider excluding large binary files"
    ],
    "timestamp": "2025-01-15T10:30:00Z"
  }
}
```

## Security Considerations

### Data Protection
- All processing occurs locally, no code sent to external services (except for embedding generation when using cloud providers)
- Sensitive file patterns excluded by default (.env, secrets, keys)
- Configurable data retention policies
- Optional encryption for cache and database storage

### Access Control
- File system permissions respected
- Workspace boundary enforcement
- Optional authentication for embedding providers
- Rate limiting for resource-intensive operations

### Privacy Features
- Opt-out telemetry with clear data collection policies
- Local-only operation mode without external dependencies
- Configurable data retention and cleanup
- Clear audit trail of all operations

## Deployment Options

### NPM Package
```bash
npm install -g @codeindex/mcp-server
codeindex-mcp --workspace /path/to/project
```

### Docker Container
```bash
docker run -v /local/project:/workspace codeindex/mcp-server
```

### Development Setup
```bash
git clone https://github.com/codeindex/mcp-server
cd mcp-server
npm install
npm run build
npm start -- --workspace /path/to/project
```

This specification provides the foundation for implementing a production-grade MCP server that delivers sophisticated code indexing and search capabilities while maintaining the flexibility and performance characteristics of the original VS Code implementation.