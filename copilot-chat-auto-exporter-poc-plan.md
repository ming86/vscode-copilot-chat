# Copilot Chat Auto-Exporter - PoC Development Plan

## Project Overview

**Goal:** Develop a VS Code extension that automatically detects and retrieves Copilot Chat request data in real-time without any manual user interaction.

**Key Requirements:**
- ✅ Fully automatic - zero user intervention
- ✅ Real-time detection when data becomes available
- ✅ Capture all request/response data
- ✅ Persist data to filesystem
- ✅ Non-intrusive to user workflow

---

## Architecture Strategy

### Detection Approach

We will use **multiple detection mechanisms** to ensure complete coverage:

1. **IRequestLogger Service Hook** (Primary - No User Action Required!)
   - Access Copilot's `IRequestLogger` service directly via DI container
   - Subscribe to `onDidChangeRequests` event
   - Get raw `LoggedInfo[]` entries from `getRequests()` immediately
   - Convert entries to markdown/JSON ourselves (no virtual docs needed!)

2. **Direct Entry Processing** (Secondary - Bypass Virtual Documents)
   - Process `LoggedRequestInfo`, `LoggedToolCall`, `LoggedElementInfo` directly
   - Extract request/response data from entry objects
   - Render to markdown using same logic as `_renderRequestToMarkdown()`
   - Save to disk automatically - **NO USER CLICKS REQUIRED**

3. **Output Channel Monitoring** (Fallback)
   - Watch VS Code's output channel logs for URIs
   - Parse `ccreq:` URIs from logs
   - Used only if service access fails

4. **Proxy Text Document Provider** (Backup - Still Automatic)
   - Intercept `provideTextDocumentContent` calls
   - Capture content generation even if user never opens files
   - Hook into internal rendering pipeline

---

## PoC Implementation Plan

### Phase 1: Project Setup (Day 1)

#### 1.1 Initialize Extension Project

**Tasks:**
- Create new VS Code extension project
- Set up TypeScript configuration
- Configure build system (esbuild or webpack)
- Set up development environment

**Files to create:**
```
copilot-chat-auto-exporter/
├── package.json
├── tsconfig.json
├── src/
│   ├── extension.ts
│   ├── monitors/
│   │   ├── documentMonitor.ts
│   │   ├── outputChannelMonitor.ts
│   │   └── serviceMonitor.ts
│   ├── storage/
│   │   ├── fileStorage.ts
│   │   └── dataSerializer.ts
│   └── utils/
│       ├── uriParser.ts
│       └── logger.ts
└── README.md
```

**package.json configuration:**
```json
{
  "name": "copilot-chat-auto-exporter",
  "displayName": "Copilot Chat Auto Exporter",
  "description": "Automatically captures and exports Copilot Chat requests",
  "version": "0.1.0",
  "engines": {
    "vscode": "^1.85.0"
  },
  "categories": ["Other"],
  "extensionDependencies": [
    "GitHub.copilot-chat"
  ],
  "activationEvents": [
    "onStartupFinished"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "configuration": {
      "title": "Copilot Chat Auto Exporter",
      "properties": {
        "copilotAutoExporter.enabled": {
          "type": "boolean",
          "default": true,
          "description": "Enable automatic export of Copilot Chat logs"
        },
        "copilotAutoExporter.exportPath": {
          "type": "string",
          "default": "~/.vscode/copilot-chat-logs",
          "description": "Directory to store exported logs"
        },
        "copilotAutoExporter.exportFormat": {
          "type": "string",
          "enum": ["markdown", "json", "both"],
          "default": "both",
          "description": "Export format for logs"
        },
        "copilotAutoExporter.includeMetadata": {
          "type": "boolean",
          "default": true,
          "description": "Include metadata in exported logs"
        }
      }
    },
    "commands": [
      {
        "command": "copilotAutoExporter.openExportFolder",
        "title": "Open Copilot Logs Folder"
      },
      {
        "command": "copilotAutoExporter.clearLogs",
        "title": "Clear Copilot Logs"
      },
      {
        "command": "copilotAutoExporter.showStatus",
        "title": "Show Copilot Export Status"
      }
    ]
  }
}
```

---

### Phase 2: Core Detection System (Day 1-2)

#### 2.1 Virtual Document Monitor

**File:** `src/monitors/documentMonitor.ts`

**Implementation:**
```typescript
import * as vscode from 'vscode';

export class DocumentMonitor {
  private disposables: vscode.Disposable[] = [];
  private onDocumentCaptured: (uri: vscode.Uri, content: string) => void;

  constructor(onCapture: (uri: vscode.Uri, content: string) => void) {
    this.onDocumentCaptured = onCapture;
  }

  public start(): void {
    // Monitor document opens
    this.disposables.push(
      vscode.workspace.onDidOpenTextDocument(doc => {
        if (doc.uri.scheme === 'ccreq') {
          this.captureDocument(doc);
        }
      })
    );

    // Monitor document changes (for streaming responses)
    this.disposables.push(
      vscode.workspace.onDidChangeTextDocument(e => {
        if (e.document.uri.scheme === 'ccreq') {
          this.captureDocument(e.document);
        }
      })
    );

    // Check all currently open documents
    for (const doc of vscode.workspace.textDocuments) {
      if (doc.uri.scheme === 'ccreq') {
        this.captureDocument(doc);
      }
    }
  }

  private captureDocument(doc: vscode.TextDocument): void {
    const uri = doc.uri;
    const content = doc.getText();
    
    if (content && content.length > 0) {
      this.onDocumentCaptured(uri, content);
    }
  }

  public stop(): void {
    this.disposables.forEach(d => d.dispose());
    this.disposables = [];
  }
}
```

