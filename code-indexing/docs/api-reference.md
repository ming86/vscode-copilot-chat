# API Reference Documentation

## Overview

This document provides comprehensive technical reference for the key interfaces, classes, and types in the VS Code Copilot Chat indexing system. It serves as a developer reference for understanding and extending the indexing functionality.

## Core Service Interfaces

### IWorkspaceChunkSearchService

**Location**: `src/platform/workspaceChunkSearch/node/workspaceChunkSearchService.ts:72-95`

The primary service interface for workspace search operations.

```typescript
export const IWorkspaceChunkSearchService = createServiceIdentifier<IWorkspaceChunkSearchService>('IWorkspaceChunkSearchService');

export interface IWorkspaceChunkSearchService extends IDisposable {
    readonly _serviceBrand: undefined;
    readonly onDidChangeIndexState: Event<void>;

    /**
     * Get the current state of local and remote indexes
     */
    getIndexState(): Promise<WorkspaceIndexState>;

    /**
     * Check if fast search is available for the given sizing parameters
     */
    hasFastSearch(sizing: StrategySearchSizing): Promise<boolean>;

    /**
     * Search for file chunks matching the given query
     */
    searchFileChunks(
        sizing: WorkspaceChunkSearchSizing,
        query: WorkspaceChunkQuery,
        options: WorkspaceChunkSearchOptions,
        telemetryInfo: TelemetryCorrelationId,
        progress: vscode.Progress<vscode.ChatResponsePart> | undefined,
        token: CancellationToken,
    ): Promise<WorkspaceChunkSearchResult>;

    /**
     * Trigger local indexing (embeddings or TF-IDF)
     */
    triggerLocalIndexing(
        trigger: BuildIndexTriggerReason, 
        telemetryInfo: TelemetryCorrelationId
    ): Promise<Result<true, TriggerIndexingError>>;

    /**
     * Trigger remote indexing (GitHub code search)
     */
    triggerRemoteIndexing(
        trigger: BuildIndexTriggerReason, 
        telemetryInfo: TelemetryCorrelationId
    ): Promise<Result<true, TriggerIndexingError>>;
}
```

#### Events

- **onDidChangeIndexState**: Fired when local or remote index state changes
  - Debounced to 250ms to prevent excessive events
  - Combines events from multiple search strategies

#### Methods

**getIndexState()**
- Returns current state of both local embeddings and remote code search indexes
- Used by UI to show indexing progress and availability
- Non-blocking operation

**hasFastSearch(sizing)**
- Determines if fast search strategies are available
- Considers file count, index readiness, and authentication status
- Used to optimize UI behavior and user expectations

**searchFileChunks(sizing, query, options, telemetryInfo, progress, token)**
- Main search entry point with comprehensive parameter set
- Handles strategy selection, fallback, and result processing
- Includes progress reporting and telemetry integration

**triggerLocalIndexing(trigger, telemetryInfo)**
- Manually initiate local indexing (embeddings or TF-IDF)
- Returns success/failure result with error details
- Respects workspace size limits and user preferences

**triggerRemoteIndexing(trigger, telemetryInfo)**
- Manually initiate GitHub code search indexing
- Requires authentication and repository access
- Handles repository detection and indexing requests

### IWorkspaceFileIndex

**Location**: `src/platform/workspaceChunkSearch/node/workspaceFileIndex.ts`

Interface for workspace file discovery and management.

```typescript
export const IWorkspaceFileIndex = createServiceIdentifier<IWorkspaceFileIndex>('IWorkspaceFileIndex');

export interface IWorkspaceFileIndex extends IDisposable {
    readonly _serviceBrand: undefined;
    readonly onDidChangeFileIndex: Event<void>;

    /**
     * Total number of files currently in the index
     */
    readonly fileCount: number;

    /**
     * Get all indexable files in the workspace
     */
    getFiles(token: CancellationToken): Promise<Iterable<URI>>;

    /**
     * Get content for a specific file
     */
    getFileContent(uri: URI, token: CancellationToken): Promise<string | undefined>;

    /**
     * Check if a file should be indexed
     */
    shouldIndexFile(uri: URI, token: CancellationToken): Promise<boolean>;

    /**
     * Refresh the file index
     */
    refresh(token: CancellationToken): Promise<void>;
}
```

### IWorkspaceChunkSearchStrategy

**Location**: `src/platform/workspaceChunkSearch/common/workspaceChunkSearch.ts`

Interface that all search strategies must implement.

