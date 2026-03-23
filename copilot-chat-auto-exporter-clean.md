# Copilot Chat Auto-Exporter PoC - Clean Service-Based Plan

## Executive Summary

This PoC will create a VS Code extension that **automatically** captures all GitHub Copilot Chat interactions and saves them to disk **without requiring ANY user interaction**. The extension will use direct service access to `IRequestLogger` for event-driven, fully automatic capture.

**Key Constraint:** ZERO user interaction - no clicking, no opening documents, no running commands. Everything happens automatically in the background.

---

## Architecture Overview

### Service-Based Detection (Single Method)

We use **direct access** to the `IRequestLogger` service:

```
User chats with Copilot
  ↓
IRequestLogger captures request (automatic, built into Copilot)
  ↓
onDidChangeRequests event fires (automatic)
  ↓
ServiceMonitor receives notification (automatic)
  ↓
EntryProcessor converts to markdown/JSON (automatic)
  ↓
FileStorage saves to disk (automatic)
  ↓
Status bar updates (automatic)
```

**No fallbacks needed** - this is the direct source of truth.

---

##Data Format Reference

### What You'll Receive from `IRequestLogger`

The `IRequestLogger.getRequests()` method returns an array of `LoggedInfo` entries. Each entry is one of three types:

#### 1. Element Info (`ILoggedElementInfo`)

Represents a prompt element/component:

```typescript
interface ILoggedElementInfo {
  kind: LoggedInfoKind.Element;  // enum value: 0
  id: string;                     // e.g., "elem_abc123"
  name: string;                   // e.g., "SystemMessage", "UserPrompt"
  tokens: number;                 // e.g., 150
  maxTokens: number;              // e.g., 4096
  trace: HTMLTracer;              // HTML trace object for debugging
  chatRequest: ChatRequest | undefined;
  toJSON(): object;               // Returns simplified JSON representation
}
```

**JSON Output:**
```json
{
  "id": "elem_abc123",
  "kind": "element",
  "name": "SystemMessage",
  "tokens": 150,
  "maxTokens": 4096
}
```

---

#### 2. Request Info (`ILoggedRequestInfo`) - **MOST IMPORTANT**

Represents a complete chat request/response cycle:

```typescript
interface ILoggedRequestInfo {
  kind: LoggedInfoKind.Request;  // enum value: 1
  id: string;                    // e.g., "req_xyz789"
  entry: LoggedRequest;          // Full request details (see below)
  chatRequest: ChatRequest | undefined;
  toJSON(): object;              // Returns comprehensive metadata + data
}
```

**The `entry.LoggedRequest` contains:**

##### Request Types
- `ChatMLSuccess`: Successful chat completion
- `ChatMLFailure`: Failed chat request
- `ChatMLCancelation`: User canceled request
- `CompletionSuccess`: Successful completion (non-chat)
- `CompletionFailure`: Failed completion
- `MarkdownContentRequest`: Markdown content display

##### Metadata Fields
```typescript
{
  debugName: string;              // e.g., "chat/executeCommand", "inline/edit"
  startTime: Date;                // Request start timestamp
  endTime: Date;                  // Request end timestamp
  timeToFirstToken: number;       // Milliseconds to first response token
  
  chatEndpoint: {
    model: string;                // e.g., "gpt-4o", "claude-3.5-sonnet"
    modelMaxPromptTokens: number; // e.g., 128000
    urlOrRequestMetadata: string | object; // API endpoint or metadata
  };
  
  chatParams: {
    model: string;                // Model identifier
    location: string;             // e.g., "panel", "editor", "inline"
    intent: string;               // e.g., "chat", "edit", "explain"
    ourRequestId: string;         // Internal request ID
    messages: ChatMessage[];      // Array of conversation messages
    tools: FunctionTool[];        // Available tools/functions
    postOptions: {
      max_tokens: number;         // Max response tokens
      temperature: number;        // Sampling temperature
      prediction: object;         // Prediction/speculation data
      // ... more options
    };
  };
}
```

##### Request Data
```typescript
{
  messages: [
    {
      role: "system" | "user" | "assistant" | "tool",
      content: string,            // Message content
      name?: string               // Tool/function name if applicable
    }
  ],
  tools: [
    {
      type: "function",
      function: {
        name: string,             // e.g., "read_file", "search_workspace"
        description: string,
        parameters: object        // JSON schema for parameters
      }
    }
  ],
  postOptions: {
    max_tokens: number,
    temperature: number,
    top_p: number,
    presence_penalty: number,
    frequency_penalty: number,
    prediction: {
      type: "content",
      content: string             // Predicted/expected content
    }
  }
}
```