**Testing Strategy:**
- Open Copilot Chat panel
- Send a chat message
- Verify monitor detects virtual document
- Verify content is captured

---

#### 2.2 Output Channel Monitor

**File:** `src/monitors/outputChannelMonitor.ts`

**Implementation:**
```typescript
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';

export class OutputChannelMonitor {
  private watcher: fs.FSWatcher | undefined;
  private onUriDetected: (uri: string) => void;
  private lastPosition = 0;

  constructor(onUriFound: (uri: string) => void) {
    this.onUriDetected = onUriFound;
  }

  public async start(): Promise<void> {
    const outputDir = path.join(os.homedir(), '.vscode', 'output');
    
    // Find Copilot Chat output channel directory
    const channelDir = await this.findCopilotChannelDir(outputDir);
    if (!channelDir) {
      console.warn('Copilot output channel not found');
      return;
    }

    const logFile = path.join(channelDir, 'log.txt');
    
    // Watch for changes
    this.watcher = fs.watch(logFile, (eventType) => {
      if (eventType === 'change') {
        this.processNewContent(logFile);
      }
    });

    // Process existing content
    this.processNewContent(logFile);
  }

  private async findCopilotChannelDir(outputDir: string): Promise<string | null> {
    try {
      const dirs = await fs.promises.readdir(outputDir);
      for (const dir of dirs) {
        if (dir.toLowerCase().includes('github') || 
            dir.toLowerCase().includes('copilot')) {
          return path.join(outputDir, dir);
        }
      }
    } catch (err) {
      console.error('Error finding output dir:', err);
    }
    return null;
  }

  private processNewContent(logFile: string): void {
    try {
      const stats = fs.statSync(logFile);
      const stream = fs.createReadStream(logFile, {
        start: this.lastPosition,
        encoding: 'utf-8'
      });

      let buffer = '';
      stream.on('data', (chunk) => {
        buffer += chunk;
      });

      stream.on('end', () => {
        this.lastPosition = stats.size;
        this.extractUris(buffer);
      });
    } catch (err) {
      console.error('Error reading log file:', err);
    }
  }

  private extractUris(content: string): void {
    const uriPattern = /ccreq:[^\s]+\.(copilotmd|json|request\.json)/g;
    const matches = content.matchAll(uriPattern);
    
    for (const match of matches) {
      this.onUriDetected(match[0]);
    }
  }

  public stop(): void {
    if (this.watcher) {
      this.watcher.close();
      this.watcher = undefined;
    }
  }
}
```

**Testing Strategy:**
- Monitor output channel file
- Send chat messages
- Verify URIs are detected in real-time
- Verify no duplicates are captured

---

#### 2.3 Service Monitor (PRIMARY - Direct Access, No User Action!)

**File:** `src/monitors/serviceMonitor.ts`

**This is the KEY to automatic capture without user interaction!**

**Implementation:**
```typescript
import * as vscode from 'vscode';

// Import types from Copilot Chat (if extension exports them)
interface IRequestLogger {
  onDidChangeRequests: vscode.Event<void>;
  getRequests(): LoggedInfo[];
}

interface LoggedInfo {
  id: string;
  kind: 'Element' | 'Request' | 'ToolCall';
  chatRequest?: any;
}

interface ILoggedRequestInfo extends LoggedInfo {
  kind: 'Request';
  entry: any; // LoggedRequest with all the data
}

export class ServiceMonitor {
  private disposable: vscode.Disposable | undefined;
  private requestLogger: IRequestLogger | undefined;
  private onEntriesChanged: (entries: LoggedInfo[]) => void;
  private processedIds = new Set<string>();

  constructor(onChanged: (entries: LoggedInfo[]) => void) {
    this.onEntriesChanged = onChanged;
  }

  public async start(): Promise<boolean> {
    try {
      // Method 1: Try to access via extension API
      const copilotExt = vscode.extensions.getExtension('GitHub.copilot-chat');
      
      if (copilotExt) {
        if (!copilotExt.isActive) {
          await copilotExt.activate();
        }

        const api = copilotExt.exports;
        if (api && api.requestLogger) {
          this.requestLogger = api.requestLogger;
          return this.setupMonitoring();
        }
      }

      // Method 2: Try to access via VS Code's service container
      // This might work if Copilot exposes its services
      // We'd need to access the DI container somehow
      
      return false;
    } catch (err) {
      console.error('Error accessing service:', err);
      return false;
    }
  }

  private setupMonitoring(): boolean {
    if (!this.requestLogger) {
      return false;
    }

    // Subscribe to changes - fires EVERY time a new request is logged!
    this.disposable = this.requestLogger.onDidChangeRequests(() => {
      this.processNewEntries();
    });

    // Process any existing entries
    this.processNewEntries();

    return true;
  }

  private processNewEntries(): void {
    if (!this.requestLogger) {
      return;
    }

    const allEntries = this.requestLogger.getRequests();
    const newEntries: LoggedInfo[] = [];

    // Find entries we haven't processed yet
    for (const entry of allEntries) {
      if (!this.processedIds.has(entry.id)) {
        this.processedIds.add(entry.id);
        newEntries.push(entry);
      }
    }

    if (newEntries.length > 0) {
      console.log(`ServiceMonitor: ${newEntries.length} new entries detected`);
      this.onEntriesChanged(newEntries);
    }
  }

  public stop(): void {
    if (this.disposable) {
      this.disposable.dispose();
      this.disposable = undefined;
    }
  }
}
```