```typescript
export interface IWorkspaceChunkSearchStrategy {
    /**
     * Unique identifier for this strategy
     */
    readonly id: WorkspaceChunkSearchStrategyId;

    /**
     * Optional preparation step before search
     */
    prepareSearchWorkspace?(
        telemetryInfo: TelemetryCorrelationId, 
        token: CancellationToken
    ): Promise<void>;

    /**
     * Execute the search strategy
     */
    searchWorkspace(
        sizing: StrategySearchSizing,
        query: WorkspaceChunkQueryWithEmbeddings,
        options: WorkspaceChunkSearchOptions,
        telemetryInfo: TelemetryCorrelationId,
        token: CancellationToken
    ): Promise<StrategySearchResult | undefined>;
}
```

#### Strategy IDs

```typescript
export enum WorkspaceChunkSearchStrategyId {
    FullWorkspace = 'fullWorkspace',
    CodeSearch = 'codeSearch',
    Embeddings = 'embeddings',
    TfIdfWithSemantic = 'tfIdfWithSemantic',
    TfIdf = 'tfIdf'
}
```

## Data Types and Interfaces

### Chunk Types

**Location**: `src/platform/chunking/common/chunk.ts`

```typescript
/**
 * Basic file chunk with metadata
 */
export interface FileChunk {
    /** Source file URI */
    readonly file: URI;
    /** Processed text content for search */
    readonly text: string;
    /** Original text content for display */
    readonly rawText: string;
    /** True if this chunk contains the entire file */
    readonly isFullFile: boolean;
    /** Line range within the source file */
    readonly range: Range;
}

/**
 * File chunk with associated embedding vector
 */
export interface FileChunkWithEmbedding extends FileChunk {
    readonly embedding: Embedding;
}

/**
 * File chunk with similarity score
 */
export interface FileChunkAndScore extends FileChunk {
    readonly distance?: EmbeddingDistance;
}
```

### Query Types

**Location**: `src/platform/workspaceChunkSearch/common/workspaceChunkSearch.ts`

```typescript
/**
 * Search query interface
 */
export interface WorkspaceChunkQuery {
    /**
     * Resolve the query to searchable text
     */
    resolveQuery(token: CancellationToken): Promise<string>;
}

/**
 * Query with embedded representations
 */
export interface WorkspaceChunkQueryWithEmbeddings extends WorkspaceChunkQuery {
    /**
     * Resolve query to embedding vector
     */
    resolveQueryEmbeddings(token: CancellationToken): Promise<Embedding>;
}
```

### Search Results

```typescript
/**
 * Search result from a strategy
 */
export interface StrategySearchResult {
    readonly chunks: readonly FileChunkAndScore[];
    readonly alerts?: readonly WorkspaceSearchAlert[];
}

/**
 * Final search result with metadata
 */
export interface WorkspaceChunkSearchResult {
    readonly chunks: readonly FileChunkAndScore[];
    readonly isFullWorkspace: boolean;
    readonly alerts?: readonly WorkspaceSearchAlert[];
    readonly strategy?: string;
}
```

### Search Configuration

```typescript
/**
 * Search sizing parameters
 */
export interface StrategySearchSizing {
    readonly endpoint: IChatEndpoint;
    readonly tokenBudget: number | undefined;
    readonly maxResultCountHint: number | undefined;
}

/**
 * Workspace-specific search sizing
 */
export interface WorkspaceChunkSearchSizing {
    readonly endpoint: IChatEndpoint;
    readonly tokenBudget: number | undefined;
    readonly maxResults: number | undefined;
}

/**
 * Search options and preferences
 */
export interface WorkspaceChunkSearchOptions {
    readonly includeOpenFiles?: boolean;
    readonly respectIgnoreFiles?: boolean;
    readonly maxFileSize?: number;
}
```

### Index State Types

```typescript
/**
 * Combined index state
 */
export interface WorkspaceIndexState {
    readonly remoteIndexState: CodeSearchRemoteIndexState;
    readonly localIndexState: LocalEmbeddingsIndexState;
}

/**
 * Remote code search index state
 */
export interface CodeSearchRemoteIndexState {
    readonly status: 'disabled' | 'loading' | 'loaded';
    readonly repos: readonly RepoIndexInfo[];
}

/**
 * Local embeddings index state
 */
export interface LocalEmbeddingsIndexState {
    readonly status: LocalEmbeddingsIndexStatus;
    readonly getState: () => Promise<EmbeddingsIndexDetails | undefined>;
}

export enum LocalEmbeddingsIndexStatus {
    Unknown = 'unknown',
    Ready = 'ready',
    UpdatingIndex = 'updating',
    RequiresIndexing = 'requiresIndexing'
}
```