##### Response Data (for Success)
```typescript
{
  result: {
    type: "Success",
    value: string | string[],     // AI's response (single or multiple)
    requestId: string,            // Server request ID
    serverRequestId: string,      // Server-side tracking ID
  },
  usage: {
    prompt_tokens: number,        // Tokens in prompt
    completion_tokens: number,    // Tokens in completion
    total_tokens: number          // Total tokens used
  },
  deltas: [                       // Streaming response chunks (optional)
    {
      content: string,
      delta: string,
      timestamp: number
    }
  ]
}
```

##### Error Data (for Failure)
```typescript
{
  result: {
    type: "Failure" | "Canceled" | "Length",
    reason: string,               // Error message
    truncatedValue?: string       // Partial response if length-limited
  }
}
```

**Complete JSON Output Example:**
```json
{
  "id": "req_xyz789",
  "kind": "request",
  "type": "ChatMLSuccess",
  "name": "chat/executeCommand",
  "metadata": {
    "url": "https://api.githubcopilot.com/chat/completions",
    "model": "gpt-4o",
    "maxPromptTokens": 128000,
    "maxResponseTokens": 4096,
    "location": "panel",
    "intent": "chat",
    "startTime": "2025-10-23T10:30:00.000Z",
    "endTime": "2025-10-23T10:30:03.500Z",
    "duration": 3500,
    "timeToFirstToken": 450,
    "ourRequestId": "abc-123-def-456",
    "requestId": "chatcmpl-xyz789",
    "serverRequestId": "req_server_abc123",
    "usage": {
      "prompt_tokens": 1250,
      "completion_tokens": 340,
      "total_tokens": 1590
    },
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "read_file",
          "description": "Read contents of a file",
          "parameters": {
            "type": "object",
            "properties": {
              "filePath": { "type": "string" }
            }
          }
        }
      }
    ],
    "postOptions": {
      "max_tokens": 4096,
      "temperature": 0.7,
      "top_p": 0.95
    }
  },
  "messages": [
    {
      "role": "system",
      "content": "You are GitHub Copilot, an AI assistant..."
    },
    {
      "role": "user",
      "content": "How do I create a React component?"
    },
    {
      "role": "assistant",
      "content": "Here's how to create a React component:\n\n```tsx\n..."
    }
  ],
  "response": {
    "type": "success",
    "message": "Here's how to create a React component:\n\n```tsx\nimport React from 'react';\n\nfunction MyComponent() {\n  return <div>Hello World</div>;\n}\n\nexport default MyComponent;\n```"
  }
}
```

---

#### 3. Tool Call Info (`ILoggedToolCall`)

Represents a language model tool execution:

```typescript
interface ILoggedToolCall {
  kind: LoggedInfoKind.ToolCall;  // enum value: 2
  id: string;                      // e.g., "tool_def456"
  name: string;                    // Tool name (e.g., "read_file", "search")
  args: unknown;                   // Tool arguments object
  response: LanguageModelToolResult2;  // Tool execution result
  chatRequest: ChatRequest | undefined;
  time: number;                    // Execution time in milliseconds
  thinking?: ThinkingData;         // Reasoning trace if available
  toJSON(): Promise<object>;       // Returns async JSON (note: async!)
}
```

**JSON Output:**
```json
{
  "id": "tool_def456",
  "kind": "toolCall",
  "name": "read_file",
  "args": {
    "filePath": "/Users/user/project/src/App.tsx"
  },
  "time": 125,
  "response": {
    "content": "import React from 'react';\n\nfunction App() {\n  return <div>App</div>;\n}",
    "metadata": {
      "size": 256,
      "lines": 8
    }
  },
  "thinking": {
    "reasoning": "Reading the main App component to understand the structure"
  }
}
```

---

### Accessing Metadata

You can access metadata in two ways:

#### 1. Direct Property Access
```typescript
const entry: ILoggedRequestInfo = ...; // from getRequests()