**Why This Works WITHOUT User Action:**

1. **Event-Driven:** `onDidChangeRequests` fires automatically when Copilot logs any request
2. **Immediate Access:** `getRequests()` gives us the raw entry objects with ALL data
3. **No Virtual Docs:** We process `LoggedRequestInfo` directly, never need to open files
4. **Real-Time:** Captures the instant Copilot creates the log entry
5. **Complete Data:** Entry objects contain request params, response, metadata, everything!

**Testing Strategy:**
- Verify service access on extension activation
- Send chat message and confirm event fires automatically
- Verify no user interaction needed
- Confirm all data is captured from entries

---

### Phase 2.5: Entry Processor (CRITICAL - Converts Raw Data to Markdown)

**File:** `src/processors/entryProcessor.ts`

**This is how we avoid needing virtual documents!**

**Implementation:**
```typescript
import { LoggedInfo, LoggedInfoKind, ILoggedRequestInfo, ILoggedToolCall } from '../types';

export class EntryProcessor {
  
  /**
   * Converts a LoggedInfo entry directly to markdown
   * This replicates what _renderRequestToMarkdown() does in Copilot
   * But WE control it and can capture the output immediately!
   */
  public processEntry(entry: LoggedInfo): { markdown: string; json: any } | null {
    switch (entry.kind) {
      case 'Request':
        return this.processRequest(entry as ILoggedRequestInfo);
      case 'ToolCall':
        return this.processToolCall(entry as ILoggedToolCall);
      case 'Element':
        // Elements are HTML, we'll skip or handle differently
        return null;
      default:
        return null;
    }
  }

  private processRequest(entry: ILoggedRequestInfo): { markdown: string; json: any } {
    const logEntry = entry.entry;
    const markdown: string[] = [];

    // Header
    markdown.push(`# ${logEntry.debugName} - ${entry.id}`);
    markdown.push('');

    // Metadata section
    markdown.push('## Metadata');
    markdown.push('```');
    markdown.push(`model            : ${logEntry.chatEndpoint?.model || 'unknown'}`);
    markdown.push(`maxPromptTokens  : ${logEntry.chatEndpoint?.modelMaxPromptTokens || 'unknown'}`);
    
    if (logEntry.type === 'ChatMLSuccess' || logEntry.type === 'CompletionSuccess') {
      const startTime = logEntry.startTime.toISOString();
      const endTime = logEntry.endTime.toISOString();
      const duration = logEntry.endTime.getTime() - logEntry.startTime.getTime();
      
      markdown.push(`startTime        : ${startTime}`);
      markdown.push(`endTime          : ${endTime}`);
      markdown.push(`duration         : ${duration}ms`);
      
      if (logEntry.type === 'ChatMLSuccess' && logEntry.usage) {
        markdown.push(`usage            : ${JSON.stringify(logEntry.usage)}`);
      }
    }
    markdown.push('```');
    markdown.push('');

    // Request Messages
    if ('messages' in logEntry.chatParams && logEntry.chatParams.messages) {
      markdown.push('## Request Messages');
      for (const msg of logEntry.chatParams.messages) {
        markdown.push(`### ${msg.role}`);
        markdown.push('```markdown');
        markdown.push(this.messageToString(msg));
        markdown.push('```');
        markdown.push('');
      }
    }

    // Response
    markdown.push('## Response');
    if (logEntry.type === 'ChatMLSuccess') {
      markdown.push('### Assistant');
      markdown.push('```markdown');
      if (logEntry.deltas && logEntry.deltas.length > 0) {
        // Process streaming deltas
        const responseText = logEntry.deltas
          .map(d => d.text || '')
          .join('');
        markdown.push(responseText);
      } else if (logEntry.result.value) {
        markdown.push(Array.isArray(logEntry.result.value) 
          ? logEntry.result.value.join('\n') 
          : logEntry.result.value);
      }
      markdown.push('```');
    } else if (logEntry.type === 'ChatMLFailure') {
      markdown.push(`**FAILED:** ${logEntry.result.reason || 'Unknown error'}`);
    } else if (logEntry.type === 'ChatMLCancelation') {
      markdown.push('**CANCELED**');
    }

    // Build JSON representation
    const json = {
      id: entry.id,
      kind: 'request',
      debugName: logEntry.debugName,
      type: logEntry.type,
      model: logEntry.chatEndpoint?.model,
      startTime: logEntry.startTime?.toISOString(),
      endTime: logEntry.endTime?.toISOString(),
      duration: logEntry.endTime && logEntry.startTime 
        ? logEntry.endTime.getTime() - logEntry.startTime.getTime() 
        : null,
      usage: logEntry.type === 'ChatMLSuccess' ? logEntry.usage : null,
      request: 'messages' in logEntry.chatParams ? logEntry.chatParams.messages : null,
      response: logEntry.type === 'ChatMLSuccess' ? logEntry.result.value : null
    };

    return {
      markdown: markdown.join('\n'),
      json
    };
  }

  private processToolCall(entry: ILoggedToolCall): { markdown: string; json: any } {
    const markdown: string[] = [];
    
    markdown.push(`# Tool Call: ${entry.name} - ${entry.id}`);
    markdown.push('');
    markdown.push('## Arguments');
    markdown.push('```json');
    markdown.push(JSON.stringify(entry.args, null, 2));
    markdown.push('```');
    markdown.push('');
    markdown.push('## Response');
    markdown.push('```');
    markdown.push(JSON.stringify(entry.response, null, 2));
    markdown.push('```');

    const json = {
      id: entry.id,
      kind: 'toolCall',
      name: entry.name,
      args: entry.args,
      response: entry.response,
      timestamp: entry.time
    };

    return {
      markdown: markdown.join('\n'),
      json
    };
  }

  private messageToString(msg: any): string {
    if (typeof msg.content === 'string') {
      return msg.content;
    } else if (Array.isArray(msg.content)) {
      return msg.content
        .map(part => {
          if (typeof part === 'string') return part;
          if (part.type === 'text') return part.text;
          if (part.type === 'image_url') return `[Image: ${part.image_url?.url}]`;
          return JSON.stringify(part);
        })
        .join('\n');
    }
    return JSON.stringify(msg.content);
  }
}
```

**Why This Is Crucial:**

1. **No Virtual Documents:** Processes raw entry objects directly
2. **Immediate Capture:** Runs as soon as entry is available
3. **No User Action:** Completely automatic, triggered by events
4. **Full Control:** We decide what to capture and how to format it
5. **Flexible Output:** Can generate markdown, JSON, or any format

---

### Phase 3: Data Storage System (Day 2)

#### 3.1 File Storage Manager

**File:** `src/storage/fileStorage.ts`

**Implementation:**
```typescript
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';
import * as os from 'os';