## Embedding System

### Embedding Types

**Location**: `src/platform/embeddings/common/embeddingsComputer.ts`

```typescript
/**
 * Embedding vector with type information
 */
export interface Embedding {
    readonly embeddingType: EmbeddingType;
    readonly values: readonly number[];
}

/**
 * Collection of embeddings
 */
export interface Embeddings {
    readonly embeddingType: EmbeddingType;
    readonly values: readonly Embedding[];
}

/**
 * Distance between embeddings
 */
export interface EmbeddingDistance {
    readonly value: number;
    readonly embeddingType: EmbeddingType;
}

/**
 * Embedding model type
 */
export interface EmbeddingType {
    readonly model: string;
    readonly version: string;
    equals(other: EmbeddingType): boolean;
}
```

### Embedding Computer Interface

```typescript
export interface IEmbeddingsComputer {
    /**
     * Compute embeddings for the given strings
     */
    computeEmbeddings(
        embeddingType: EmbeddingType,
        strings: readonly string[],
        options: { inputType: 'query' | 'document' },
        token: CancellationToken
    ): Promise<Embeddings | undefined>;
}
```

### Distance Calculation

```typescript
/**
 * Compute cosine similarity distance between embeddings
 */
export function distance(a: Embedding, b: Embedding): EmbeddingDistance {
    if (!a.embeddingType.equals(b.embeddingType)) {
        throw new Error('Cannot compute distance between different embedding types');
    }
    
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;
    
    for (let i = 0; i < a.values.length; i++) {
        dotProduct += a.values[i] * b.values[i];
        normA += a.values[i] * a.values[i];
        normB += b.values[i] * b.values[i];
    }
    
    const similarity = dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
    return {
        value: similarity,
        embeddingType: a.embeddingType
    };
}
```

## Chunking System

### NaiveChunker

**Location**: `src/platform/chunking/node/naiveChunker.ts:21-141`

```typescript
export class NaiveChunker {
    constructor(
        endpoint: TokenizationEndpoint,
        @ITokenizerProvider tokenizerProvider: ITokenizerProvider
    );

    /**
     * Split file content into chunks
     */
    async chunkFile(
        uri: URI, 
        text: string, 
        options: {
            maxTokenLength?: number;
            removeEmptyLines?: boolean;
        }, 
        token: CancellationToken
    ): Promise<FileChunk[]>;
}
```

### Chunking Configuration

```typescript
export const MAX_CHUNK_SIZE_TOKENS = 250;

interface ChunkingOptions {
    /** Maximum tokens per chunk */
    maxTokenLength?: number;
    /** Remove empty lines from output */
    removeEmptyLines?: boolean;
}
```

## File System Integration

### File Filtering

**Location**: `src/platform/workspaceChunkSearch/node/workspaceFileIndex.ts:183-237`

```typescript
/**
 * Check if file should always be ignored
 */
export function shouldAlwaysIgnoreFile(resource: URI): boolean;

/**
 * Check if file should be indexed (includes ignore file checks)
 */
export async function shouldIndexFile(
    accessor: ServicesAccessor, 
    resource: URI, 
    token: CancellationToken
): Promise<boolean>;
```

### Configuration Constants

```typescript
/** Maximum size of indexable files */
const maxIndexableFileSize = 1.5 * 1024 * 1024; // 1.5 MB

/** File extensions to exclude */
const EXCLUDE_EXTENSIONS = new Set([
    // Images, videos, audio, archives, etc.
]);

/** Folder names to exclude */
const EXCLUDED_FOLDERS = [
    'node_modules', 'venv', 'out', 'dist', '.git', // ...
];

/** Specific files to exclude */
const EXCLUDED_FILES = [
    '.ds_store', 'thumbs.db', 'package-lock.json', // ...
];

/** URI schemes to exclude */
const EXCLUDED_SCHEMES = [
    Schemas.vscode, Schemas.vscodeUserData, 'output', // ...
];
```

## Error Handling

### Result Types

```typescript
/**
 * Generic result type for operations that can fail
 */
export type Result<T, E> = {
    readonly isOk: () => boolean;
    readonly isError: () => boolean;
    readonly val: T;
    readonly err: E;
};

export namespace Result {
    export function ok<T>(value: T): Result<T, never>;
    export function error<E>(error: E): Result<never, E>;
}
```