// Access metadata directly
const model = entry.entry.chatEndpoint.model;
const duration = entry.entry.endTime.getTime() - entry.entry.startTime.getTime();
const tokens = entry.entry.type === 'ChatMLSuccess' ? entry.entry.usage : undefined;
const messages = entry.entry.chatParams.messages;
const intent = entry.entry.chatParams.intent;
const location = entry.entry.chatParams.location;
```

#### 2. Using toJSON() Method
```typescript
const entry: ILoggedRequestInfo = ...;

// Get structured JSON representation
const json = entry.toJSON();

// json contains all metadata in clean format
console.log(json.metadata.model);        // "gpt-4o"
console.log(json.metadata.duration);     // 3500
console.log(json.metadata.usage);        // { prompt_tokens: ..., }
console.log(json.messages);              // Array of messages
console.log(json.response);              // Response data
```

**Best Practice:** Use `toJSON()` for file output, direct property access for filtering/processing.

---

## Implementation Plan

### Phase 1: Project Setup (Day 1)

#### 1.1 Initialize Extension Project

Create a new VS Code extension:

```bash
mkdir copilot-chat-auto-exporter
cd copilot-chat-auto-exporter
npm init -y
npm install --save-dev @types/vscode @types/node typescript esbuild
```

**Project Structure:**
```
copilot-chat-auto-exporter/
├── src/
│   ├── extension.ts              # Main entry point
│   ├── serviceMonitor.ts         # IRequestLogger event listener
│   ├── entryProcessor.ts         # Convert entries to markdown/JSON
│   ├── fileStorage.ts            # Save files to disk
│   ├── deduplicator.ts           # Prevent duplicate captures
│   ├── config.ts                 # Configuration management
│   └── types.ts                  # TypeScript interfaces
├── test/
│   ├── unit.test.ts
│   └── integration.test.ts
├── package.json                  # Extension manifest
├── tsconfig.json
└── README.md
```

**package.json:**
```json
{
  "name": "copilot-chat-auto-exporter",
  "displayName": "Copilot Chat Auto-Exporter",
  "description": "Automatically export Copilot Chat interactions to disk",
  "version": "0.1.0",
  "engines": {
    "vscode": "^1.85.0"
  },
  "categories": ["Other"],
  "activationEvents": ["onStartupFinished"],
  "main": "./dist/extension.js",
  "contributes": {
    "configuration": {
      "title": "Copilot Chat Auto-Exporter",
      "properties": {
        "copilotExporter.enabled": {
          "type": "boolean",
          "default": true,
          "description": "Enable automatic export"
        },
        "copilotExporter.outputPath": {
          "type": "string",
          "default": "~/copilot-chat-exports",
          "description": "Directory to save exports"
        },
        "copilotExporter.format": {
          "type": "string",
          "enum": ["markdown", "json", "both"],
          "default": "both",
          "description": "Export format"
        }
      }
    }
  },
  "scripts": {
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "package": "esbuild ./src/extension.ts --bundle --outfile=dist/extension.js --external:vscode --format=cjs --platform=node",
    "test": "vitest"
  }
}
```

---

### Phase 2: Service Monitor

**File:** `src/serviceMonitor.ts`

**Goal:** Subscribe to `IRequestLogger` events and capture entries automatically.

**Implementation:**
```typescript
import * as vscode from 'vscode';

export class ServiceMonitor {
  private disposables: vscode.Disposable[] = [];
  private onEntriesChanged: (entries: any[]) => void;

  constructor(onEntriesChanged: (entries: any[]) => void) {
    this.onEntriesChanged = onEntriesChanged;
  }

  async start(): Promise<void> {
    try {
      // Access IRequestLogger service from GitHub Copilot extension
      const copilotExt = vscode.extensions.getExtension('GitHub.copilot');
      
      if (!copilotExt) {
        throw new Error('GitHub Copilot extension not found');
      }

      if (!copilotExt.isActive) {
        await copilotExt.activate();
      }

      // Access the internal API (this requires the extension to expose it)
      const copilotAPI = copilotExt.exports;
      
      if (!copilotAPI || !copilotAPI.requestLogger) {
        console.warn('IRequestLogger not accessible via extension API');
        // We may need to use alternative approaches or wait for official API
        return;
      }

      const requestLogger = copilotAPI.requestLogger;

      // Subscribe to onDidChangeRequests event
      this.disposables.push(
        requestLogger.onDidChangeRequests(() => {
          console.log('📝 New Copilot request detected!');
          const entries = requestLogger.getRequests();
          this.onEntriesChanged(entries);
        })
      );

      console.log('✅ Service monitor started - listening for Copilot requests');
      
      // Get initial entries
      const initialEntries = requestLogger.getRequests();
      if (initialEntries.length > 0) {
        console.log(`Found ${initialEntries.length} existing entries`);
        this.onEntriesChanged(initialEntries);
      }

    } catch (error) {
      console.error('Failed to start service monitor:', error);
      throw error;
    }
  }