export interface StorageConfig {
  basePath: string;
  format: 'markdown' | 'json' | 'both';
  includeMetadata: boolean;
}

export class FileStorage {
  private config: StorageConfig;

  constructor(config: StorageConfig) {
    this.config = config;
    this.ensureDirectory();
  }

  private ensureDirectory(): void {
    const expandedPath = this.config.basePath.replace('~', os.homedir());
    if (!fs.existsSync(expandedPath)) {
      fs.mkdirSync(expandedPath, { recursive: true });
    }
  }

  public async saveDocument(
    uri: vscode.Uri, 
    content: string
  ): Promise<string[]> {
    const savedFiles: string[] = [];
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const uriId = this.extractIdFromUri(uri);
    const baseName = `${timestamp}_${uriId}`;
    const basePath = this.config.basePath.replace('~', os.homedir());

    // Save as markdown
    if (this.config.format === 'markdown' || this.config.format === 'both') {
      const mdPath = path.join(basePath, `${baseName}.md`);
      await fs.promises.writeFile(mdPath, content, 'utf-8');
      savedFiles.push(mdPath);
    }

    // Save as JSON
    if (this.config.format === 'json' || this.config.format === 'both') {
      const jsonPath = path.join(basePath, `${baseName}.json`);
      const jsonData = {
        uri: uri.toString(),
        timestamp: timestamp,
        content: content,
        metadata: this.config.includeMetadata ? this.extractMetadata(content) : null
      };
      await fs.promises.writeFile(
        jsonPath, 
        JSON.stringify(jsonData, null, 2), 
        'utf-8'
      );
      savedFiles.push(jsonPath);
    }

    return savedFiles;
  }

  private extractIdFromUri(uri: vscode.Uri): string {
    const match = uri.toString().match(/ccreq:([^.]+)/);
    return match ? match[1] : 'unknown';
  }

  private extractMetadata(content: string): any {
    const metadata: any = {};
    
    // Extract model
    const modelMatch = content.match(/model\s*:\s*(.+)/);
    if (modelMatch) metadata.model = modelMatch[1].trim();
    
    // Extract duration
    const durationMatch = content.match(/duration\s*:\s*(\d+)ms/);
    if (durationMatch) metadata.durationMs = parseInt(durationMatch[1]);
    
    // Extract tokens
    const tokensMatch = content.match(/usage\s*:\s*({.+})/);
    if (tokensMatch) {
      try {
        metadata.usage = JSON.parse(tokensMatch[1]);
      } catch (e) {
        // Ignore parsing errors
      }
    }
    
    return metadata;
  }