### Error Types

```typescript
/**
 * Indexing trigger errors
 */
export enum TriggerIndexingError {
    NotSupported = 'notSupported',
    AlreadyIndexing = 'alreadyIndexing',
    AuthenticationRequired = 'authenticationRequired',
    WorkspaceTooLarge = 'workspaceTooLarge',
    NetworkError = 'networkError'
}

/**
 * Search alerts for user feedback
 */
export interface WorkspaceSearchAlert {
    readonly type: 'info' | 'warning' | 'error';
    readonly message: string;
    readonly actions?: readonly WorkspaceSearchAlertAction[];
}
```

## Telemetry Integration

### Telemetry Events

**Location**: `src/platform/workspaceChunkSearch/node/workspaceChunkSearchService.ts:364-423`

```typescript
/**
 * Strategy selection telemetry
 */
interface WorkspaceChunkSearchStrategyEvent {
    strategy: string;
    errorDiagMessage?: string;
    workspaceSearchSource: string;
    workspaceSearchCorrelationId: string;
    execTime: number;
    workspaceIndexFileCount: number;
    wasFirstSearchInWorkspace: 0 | 1;
}

/**
 * Re-ranking performance telemetry
 */
interface AdaRerankEvent {
    status: 'success' | 'error';
    workspaceSearchSource: string;
    workspaceSearchCorrelationId: string;
    execTime: number;
}
```

### Correlation Tracking

```typescript
/**
 * Telemetry correlation for tracking operations
 */
export class TelemetryCorrelationId {
    constructor(
        public readonly correlationId: string,
        public readonly callTracker: CallTracker
    );
}
```

## Extension Points

### Custom Search Strategies

To implement a custom search strategy:

```typescript
class CustomSearchStrategy implements IWorkspaceChunkSearchStrategy {
    readonly id = 'custom' as WorkspaceChunkSearchStrategyId;

    async prepareSearchWorkspace(
        telemetryInfo: TelemetryCorrelationId,
        token: CancellationToken
    ): Promise<void> {
        // Optional preparation logic
    }

    async searchWorkspace(
        sizing: StrategySearchSizing,
        query: WorkspaceChunkQueryWithEmbeddings,
        options: WorkspaceChunkSearchOptions,
        telemetryInfo: TelemetryCorrelationId,
        token: CancellationToken
    ): Promise<StrategySearchResult | undefined> {
        // Strategy implementation
        return {
            chunks: [], // Your search results
            alerts: []  // Optional user alerts
        };
    }
}
```

### Custom Embedding Computers

```typescript
class CustomEmbeddingsComputer implements IEmbeddingsComputer {
    async computeEmbeddings(
        embeddingType: EmbeddingType,
        strings: readonly string[],
        options: { inputType: 'query' | 'document' },
        token: CancellationToken
    ): Promise<Embeddings | undefined> {
        // Custom embedding computation
        return {
            embeddingType,
            values: [] // Computed embeddings
        };
    }
}
```

## Configuration

### Workspace Limits

```typescript
interface WorkspaceLimits {
    /** Maximum files for automatic embeddings indexing */
    readonly autoEmbeddingsFileLimit: number;      // 750 (standard) / 50,000 (expanded)
    
    /** Maximum files for manual embeddings indexing */
    readonly manualEmbeddingsFileLimit: number;    // 2,500 (standard) / 50,000 (expanded)
    
    /** Maximum files for TF-IDF indexing */
    readonly tfidfFileLimit: number;               // 25,000
    
    /** Maximum file size for indexing */
    readonly maxFileSize: number;                  // 1.5 MB
    
    /** Maximum chunk size in tokens */
    readonly maxChunkSize: number;                 // 250 tokens
}
```

### Performance Tuning

```typescript
interface PerformanceConfig {
    /** Maximum concurrent file processing operations */
    readonly maxConcurrentFiles: number;          // 50
    
    /** Memory cache size for embeddings */
    readonly embeddingCacheSize: number;          // 5,000
    
    /** Memory cache size for chunks */
    readonly chunkCacheSize: number;              // 10,000
    
    /** Timeout for remote search operations */
    readonly remoteSearchTimeout: number;         // 12,500ms
    
    /** Timeout for local embeddings search */
    readonly embeddingsSearchTimeout: number;    // 8,000ms
    
    /** File change debounce delay */
    readonly fileChangeDebounce: number;          // 60,000ms
}
```

This API reference provides comprehensive technical documentation for developers working with or extending the VS Code Copilot Chat indexing system.