  stop(): void {
    this.disposables.forEach(d => d.dispose());
    this.disposables = [];
    console.log('🛑 Service monitor stopped');
  }
}
```

**Key Points:**
- ✅ Event-driven - no polling required
- ✅ Automatic - fires on every new request
- ✅ Direct access to source of truth
- ⚠️ Requires Copilot extension to expose `IRequestLogger` in its API

**Alternative:** If Copilot doesn't expose the API, we may need to:
1. Use VS Code's dependency injection to access the service directly
2. Request Copilot team to add it to their public API
3. Use undocumented internal APIs (not recommended for production)

---

### Phase 3: Entry Processor

**File:** `src/entryProcessor.ts`

**Goal:** Convert raw `LoggedInfo` entries to markdown and JSON without opening virtual documents.

**Implementation:**
```typescript
import { LoggedInfo, ILoggedRequestInfo, ILoggedElementInfo, ILoggedToolCall } from './types';

export interface ProcessedEntry {
  markdown: string;
  json: any;
}

export class EntryProcessor {
  processEntry(entry: LoggedInfo): ProcessedEntry | null {
    if (entry.kind === 0) { // LoggedInfoKind.Element
      return this.processElement(entry as ILoggedElementInfo);
    } else if (entry.kind === 1) { // LoggedInfoKind.Request
      return this.processRequest(entry as ILoggedRequestInfo);
    } else if (entry.kind === 2) { // LoggedInfoKind.ToolCall
      return this.processToolCall(entry as ILoggedToolCall);
    }
    return null;
  }

  private processElement(entry: ILoggedElementInfo): ProcessedEntry {
    const markdown = this.generateElementMarkdown(entry);
    const json = entry.toJSON();
    return { markdown, json };
  }

  private processRequest(entry: ILoggedRequestInfo): ProcessedEntry {
    const markdown = this.generateRequestMarkdown(entry);
    const json = entry.toJSON();
    return { markdown, json };
  }

  private async processToolCall(entry: ILoggedToolCall): Promise<ProcessedEntry> {
    const markdown = this.generateToolCallMarkdown(entry);
    const json = await entry.toJSON(); // Note: async!
    return { markdown, json };
  }