  public async getStoredFiles(): Promise<string[]> {
    const basePath = this.config.basePath.replace('~', os.homedir());
    const files = await fs.promises.readdir(basePath);
    return files.map(f => path.join(basePath, f));
  }

  public async clearLogs(): Promise<number> {
    const files = await this.getStoredFiles();
    let count = 0;
    
    for (const file of files) {
      if (file.endsWith('.md') || file.endsWith('.json')) {
        await fs.promises.unlink(file);
        count++;
      }
    }
    
    return count;
  }
}
```

**Testing Strategy:**
- Save test documents
- Verify file creation
- Verify metadata extraction
- Test clear functionality

---

#### 3.2 Deduplication System

**File:** `src/storage/deduplicator.ts`

**Implementation:**
```typescript
import * as crypto from 'crypto';

export class Deduplicator {
  private seenHashes = new Set<string>();
  private seenUris = new Set<string>();

  public isDuplicate(uri: string, content: string): boolean {
    // Check by URI
    if (this.seenUris.has(uri)) {
      return true;
    }

    // Check by content hash
    const hash = this.hashContent(content);
    if (this.seenHashes.has(hash)) {
      return true;
    }

    // Mark as seen
    this.seenUris.add(uri);
    this.seenHashes.add(hash);
    
    return false;
  }

  private hashContent(content: string): string {
    return crypto
      .createHash('sha256')
      .update(content)
      .digest('hex');
  }

  public clear(): void {
    this.seenHashes.clear();
    this.seenUris.clear();
  }

  public getStats(): { uris: number; hashes: number } {
    return {
      uris: this.seenUris.size,
      hashes: this.seenHashes.size
    };
  }
}
```

---

### Phase 4: Main Extension Logic (Day 3)

#### 4.1 Extension Entry Point

**File:** `src/extension.ts`

**Implementation:**
```typescript
import * as vscode from 'vscode';
import { DocumentMonitor } from './monitors/documentMonitor';
import { OutputChannelMonitor } from './monitors/outputChannelMonitor';
import { ServiceMonitor } from './monitors/serviceMonitor';
import { FileStorage, StorageConfig } from './storage/fileStorage';
import { Deduplicator } from './storage/deduplicator';

let documentMonitor: DocumentMonitor;
let outputMonitor: OutputChannelMonitor;
let serviceMonitor: ServiceMonitor;
let storage: FileStorage;
let deduplicator: Deduplicator;
let statusBarItem: vscode.StatusBarItem;
let captureCount = 0;

export async function activate(context: vscode.ExtensionContext) {
  console.log('Copilot Chat Auto Exporter activating...');

  // Initialize components
  deduplicator = new Deduplicator();
  storage = new FileStorage(getStorageConfig());

  // Create status bar
  statusBarItem = vscode.window.createStatusBarItem(
    vscode.StatusBarAlignment.Right, 
    100
  );
  statusBarItem.text = '$(copilot) 0 logs';
  statusBarItem.tooltip = 'Copilot Chat Auto Exporter';
  statusBarItem.command = 'copilotAutoExporter.showStatus';
  statusBarItem.show();
  context.subscriptions.push(statusBarItem);

  // Initialize monitors
  documentMonitor = new DocumentMonitor(handleDocumentCapture);
  outputMonitor = new OutputChannelMonitor(handleUriDetected);
  serviceMonitor = new ServiceMonitor(handleEntriesChanged);

  // Start all monitors
  documentMonitor.start();
  await outputMonitor.start();
  const serviceAvailable = await serviceMonitor.start();

  console.log(`Service monitor: ${serviceAvailable ? 'active' : 'not available'}`);

  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand('copilotAutoExporter.openExportFolder', openExportFolder),
    vscode.commands.registerCommand('copilotAutoExporter.clearLogs', clearLogs),
    vscode.commands.registerCommand('copilotAutoExporter.showStatus', showStatus)
  );

  // Watch for config changes
  context.subscriptions.push(
    vscode.workspace.onDidChangeConfiguration(e => {
      if (e.affectsConfiguration('copilotAutoExporter')) {
        storage = new FileStorage(getStorageConfig());
      }
    })
  );

  vscode.window.showInformationMessage(
    'Copilot Chat Auto Exporter is now active!'
  );
}

function getStorageConfig(): StorageConfig {
  const config = vscode.workspace.getConfiguration('copilotAutoExporter');
  return {
    basePath: config.get('exportPath', '~/.vscode/copilot-chat-logs'),
    format: config.get('exportFormat', 'both'),
    includeMetadata: config.get('includeMetadata', true)
  };
}

async function handleDocumentCapture(uri: vscode.Uri, content: string): Promise<void> {
  const uriString = uri.toString();
  
  if (deduplicator.isDuplicate(uriString, content)) {
    console.log(`Skipping duplicate: ${uriString}`);
    return;
  }

  try {
    const savedFiles = await storage.saveDocument(uri, content);
    captureCount++;
    updateStatusBar();
    
    console.log(`Captured: ${uriString}`);
    console.log(`Saved to: ${savedFiles.join(', ')}`);
  } catch (err) {
    console.error('Error saving document:', err);
    vscode.window.showErrorMessage(`Failed to save log: ${err}`);
  }
}

async function handleUriDetected(uri: string): Promise<void> {
  // Try to open the URI to get its content
  try {
    const vscodeUri = vscode.Uri.parse(uri);
    const doc = await vscode.workspace.openTextDocument(vscodeUri);
    await handleDocumentCapture(vscodeUri, doc.getText());
  } catch (err) {
    console.error(`Error opening URI ${uri}:`, err);
  }
}

function handleEntriesChanged(entries: any[]): void {
  console.log(`Service reported ${entries.length} new entries`);
  
  // Process entries directly - NO NEED TO OPEN VIRTUAL DOCUMENTS!
  for (const entry of entries) {
    processEntryDirectly(entry);
  }
}

async function processEntryDirectly(entry: any): Promise<void> {
  const entryId = `${entry.id}_${entry.kind}`;
  
  if (deduplicator.isDuplicate(entryId, JSON.stringify(entry))) {
    console.log(`Skipping duplicate entry: ${entryId}`);
    return;
  }

  try {
    // Use EntryProcessor to convert raw entry to markdown/JSON
    const processor = new EntryProcessor();
    const result = processor.processEntry(entry);
    
    if (!result) {
      console.log(`Skipping unsupported entry type: ${entry.kind}`);
      return;
    }

    // Save directly - no virtual documents, no user clicks!
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const baseName = `${timestamp}_${entry.id}`;
    const basePath = getStorageConfig().basePath.replace('~', require('os').homedir());

    // Save markdown
    if (getStorageConfig().format === 'markdown' || getStorageConfig().format === 'both') {
      const mdPath = path.join(basePath, `${baseName}.md`);
      await fs.promises.writeFile(mdPath, result.markdown, 'utf-8');
      console.log(`Saved markdown: ${mdPath}`);
    }

    // Save JSON
    if (getStorageConfig().format === 'json' || getStorageConfig().format === 'both') {
      const jsonPath = path.join(basePath, `${baseName}.json`);
      await fs.promises.writeFile(jsonPath, JSON.stringify(result.json, null, 2), 'utf-8');
      console.log(`Saved JSON: ${jsonPath}`);
    }

    captureCount++;
    updateStatusBar();
    
    console.log(`✅ AUTO-CAPTURED (NO USER ACTION): ${entry.kind} - ${entry.id}`);
  } catch (err) {
    console.error('Error processing entry directly:', err);
    vscode.window.showErrorMessage(`Failed to auto-capture log: ${err}`);
  }
}

function updateStatusBar(): void {
  statusBarItem.text = `$(copilot) ${captureCount} logs`;
}

async function openExportFolder(): Promise<void> {
  const config = getStorageConfig();
  const folderUri = vscode.Uri.file(config.basePath.replace('~', require('os').homedir()));
  await vscode.commands.executeCommand('revealFileInOS', folderUri);
}

async function clearLogs(): Promise<void> {
  const answer = await vscode.window.showWarningMessage(
    'Are you sure you want to clear all exported logs?',
    'Yes', 'No'
  );
  
  if (answer === 'Yes') {
    const count = await storage.clearLogs();
    deduplicator.clear();
    captureCount = 0;
    updateStatusBar();
    vscode.window.showInformationMessage(`Cleared ${count} log files`);
  }
}

async function showStatus(): Promise<void> {
  const stats = deduplicator.getStats();
  const files = await storage.getStoredFiles();
  
  const message = `
Copilot Chat Auto Exporter Status:
- Captured: ${captureCount} logs
- Stored files: ${files.length}
- Unique URIs: ${stats.uris}
- Unique contents: ${stats.hashes}
  `.trim();
  
  vscode.window.showInformationMessage(message);
}

export function deactivate() {
  if (documentMonitor) documentMonitor.stop();
  if (outputMonitor) outputMonitor.stop();
  if (serviceMonitor) serviceMonitor.stop();
}
```

---

### Phase 5: Testing & Validation (Day 3-4)

#### 5.1 Unit Tests

**Test Cases:**
1. URI parsing and validation
2. Content deduplication
3. File storage operations
4. Metadata extraction
5. Configuration handling

**Test Framework:** Mocha + Chai (VS Code standard)

**File:** `src/test/suite/storage.test.ts`
```typescript
import * as assert from 'assert';
import { Deduplicator } from '../../storage/deduplicator';