  private generateRequestMarkdown(entry: ILoggedRequestInfo): string {
    const e = entry.entry;
    const lines: string[] = [];

    // Header
    lines.push(`# Copilot Request Log`);
    lines.push(`**ID:** ${entry.id}`);
    lines.push(`**Type:** ${e.type}`);
    lines.push(`**Name:** ${e.debugName}`);
    lines.push('');

    // Metadata
    lines.push(`## Metadata`);
    lines.push('');
    lines.push(`- **Model:** ${e.chatParams.model}`);
    lines.push(`- **Location:** ${e.chatParams.location}`);
    if (e.chatParams.intent) {
      lines.push(`- **Intent:** ${e.chatParams.intent}`);
    }
    lines.push(`- **Start Time:** ${e.startTime?.toISOString()}`);
    lines.push(`- **End Time:** ${e.endTime?.toISOString()}`);
    
    if (e.startTime && e.endTime) {
      const duration = e.endTime.getTime() - e.startTime.getTime();
      lines.push(`- **Duration:** ${duration}ms`);
    }

    if (e.type === 'ChatMLSuccess' && e.timeToFirstToken) {
      lines.push(`- **Time to First Token:** ${e.timeToFirstToken}ms`);
    }

    if (e.type === 'ChatMLSuccess' && e.usage) {
      lines.push('');
      lines.push(`### Token Usage`);
      lines.push(`- Prompt Tokens: ${e.usage.prompt_tokens}`);
      lines.push(`- Completion Tokens: ${e.usage.completion_tokens}`);
      lines.push(`- Total Tokens: ${e.usage.total_tokens}`);
    }

    lines.push('');

    // Request Messages
    if ('messages' in e.chatParams && e.chatParams.messages) {
      lines.push(`## Request Messages`);
      lines.push('');
      
      for (const msg of e.chatParams.messages) {
        lines.push(`### ${msg.role.toUpperCase()}`);
        lines.push('```');
        lines.push(typeof msg.content === 'string' ? msg.content : JSON.stringify(msg.content, null, 2));
        lines.push('```');
        lines.push('');
      }
    }

    // Response
    lines.push(`## Response`);
    lines.push('');
    
    if (e.type === 'ChatMLSuccess') {
      const content = Array.isArray(e.result.value) ? e.result.value.join('\n') : e.result.value;
      lines.push('```');
      lines.push(content);
      lines.push('```');
    } else if (e.type === 'CompletionSuccess') {
      lines.push('```');
      lines.push(e.result.value);
      lines.push('```');
    } else if (e.type === 'ChatMLFailure') {
      lines.push(`**Error:** ${e.result.reason}`);
      if (e.result.type === 'Length' && e.result.truncatedValue) {
        lines.push('');
        lines.push('**Truncated Response:**');
        lines.push('```');
        lines.push(e.result.truncatedValue);
        lines.push('```');
      }
    } else if (e.type === 'ChatMLCancelation') {
      lines.push('*Request was canceled*');
    }

    lines.push('');

    // Tools
    if ('tools' in e.chatParams && e.chatParams.tools && e.chatParams.tools.length > 0) {
      lines.push(`## Available Tools`);
      lines.push('');
      for (const tool of e.chatParams.tools) {
        lines.push(`### ${tool.function.name}`);
        lines.push(tool.function.description);
        lines.push('```json');
        lines.push(JSON.stringify(tool.function.parameters, null, 2));
        lines.push('```');
        lines.push('');
      }
    }

    return lines.join('\n');
  }

  private generateElementMarkdown(entry: ILoggedElementInfo): string {
    return `# Prompt Element: ${entry.name}\n\n` +
           `**ID:** ${entry.id}\n` +
           `**Tokens:** ${entry.tokens} / ${entry.maxTokens}\n`;
  }

  private generateToolCallMarkdown(entry: ILoggedToolCall): string {
    const lines: string[] = [];
    
    lines.push(`# Tool Call: ${entry.name}`);
    lines.push(`**ID:** ${entry.id}`);
    lines.push(`**Execution Time:** ${entry.time}ms`);
    lines.push('');
    
    lines.push(`## Arguments`);
    lines.push('```json');
    lines.push(JSON.stringify(entry.args, null, 2));
    lines.push('```');
    lines.push('');
    
    lines.push(`## Response`);
    lines.push('```json');
    lines.push(JSON.stringify(entry.response, null, 2));
    lines.push('```');
    
    if (entry.thinking) {
      lines.push('');
      lines.push(`## Thinking/Reasoning`);
      lines.push('```json');
      lines.push(JSON.stringify(entry.thinking, null, 2));
      lines.push('```');
    }
    
    return lines.join('\n');
  }
}
```

**Key Points:**
- ✅ Processes entries directly without opening virtual documents
- ✅ Generates markdown independently of Copilot's renderer
- ✅ Extracts all metadata from entry objects
- ✅ Handles all three entry types
- ✅ Full control over output format

---

### Phase 4: File Storage

**File:** `src/fileStorage.ts`

**Goal:** Save processed entries to disk with organized naming.

**Implementation:**
```typescript
import * as vscode from 'vscode';
import * as fs from 'fs/promises';
import * as path from 'path';
import * as os from 'os';
import { ProcessedEntry } from './entryProcessor';

export interface StorageConfig {
  basePath: string;
  format: 'markdown' | 'json' | 'both';
}

export class FileStorage {
  private config: StorageConfig;

  constructor(config: StorageConfig) {
    this.config = config;
  }

  async initialize(): Promise<void> {
    const basePath = this.config.basePath.replace('~', os.homedir());
    await fs.mkdir(basePath, { recursive: true });
    console.log(`📁 Storage initialized at: ${basePath}`);
  }