suite('Deduplicator Test Suite', () => {
  test('Should detect duplicate URIs', () => {
    const dedup = new Deduplicator();
    const uri = 'ccreq:test123.copilotmd';
    const content = 'test content';
    
    assert.strictEqual(dedup.isDuplicate(uri, content), false);
    assert.strictEqual(dedup.isDuplicate(uri, content), true);
  });
  
  test('Should detect duplicate content', () => {
    const dedup = new Deduplicator();
    const content = 'same content';
    
    assert.strictEqual(dedup.isDuplicate('uri1', content), false);
    assert.strictEqual(dedup.isDuplicate('uri2', content), true);
  });
});
```

---

#### 5.2 Integration Tests

**Test Scenarios:**

1. **Scenario: New Chat Message**
   - Start extension
   - Send chat message in Copilot
   - Verify document is detected
   - Verify file is saved
   - Verify no duplicates

2. **Scenario: Multiple Messages**
   - Send 5 consecutive messages
   - Verify all are captured
   - Verify correct file count
   - Verify unique IDs

3. **Scenario: Streaming Response**
   - Send message with long response
   - Verify document updates are captured
   - Verify final content is complete

4. **Scenario: Tool Usage**
   - Send message that uses tools
   - Verify tool calls are captured
   - Verify tool responses included

5. **Scenario: Extension Restart**
   - Capture logs
   - Reload extension
   - Send new messages
   - Verify no data loss

---

#### 5.3 Manual Testing Checklist

- [ ] Extension activates without errors
- [ ] Status bar item appears
- [ ] First message is captured
- [ ] Subsequent messages are captured
- [ ] No duplicate files created
- [ ] Markdown format is correct
- [ ] JSON format is valid
- [ ] Metadata is extracted correctly
- [ ] Config changes are applied
- [ ] Open folder command works
- [ ] Clear logs command works
- [ ] Show status command works
- [ ] Performance is acceptable
- [ ] No memory leaks
- [ ] Works with different models

---

### Phase 6: Optimization & Polish (Day 4)

#### 6.1 Performance Optimization

**Bottlenecks to address:**
1. File I/O operations (use async/await properly)
2. Content hashing (cache results)
3. Duplicate detection (use efficient data structures)
4. Output channel monitoring (debounce file reads)

**Implementation:**
```typescript
// Debounced file reading
class DebouncedFileReader {
  private timeout: NodeJS.Timeout | undefined;
  private pendingRead: (() => void) | undefined;

  public scheduleRead(callback: () => void, delay: number = 500): void {
    if (this.timeout) {
      clearTimeout(this.timeout);
    }
    
    this.pendingRead = callback;
    this.timeout = setTimeout(() => {
      if (this.pendingRead) {
        this.pendingRead();
        this.pendingRead = undefined;
      }
    }, delay);
  }
}
```

---

#### 6.2 Error Handling

**Add robust error handling:**
```typescript
class ErrorHandler {
  private errorCount = 0;
  private maxErrors = 10;

  public async handleError(error: Error, context: string): Promise<void> {
    this.errorCount++;
    
    console.error(`[${context}] Error:`, error);
    
    if (this.errorCount > this.maxErrors) {
      vscode.window.showErrorMessage(
        `Copilot Auto Exporter: Too many errors (${this.errorCount}). Please check the logs.`
      );
      // Consider disabling extension
      return;
    }
    
    if (this.errorCount % 5 === 0) {
      vscode.window.showWarningMessage(
        `Copilot Auto Exporter: ${this.errorCount} errors occurred`
      );
    }
  }

  public reset(): void {
    this.errorCount = 0;
  }
}
```

---

#### 6.3 Logging System

**Add structured logging:**
```typescript
// src/utils/logger.ts
import * as vscode from 'vscode';

export class Logger {
  private outputChannel: vscode.OutputChannel;

  constructor() {
    this.outputChannel = vscode.window.createOutputChannel(
      'Copilot Chat Auto Exporter'
    );
  }

  public info(message: string): void {
    this.log('INFO', message);
  }

  public warn(message: string): void {
    this.log('WARN', message);
  }

  public error(message: string, err?: Error): void {
    this.log('ERROR', message);
    if (err) {
      this.log('ERROR', err.stack || err.message);
    }
  }

  private log(level: string, message: string): void {
    const timestamp = new Date().toISOString();
    this.outputChannel.appendLine(`[${timestamp}] [${level}] ${message}`);
  }

  public show(): void {
    this.outputChannel.show();
  }
}
```

---

### Phase 7: Documentation (Day 4)

#### 7.1 README.md

**Content:**
- Extension overview
- Features list
- Installation instructions
- Configuration options
- Usage examples
- Troubleshooting guide
- Privacy notice

#### 7.2 CHANGELOG.md

**Version tracking:**
- v0.1.0 - Initial PoC release
- Feature additions
- Bug fixes
- Breaking changes

#### 7.3 Code Documentation

**Add JSDoc comments:**
```typescript
/**
 * Monitors VS Code text documents for Copilot Chat virtual documents.
 * Automatically captures content when ccreq: scheme documents are opened.
 */