  async saveEntry(
    entryId: string,
    entryKind: string,
    processed: ProcessedEntry
  ): Promise<void> {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const baseName = `${timestamp}_${entryKind}_${entryId}`;
    const basePath = this.config.basePath.replace('~', os.homedir());

    if (this.config.format === 'markdown' || this.config.format === 'both') {
      const mdPath = path.join(basePath, `${baseName}.md`);
      await fs.writeFile(mdPath, processed.markdown, 'utf-8');
      console.log(`💾 Saved markdown: ${mdPath}`);
    }

    if (this.config.format === 'json' || this.config.format === 'both') {
      const jsonPath = path.join(basePath, `${baseName}.json`);
      await fs.writeFile(jsonPath, JSON.stringify(processed.json, null, 2), 'utf-8');
      console.log(`💾 Saved JSON: ${jsonPath}`);
    }
  }
}
```

---

### Phase 5: Deduplication

**File:** `src/deduplicator.ts`

**Goal:** Prevent capturing the same entry multiple times.

**Implementation:**
```typescript
import * as crypto from 'crypto';

export class Deduplicator {
  private seen = new Map<string, string>(); // id -> content hash

  isDuplicate(id: string, content: string): boolean {
    const hash = this.hashContent(content);
    const existing = this.seen.get(id);

    if (existing === hash) {
      return true; // Exact duplicate
    }

    this.seen.set(id, hash);
    return false;
  }

  private hashContent(content: string): string {
    return crypto.createHash('sha256').update(content).digest('hex');
  }

  clear(): void {
    this.seen.clear();
  }
}
```

---

### Phase 6: Main Extension Logic

**File:** `src/extension.ts`

**Goal:** Wire everything together and handle lifecycle.

**Implementation:**
```typescript
import * as vscode from 'vscode';
import { ServiceMonitor } from './serviceMonitor';
import { EntryProcessor, ProcessedEntry } from './entryProcessor';
import { FileStorage, StorageConfig } from './fileStorage';
import { Deduplicator } from './deduplicator';
import { LoggedInfo } from './types';

let statusBar: vscode.StatusBarItem;
let captureCount = 0;

export async function activate(context: vscode.ExtensionContext) {
  console.log('🚀 Copilot Chat Auto-Exporter activating...');

  // Create status bar
  statusBar = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 100);
  statusBar.text = "$(cloud-download) Copilot: 0 captured";
  statusBar.show();
  context.subscriptions.push(statusBar);

  // Get configuration
  const config = vscode.workspace.getConfiguration('copilotExporter');
  const enabled = config.get<boolean>('enabled', true);
  
  if (!enabled) {
    console.log('❌ Auto-exporter disabled in settings');
    return;
  }

  const outputPath = config.get<string>('outputPath', '~/copilot-chat-exports');
  const format = config.get<'markdown' | 'json' | 'both'>('format', 'both');

  // Initialize components
  const deduplicator = new Deduplicator();
  const processor = new EntryProcessor();
  const storage = new FileStorage({ basePath: outputPath, format });

  await storage.initialize();

  // Start service monitor
  const monitor = new ServiceMonitor(handleEntriesChanged);
  
  try {
    await monitor.start();
  } catch (error) {
    vscode.window.showErrorMessage(
      `Failed to start Copilot Chat Auto-Exporter: ${error}`
    );
    return;
  }

  context.subscriptions.push({ dispose: () => monitor.stop() });

  // Handle new entries
  function handleEntriesChanged(entries: LoggedInfo[]): void {
    console.log(`📨 Received ${entries.length} entries`);

    for (const entry of entries) {
      processEntryDirectly(entry);
    }
  }

  async function processEntryDirectly(entry: LoggedInfo): Promise<void> {
    const entryId = entry.id;
    const entryKind = entry.kind === 0 ? 'element' : entry.kind === 1 ? 'request' : 'toolcall';

    // Deduplicate
    const contentForHash = JSON.stringify(entry);
    if (deduplicator.isDuplicate(entryId, contentForHash)) {
      console.log(`⏭️  Skipping duplicate: ${entryId}`);
      return;
    }

    try {
      // Process entry
      const processed = processor.processEntry(entry);
      if (!processed) {
        console.log(`⏭️  Skipping unsupported entry type: ${entryKind}`);
        return;
      }

      // Save to disk
      await storage.saveEntry(entryId, entryKind, processed);

      // Update UI
      captureCount++;
      updateStatusBar();

      console.log(`✅ AUTO-CAPTURED: ${entryKind} - ${entryId}`);
    } catch (error) {
      console.error(`❌ Error processing entry ${entryId}:`, error);
      vscode.window.showErrorMessage(`Failed to capture Copilot log: ${error}`);
    }
  }

  function updateStatusBar(): void {
    statusBar.text = `$(cloud-download) Copilot: ${captureCount} captured`;
  }

  console.log('✅ Copilot Chat Auto-Exporter activated');
}