export class DocumentMonitor {
  /**
   * Starts monitoring for document events
   * @remarks Subscribes to onDidOpenTextDocument and onDidChangeTextDocument
   */
  public start(): void {
    // ...
  }
}
```

---

## Testing Plan Summary

### Automated Tests
- Unit tests: 20+ test cases
- Integration tests: 5 scenarios
- Coverage target: >80%

### Manual Tests
- Functional testing: 15 checkpoints
- Performance testing: Memory and CPU usage
- Compatibility testing: Different VS Code versions

### Test Environment
- VS Code versions: 1.85, 1.90, latest
- OS: macOS, Windows, Linux
- Copilot Chat: Latest stable version

---

## Deployment Plan

### Phase 1: Internal Testing
- Deploy to local development environment
- Test with real Copilot usage for 1 week
- Collect metrics and logs

### Phase 2: Beta Release
- Package as VSIX
- Share with beta testers
- Gather feedback

### Phase 3: Public Release (Optional)
- Publish to VS Code Marketplace
- Set up issue tracker
- Create support channels

---

## Success Metrics

### Functional Metrics
- **Capture Rate:** >95% of all Copilot requests captured
- **Duplicate Rate:** <1% duplicate files created
- **Error Rate:** <0.1% errors per capture

### Performance Metrics
- **Capture Latency:** <100ms from document open to file write
- **Memory Usage:** <50MB additional memory
- **CPU Usage:** <1% average CPU impact

### User Experience Metrics
- **Activation Time:** <1 second
- **UI Responsiveness:** No noticeable lag
- **Storage Efficiency:** Configurable, defaults sensible

---

## Risk Assessment & Mitigation

### Risk 1: Copilot API Changes
**Impact:** High
**Probability:** Medium
**Mitigation:**
- Use multiple detection methods
- Abstract Copilot-specific code
- Monitor Copilot extension updates

### Risk 2: Performance Impact
**Impact:** Medium
**Probability:** Low
**Mitigation:**
- Implement debouncing
- Use async operations
- Add performance monitoring

### Risk 3: Data Privacy
**Impact:** High
**Probability:** Low
**Mitigation:**
- Clear privacy notice
- User-controlled storage location
- Optional encryption (future)
- No telemetry by default

### Risk 4: Storage Limitations
**Impact:** Low
**Probability:** Medium
**Mitigation:**
- Implement log rotation
- Add storage limits
- Provide cleanup commands

---

## Future Enhancements (Post-PoC)

### Phase 2 Features
1. **Advanced Filtering**
   - Filter by model, duration, token count
   - Filter by success/failure
   - Custom filter expressions

2. **Data Analysis**
   - Statistics dashboard
   - Token usage tracking
   - Response time analytics

3. **Export Formats**
   - CSV export for analysis
   - HTML reports
   - Database integration

4. **Search & Query**
   - Full-text search across logs
   - Query by metadata
   - Timeline view

5. **Security & Privacy**
   - Encrypted storage option
   - PII redaction
   - Retention policies

---

## Timeline Summary

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Project Setup | Day 1 | Project structure, build system |
| Core Detection | Day 1-2 | All monitors implemented |
| Data Storage | Day 2 | Storage system complete |
| Main Extension | Day 3 | Fully functional PoC |
| Testing | Day 3-4 | Test suite, validation |
| Optimization | Day 4 | Performance tuning |
| Documentation | Day 4 | Complete docs |

**Total:** 4 days for complete PoC

---

## Conclusion

This PoC will demonstrate:
- ✅ Fully automatic capture without user intervention
- ✅ Real-time detection using multiple strategies
- ✅ Reliable persistence to filesystem
- ✅ Configurable and extensible architecture
- ✅ Production-ready error handling and logging

**Next Steps:**
1. Review and approve plan
2. Set up development environment
3. Begin Phase 1 implementation
4. Schedule daily check-ins
5. Plan for beta testing

**Expected Outcome:**
A working VS Code extension that automatically captures all Copilot Chat interactions and saves them to disk without requiring any manual user actions.

---

## 📌 CRITICAL: Zero User Interaction Architecture

This PoC achieves **FULLY AUTOMATIC** capture with **ZERO user interaction**:

### ✅ What Users DON'T Need to Do
- ❌ NO clicking on tree view items
- ❌ NO opening virtual documents
- ❌ NO running export commands
- ❌ NO saving files manually
- ❌ NO configuring anything (uses sensible defaults)

### ✅ How It Works Automatically
1. **Extension activates** → Registers service listener
2. **User chats with Copilot** → `IRequestLogger` captures request automatically
3. **Event fires** → `onDidChangeRequests` notifies our extension
4. **Entry processed** → `EntryProcessor` converts to markdown/JSON
5. **File saved** → Automatically written to `~/copilot-chat-exports/`
6. **Status updated** → Status bar shows capture count

### 🎯 Key Design Principles
- **Event-driven**: Uses `onDidChangeRequests` event, not polling
- **Service-based**: Direct `IRequestLogger` access, no extension API dependency
- **Processing-first**: Convert entries directly, skip virtual documents entirely
- **Silent operation**: Everything happens in background
- **Zero-config**: Works out-of-box with smart defaults

### 🔧 Why This Approach Works
- **`IRequestLogger`** is the source of truth, not the UI
- **Events fire automatically** when Copilot logs requests
- **Raw entries contain all data** - no need to render virtual documents
- **File I/O is independent** - we control when/how to save
- **No UI dependency** - extension works even if tree view is closed

### 📊 Expected User Experience
1. Install extension
2. Continue using Copilot normally
3. Check `~/copilot-chat-exports/` anytime to see captured logs
4. **That's it!** No other steps required.

```