export function deactivate() {
  console.log('👋 Copilot Chat Auto-Exporter deactivated');
}
```

---

## Testing Strategy

### Unit Tests

**File:** `test/unit.test.ts`

```typescript
import { describe, it, expect } from 'vitest';
import { Deduplicator } from '../src/deduplicator';
import { EntryProcessor } from '../src/entryProcessor';

describe('Deduplicator', () => {
  it('should detect duplicate entries', () => {
    const dedup = new Deduplicator();
    expect(dedup.isDuplicate('entry1', 'content1')).toBe(false);
    expect(dedup.isDuplicate('entry1', 'content1')).toBe(true);
  });

  it('should allow different content for same entry ID', () => {
    const dedup = new Deduplicator();
    dedup.isDuplicate('entry1', 'content1');
    expect(dedup.isDuplicate('entry1', 'content2')).toBe(false);
  });
});

describe('EntryProcessor', () => {
  it('should process request entries', () => {
    const processor = new EntryProcessor();
    const mockEntry = createMockRequestEntry();
    
    const result = processor.processEntry(mockEntry);
    expect(result).toBeDefined();
    expect(result.markdown).toContain('# Copilot Request Log');
    expect(result.json).toHaveProperty('id');
  });
});
```

### Integration Tests

Test the full pipeline with mocked Copilot data.

### Manual Tests

1. Extension activates without errors
2. Status bar shows "0 captured"
3. Chat with Copilot in panel
4. Verify file created in `~/copilot-chat-exports/`
5. Check markdown format
6. Check JSON format
7. Verify status bar updates
8. Test deduplication
9. Test configuration changes
10. Test extension reload

---

## Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Project Setup | Day 1 | Project structure, package.json |
| Service Monitor | Day 1-2 | Event listener working |
| Entry Processor | Day 2 | Markdown/JSON generation |
| File Storage | Day 2 | Saving to disk |
| Main Extension | Day 3 | Full integration |
| Testing | Day 3-4 | All tests passing |
| Documentation | Day 4 | Complete docs |

**Total:** 4 days

---

## Key Benefits of This Approach

### ✅ Fully Automatic
- User installs extension
- User continues chatting with Copilot normally
- Everything else happens in background

### ✅ No User Interaction Required
- ❌ No clicking tree view items
- ❌ No opening virtual documents
- ❌ No running export commands
- ❌ No manual file saving

### ✅ Reliable
- Direct access to source of truth (`IRequestLogger`)
- Event-driven, not polling
- No dependency on UI state

### ✅ Complete Data Access
- Full request/response data
- All metadata (tokens, timing, model, etc.)
- Tool calls captured
- Error information preserved

### ✅ Flexible Output
- Markdown for human reading
- JSON for programmatic analysis
- Configurable paths
- Organized file naming

---

## Expected User Experience

1. **Install** extension from VSIX or marketplace
2. **Use** Copilot normally (no behavior change)
3. **Check** `~/copilot-chat-exports/` anytime to see captured logs
4. **Enjoy** automatic capture with zero effort

**That's it!** No other steps required.

---

## Notes & Limitations

### Current Limitation: API Access

The `IRequestLogger` service is internal to the Copilot extension and may not be exposed via its public API. Solutions:

1. **Request Official API**: Ask GitHub Copilot team to expose `IRequestLogger` in their extension API
2. **Use VS Code Extension Host DI**: Access the service through VS Code's dependency injection (advanced, may break)
3. **Wait for Official Support**: GitHub may add official export functionality

### Workaround for PoC

For the PoC, we can:
- Use TypeScript declaration merging to access internal APIs
- Document that this is a proof-of-concept requiring Copilot cooperation
- Demonstrate the value to encourage official API support

### Future Enhancements

- Filter by model, location, or intent
- Search across captured logs
- Statistics dashboard
- Cloud sync option
- Encryption for sensitive data
- Custom export templates
