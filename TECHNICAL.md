# Claude Agent Desktop - Technical Documentation

This document provides a comprehensive technical overview of the Claude Agent Desktop codebase, explaining its architecture, build system, and key components.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Technology Stack](#technology-stack)
- [Directory Structure](#directory-structure)
- [Build System](#build-system)
- [Skills System](#skills-system)
- [IPC Communication](#ipc-communication)
- [Main Process](#main-process)
- [Renderer Process](#renderer-process)
- [Configuration & State Management](#configuration--state-management)
- [Auto-Update System](#auto-update-system)
- [Development Workflow](#development-workflow)

## Architecture Overview

Claude Agent Desktop is an Electron-based desktop application that provides a graphical interface for the Claude Agent SDK. The application follows the standard Electron multi-process architecture:

```
┌──────────────────────────────────────────────────┐
│                  Main Process                     │
│  (Node.js runtime - full system access)          │
│                                                   │
│  • App lifecycle management                      │
│  • IPC handlers (chat, config, conversations)   │
│  • Claude Agent SDK integration                  │
│  • File system operations                        │
│  • Window management                             │
│  • Auto-update logic                             │
└─────────────┬────────────────────────────────────┘
              │ IPC (Inter-Process Communication)
              │
┌─────────────▼────────────────────────────────────┐
│                 Preload Script                    │
│  (Secure bridge - contextBridge)                 │
│                                                   │
│  • Exposes safe APIs to renderer                 │
│  • Type-safe IPC wrappers                        │
└─────────────┬────────────────────────────────────┘
              │
┌─────────────▼────────────────────────────────────┐
│               Renderer Process                    │
│  (React application - sandboxed)                 │
│                                                   │
│  • React 19 UI components                        │
│  • Chat interface                                │
│  • Settings panel                                │
│  • Message rendering                             │
│  • No direct system access                       │
└──────────────────────────────────────────────────┘
```

### Key Design Principles

1. **Security**: Renderer process is sandboxed with `contextIsolation: true` and `nodeIntegration: false`
2. **Separation of Concerns**: Business logic in main process, presentation in renderer
3. **Type Safety**: Shared TypeScript types across processes
4. **Modularity**: Clear separation between handlers, services, and UI components

## Technology Stack

### Core Framework
- **Electron 39**: Cross-platform desktop framework
- **React 19**: UI library with modern hooks
- **TypeScript 5.9**: Type-safe development

### Build Tools
- **Vite 7**: Fast development server and build tool
- **electron-vite 4**: Electron-specific Vite configuration
- **Bun 1.3**: Package manager, test runner, and TypeScript compiler
- **electron-builder 26**: Application packaging and distribution

### UI & Styling
- **Tailwind CSS 4**: Utility-first CSS framework
- **react-markdown 10**: Markdown rendering for chat messages
- **lucide-react**: Icon library

### Runtime Dependencies
- **@anthropic-ai/claude-agent-sdk 0.1.49**: Core agent functionality
- **electron-updater 6.6**: Auto-update mechanism

### Bundled Runtimes
The application bundles platform-specific binaries for agent execution:
- **bun**: JavaScript/TypeScript runtime
- **uv**: Python package manager and runner
- **git-portable** (Windows): Portable Git distribution
- **msys2** (Windows): Unix-like environment for Windows

## Directory Structure

```
claude-agent-desktop/
├── src/
│   ├── main/              # Electron main process
│   │   ├── handlers/      # IPC request handlers
│   │   │   ├── chat-handlers.ts       # Chat message handling
│   │   │   ├── config-handlers.ts     # Configuration management
│   │   │   ├── conversation-handlers.ts # Conversation CRUD
│   │   │   ├── shell-handlers.ts      # External URLs
│   │   │   └── update-handlers.ts     # Auto-update operations
│   │   ├── lib/           # Core business logic
│   │   │   ├── claude-session.ts      # Claude SDK integration
│   │   │   ├── config.ts              # App configuration
│   │   │   ├── conversation-db.ts     # Conversation persistence
│   │   │   ├── message-queue.ts       # Message queueing
│   │   │   ├── updater.ts             # Update checking
│   │   │   └── window-state.ts        # Window bounds persistence
│   │   ├── index.ts       # Main process entry point
│   │   └── menu.ts        # Application menu
│   │
│   ├── preload/           # Preload script (security bridge)
│   │   └── index.ts       # Context bridge exposing safe APIs
│   │
│   ├── renderer/          # React application
│   │   ├── components/    # Reusable UI components
│   │   │   ├── AttachmentPreviewList.tsx
│   │   │   ├── BlockGroup.tsx         # Groups thinking/tool blocks
│   │   │   ├── ChatHistoryDrawer.tsx  # Conversation sidebar
│   │   │   ├── ChatInput.tsx          # Message input with attachments
│   │   │   ├── Message.tsx            # Single message renderer
│   │   │   ├── MessageList.tsx        # Scrollable message list
│   │   │   ├── TitleBar.tsx           # Custom window controls
│   │   │   ├── ToolUse.tsx            # Tool execution display
│   │   │   └── Update*.tsx            # Update UI components
│   │   ├── pages/         # Top-level views
│   │   │   ├── Chat.tsx   # Main chat interface
│   │   │   └── Settings.tsx # Configuration panel
│   │   ├── hooks/         # Custom React hooks
│   │   ├── utils/         # Utility functions
│   │   ├── types/         # TypeScript types
│   │   ├── App.tsx        # Root component
│   │   ├── main.tsx       # React entry point
│   │   └── index.html     # HTML shell
│   │
│   └── shared/            # Code shared between processes
│       ├── constants.ts   # Shared constants
│       └── types/         # Shared TypeScript types
│
├── .claude/               # Claude Agent SDK integration
│   └── skills/            # Pre-configured agent skills
│       ├── docx/          # Word document handling
│       ├── frontend-design/  # UI design assistance
│       ├── pdf/           # PDF manipulation
│       ├── workspace-tools/  # File system utilities
│       └── xlsx/          # Excel spreadsheet handling
│
├── scripts/               # Build and setup scripts
│   ├── preDev.js          # Pre-development setup
│   ├── buildSkills.js     # Compile TypeScript skills to binaries
│   ├── beforeBuild.js     # Production build preparation
│   ├── afterPack.js       # Post-packaging cleanup
│   └── downloadRuntimeBinaries.js  # Download bun, uv, etc.
│
├── resources/             # Bundled binaries (downloaded during build)
│   ├── bun / bun.exe
│   ├── uv / uv.exe
│   ├── jq.exe (Windows)
│   ├── git-portable/ (Windows)
│   ├── msys2/ (Windows)
│   └── entitlements.*.plist (macOS)
│
├── static/                # Static assets
│   └── icon.png           # Application icon
│
├── out/                   # Build output (gitignored)
│   ├── main/              # Compiled main process
│   ├── preload/           # Compiled preload
│   ├── renderer/          # Compiled renderer
│   └── .claude/skills/    # Compiled skill binaries
│
├── dist/                  # Packaged applications (gitignored)
│
├── electron.vite.config.ts  # Vite configuration
├── tsconfig.json            # TypeScript configuration
├── package.json             # Dependencies and scripts
└── bun.lock                 # Dependency lockfile
```

## Build System

The build system uses **electron-vite** which configures Vite for the three Electron processes (main, preload, renderer) with different settings for each.

### Build Configuration (`electron.vite.config.ts`)

```typescript
{
  main: {
    // Node.js environment for main process
    plugins: [externalizeDepsPlugin()],  // Keep node_modules external
    build: { outDir: 'out/main' }
  },
  preload: {
    // CommonJS output for preload script
    plugins: [externalizeDepsPlugin()],
    build: {
      outDir: 'out/preload',
      format: 'cjs',  // Must be CommonJS
      entryFileNames: '[name].cjs'
    }
  },
  renderer: {
    // Browser environment for React app
    plugins: [react(), tailwindcss()],
    build: { outDir: 'out/renderer' }
  }
}
```

### Build Pipeline

#### Development (`bun run dev`)

1. **preDev.js**: Pre-development setup
   - Downloads runtime binaries (bun, uv) if not present
   - Builds skills to `out/.claude/skills/`
2. **electron-vite dev**: Starts development server
   - Main & preload: Watches and rebuilds on changes
   - Renderer: Vite dev server with HMR (Hot Module Replacement)
3. Electron launches with dev server URL

#### Production Build (`bun run build`)

1. **electron-vite build**: Compiles all three processes
   - Main process → `out/main/index.js`
   - Preload script → `out/preload/index.cjs`
   - Renderer → `out/renderer/` (static assets)

#### Distribution (`bun run build:mac` / `build:win`)

1. **beforeBuild.js**: Pre-packaging setup
   - Downloads runtime binaries for target platform
   - Copies runtime dependencies to `out/node_modules/`
     - `@anthropic-ai/claude-agent-sdk` (required for agent execution)
     - `@img/sharp-*` (optional, platform-specific image processing)
   - Builds skills to compiled binaries
2. **electron-builder**: Packages application
   - Creates `.app` bundle (macOS) or installer (Windows)
   - Unpacks specified files from asar archive (SDK, skills, native bindings)
   - Bundles runtime binaries into app resources
3. **afterPack.js**: Post-packaging cleanup
   - Removes unused JetBrains plugin from SDK
   - Verifies `.claude/skills/` directory exists
4. Output: `dist/Claude Agent Desktop.app` or `dist/Claude Agent Desktop Setup.exe`

### Package.json Scripts

```json
{
  "dev": "node scripts/preDev.js && electron-vite dev",
  "build": "electron-vite build",
  "build:mac": "bun run build && electron-builder --mac",
  "build:win": "bun run build && electron-builder --win",
  "typecheck": "tsc --noEmit",
  "lint": "eslint . --cache --max-warnings=0",
  "test": "bun test",
  "format": "prettier --experimental-cli --write ."
}
```

## Skills System

Skills are pre-configured capabilities powered by the Claude Agent SDK. They provide specialized tools for document processing, workspace management, and more.

### Skill Structure

Each skill lives in `.claude/skills/<skill-name>/`:

```
.claude/skills/workspace-tools/
├── SKILL.md              # Skill metadata (name, description, tools)
└── scripts/              # TypeScript tools
    └── list-directory/
        └── list-directory.ts  # Tool implementation
```

### Skill Metadata (`SKILL.md`)

```markdown
---
name: workspace-tools
description: Utilities for inspecting the local project workspace
license: MIT
---

# Tools

## list-directory
- Purpose: Print a depth-limited directory tree as JSON
- Usage: `./scripts/list-directory/list-directory --path ./src --depth 3`
- Flags: --path, --depth, --json
```

### Skill Compilation

TypeScript tools are compiled to standalone binaries during the build process:

1. **Input**: `.claude/skills/<skill>/scripts/**/*.ts`
2. **Compilation**: `bun build --compile` creates platform-specific executables
3. **Output**: `out/.claude/skills/<skill>/scripts/**/<name>` (or `.exe` on Windows)

The compilation happens in `scripts/buildSkills.js`:

```javascript
// For each .ts file in skills
bun build --compile --outfile <target> <source.ts>
```

**Key Points**:
- Skills use the root `package.json` dependencies (no separate `.claude/package.json`)
- TypeScript is checked via root `tsconfig.json` (includes `.claude/skills/**/*.ts`)
- Compiled binaries are standalone and require no dependencies at runtime
- Skills are synced to the workspace `.claude/` directory on app launch

### Workspace Sync

On application startup, `src/main/lib/config.ts` syncs the bundled `.claude/` directory into the user's workspace:

```typescript
export async function ensureWorkspaceDir(): Promise<void> {
  const workspaceDir = getWorkspaceDir();  // ~/Desktop/claude-agent by default
  await mkdir(workspaceDir, { recursive: true });
  
  // Sync bundled .claude to workspace .claude
  const bundledClaudeDir = join(app.getAppPath(), 'out', '.claude');
  const workspaceClaudeDir = join(workspaceDir, '.claude');
  
  if (existsSync(bundledClaudeDir)) {
    await rm(workspaceClaudeDir, { recursive: true, force: true });
    await cp(bundledClaudeDir, workspaceClaudeDir, { recursive: true });
  }
}
```

This ensures the workspace always has the latest bundled skills.

## IPC Communication

Electron's Inter-Process Communication (IPC) enables secure communication between the main and renderer processes.

### Architecture

```
Renderer Process          Preload Script           Main Process
   (React)              (Context Bridge)         (Node.js)
      │                        │                       │
      │  window.electron.     │                       │
      │  chat.sendMessage()   │                       │
      ├───────────────────────>│                       │
      │                        │  ipcRenderer.invoke  │
      │                        │  ('chat:send-message')│
      │                        ├──────────────────────>│
      │                        │                       │
      │                        │    ipcMain.handle()  │
      │                        │    executes handler  │
      │                        │<──────────────────────┤
      │  Promise resolves      │                       │
      │<───────────────────────┤                       │
      │                        │                       │
```

### Preload Script (`src/preload/index.ts`)

The preload script uses `contextBridge` to expose a safe API to the renderer:

```typescript
contextBridge.exposeInMainWorld('electron', {
  chat: {
    sendMessage: (payload: SendMessagePayload) => 
      ipcRenderer.invoke('chat:send-message', payload),
    onMessageChunk: (callback) => {
      ipcRenderer.on('chat:message-chunk', (_event, chunk) => callback(chunk));
      return () => ipcRenderer.removeListener('chat:message-chunk', ...);
    }
  },
  config: {
    getWorkspaceDir: () => ipcRenderer.invoke('config:get-workspace-dir'),
    setWorkspaceDir: (dir) => ipcRenderer.invoke('config:set-workspace-dir', dir)
  },
  // ... more APIs
});
```

**Key Features**:
- Type-safe: `SendMessagePayload` shared between processes
- Cleanup: Event listeners return unsubscribe functions
- Security: Only explicitly exposed APIs are available to renderer

### Main Process Handlers

Handlers are registered in `src/main/index.ts`:

```typescript
app.whenReady().then(async () => {
  registerConfigHandlers();     // config:*
  registerChatHandlers(() => mainWindow);  // chat:*
  registerConversationHandlers();  // conversation:*
  registerShellHandlers();      // shell:*
  registerUpdateHandlers();     // update:*
  
  createWindow();
});
```

Example handler (`src/main/handlers/chat-handlers.ts`):

```typescript
export function registerChatHandlers(getMainWindow: () => BrowserWindow | null) {
  ipcMain.handle('chat:send-message', async (_event, payload: SendMessagePayload) => {
    const apiKey = getApiKey();
    if (!apiKey) {
      return { success: false, error: 'API key not configured' };
    }
    
    // Process message, start Claude session
    const userMessage = buildUserMessage(payload.text, payload.attachments);
    await messageQueue.push({ message: userMessage });
    
    return { success: true };
  });
}
```

### Streaming Events

For real-time updates (e.g., Claude's response), the main process sends events to the renderer:

```typescript
// Main process
mainWindow.webContents.send('chat:message-chunk', chunk);

// Renderer (via preload)
window.electron.chat.onMessageChunk((chunk) => {
  console.log('Received:', chunk);
});
```

### IPC Channels

| Channel | Type | Purpose |
|---------|------|---------|
| `chat:send-message` | invoke | Send user message to Claude |
| `chat:stop-message` | invoke | Interrupt current response |
| `chat:reset-session` | invoke | Clear session and start fresh |
| `chat:message-chunk` | send | Stream text from Claude |
| `chat:thinking-start/chunk` | send | Stream Claude's thinking |
| `chat:tool-use-start` | send | Tool execution begins |
| `chat:tool-result-*` | send | Tool execution results |
| `config:get/set-*` | invoke | Configuration management |
| `conversation:list/create/update/delete` | invoke | Conversation CRUD |
| `update:check/download/install` | invoke | Auto-update operations |

## Main Process

The main process manages the application lifecycle, system resources, and business logic.

### Entry Point (`src/main/index.ts`)

```typescript
// Fix PATH to include bundled binaries (bun, uv, git, msys2)
process.env.PATH = buildEnhancedPath();

app.whenReady().then(async () => {
  // Set app metadata
  app.name = 'Claude Agent Desktop';
  
  // Register IPC handlers
  registerConfigHandlers();
  registerChatHandlers(() => mainWindow);
  registerConversationHandlers();
  registerShellHandlers();
  registerUpdateHandlers();
  
  // Create window
  createWindow();
  
  // Initialize auto-updater
  initializeUpdater(mainWindow);
  startPeriodicUpdateCheck();
  
  // Set application menu
  const menu = createApplicationMenu(mainWindow);
  Menu.setApplicationMenu(menu);
  
  // Ensure workspace exists and sync skills
  ensureWorkspaceDir().catch(console.error);
});
```

**Key Responsibilities**:
- Configure environment (PATH, security flags)
- Register IPC handlers before window creation
- Manage window lifecycle and bounds persistence
- Initialize updater and menu
- Ensure workspace directory exists

### Claude Session Management (`src/main/lib/claude-session.ts`)

This module integrates with the Claude Agent SDK to power the chat interface.

#### Session Lifecycle

```typescript
let querySession: Query | null = null;
let isProcessing = false;

export function startStreamingSession(getMainWindow: () => BrowserWindow | null) {
  if (querySession || isProcessing) return;
  
  isProcessing = true;
  
  const workspaceDir = getWorkspaceDir();
  const apiKey = getApiKey();
  const modelId = getModelIdForPreference(currentModelPreference);
  
  querySession = query({
    workspaceDir,
    apiKey,
    modelId,
    cliPath: resolveClaudeCodeCli(),  // SDK's cli.js
    onSystemPromptAppend: () => SYSTEM_PROMPT_APPEND,  // Custom instructions
    env: buildClaudeSessionEnv()  // Bundled binaries in PATH
  });
  
  // Process messages from queue
  (async () => {
    for await (const message of messageGenerator()) {
      await querySession.ask(message);
    }
  })();
}
```

#### Model Management

```typescript
const FAST_MODEL_ID = 'claude-haiku-4-5-20251001';
const SMART_MODEL_ID = 'claude-sonnet-4-5-20250929';

const MODEL_BY_PREFERENCE: Record<ChatModelPreference, string> = {
  fast: FAST_MODEL_ID,
  smart: SMART_MODEL_ID
};

export async function setChatModelPreference(preference: ChatModelPreference) {
  currentModelPreference = preference;
  
  if (querySession) {
    await querySession.setModel(getModelIdForPreference(preference));
  }
  
  setChatModelPreferenceSetting(preference);
}
```

#### Event Streaming

The session listens to SDK events and forwards them to the renderer:

```typescript
querySession.on('thinking-start', (data) => {
  mainWindow?.webContents.send('chat:thinking-start', { index: data.index });
});

querySession.on('thinking-delta', (data) => {
  mainWindow?.webContents.send('chat:thinking-chunk', {
    index: data.index,
    delta: data.delta
  });
});

querySession.on('tool-use-start', (data) => {
  mainWindow?.webContents.send('chat:tool-use-start', {
    id: data.id,
    name: data.name,
    input: data.input,
    streamIndex: data.index
  });
});

// ... more event handlers
```

### Message Queue (`src/main/lib/message-queue.ts`)

Ensures messages are processed sequentially to avoid race conditions:

```typescript
type QueuedMessage = {
  message: SDKUserMessage;
  resolve: () => void;
};

const queue: QueuedMessage[] = [];

export const messageQueue = {
  push: (item: QueuedMessage) => {
    queue.push(item);
  }
};

export async function* messageGenerator() {
  while (true) {
    while (queue.length === 0) {
      await new Promise(resolve => setTimeout(resolve, 100));  // Poll
    }
    
    const { message, resolve } = queue.shift()!;
    yield message;
    resolve();
  }
}
```

### Configuration Management (`src/main/lib/config.ts`)

Manages application settings persisted to `userData/config.json`:

```typescript
export interface AppConfig {
  workspaceDir?: string;
  debugMode?: boolean;
  chatModelPreference?: ChatModelPreference;
  apiKey?: string;
}

export function loadConfig(): AppConfig {
  const configPath = join(app.getPath('userData'), 'config.json');
  if (existsSync(configPath)) {
    return JSON.parse(readFileSync(configPath, 'utf-8'));
  }
  return {};
}

export function getApiKey(): string | null {
  // Environment variable takes precedence
  const envKey = process.env.ANTHROPIC_API_KEY?.trim();
  if (envKey) return envKey;
  
  // Fall back to stored key
  const storedKey = loadConfig().apiKey?.trim();
  return storedKey || null;
}

export function getWorkspaceDir(): string {
  const config = loadConfig();
  return config.workspaceDir || join(app.getPath('desktop'), 'claude-agent');
}
```

### Conversation Persistence (`src/main/lib/conversation-db.ts`)

Stores conversation history in JSON files under `userData/conversations/`:

```typescript
export interface ConversationMetadata {
  id: string;
  title: string;
  createdAt: string;
  updatedAt: string;
  sessionId?: string | null;
}

export interface Conversation extends ConversationMetadata {
  messages: Array<UserMessage | AssistantMessage>;
}

export async function createConversation(
  messages: Array<UserMessage | AssistantMessage>,
  sessionId?: string | null
): Promise<Conversation> {
  const id = randomUUID();
  const timestamp = new Date().toISOString();
  
  const conversation: Conversation = {
    id,
    title: generateConversationTitle(messages),
    messages,
    sessionId: sessionId ?? null,
    createdAt: timestamp,
    updatedAt: timestamp
  };
  
  const conversationsDir = join(app.getPath('userData'), 'conversations');
  await mkdir(conversationsDir, { recursive: true });
  
  const filePath = join(conversationsDir, `${id}.json`);
  await writeFile(filePath, JSON.stringify(conversation, null, 2));
  
  return conversation;
}
```

## Renderer Process

The renderer process is a sandboxed React application that provides the user interface.

### Application Structure (`src/renderer/App.tsx`)

```typescript
export default function App() {
  const [currentView, setCurrentView] = useState<'home' | 'settings'>('home');
  
  useEffect(() => {
    // Listen for navigation from main process (e.g., menu commands)
    const unsubscribe = window.electron.onNavigate((view) => {
      setCurrentView(view as 'home' | 'settings');
    });
    return unsubscribe;
  }, []);
  
  return (
    <>
      <UpdateCheckFeedback />
      <UpdateNotification />
      <UpdateReadyBanner />
      
      <div className={currentView === 'settings' ? 'block' : 'hidden'}>
        <Settings onBack={() => setCurrentView('home')} />
      </div>
      
      <div className={currentView === 'home' ? 'block' : 'hidden'}>
        <Chat />
      </div>
    </>
  );
}
```

### Chat Interface (`src/renderer/pages/Chat.tsx`)

The main chat view manages conversation state and handles user interactions:

```typescript
export default function Chat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [isStreaming, setIsStreaming] = useState(false);
  const [currentSessionId, setCurrentSessionId] = useState<string | null>(null);
  
  useEffect(() => {
    // Listen for message chunks from main process
    const unsubscribeChunk = window.electron.chat.onMessageChunk((chunk) => {
      setMessages(prev => {
        const updated = [...prev];
        const lastMsg = updated[updated.length - 1];
        if (lastMsg?.role === 'assistant') {
          lastMsg.content += chunk;
        }
        return updated;
      });
    });
    
    // Listen for tool executions
    const unsubscribeToolUse = window.electron.chat.onToolUseStart((tool) => {
      setMessages(prev => [...prev, { type: 'tool-use', ...tool }]);
    });
    
    return () => {
      unsubscribeChunk();
      unsubscribeToolUse();
      // ... more cleanup
    };
  }, []);
  
  const handleSendMessage = async (text: string, attachments: File[]) => {
    const result = await window.electron.chat.sendMessage({ text, attachments });
    if (result.success) {
      setMessages(prev => [...prev, { role: 'user', content: text }]);
    }
  };
  
  return (
    <div className="flex flex-col h-screen">
      <TitleBar onSettingsClick={() => { /* ... */ }} />
      <MessageList messages={messages} />
      <ChatInput onSend={handleSendMessage} disabled={isStreaming} />
    </div>
  );
}
```

### Key Components

#### MessageList (`src/renderer/components/MessageList.tsx`)

Renders scrollable message history with auto-scroll:

```typescript
export default function MessageList({ messages }: { messages: Message[] }) {
  const listRef = useRef<HTMLDivElement>(null);
  const shouldAutoScroll = useRef(true);
  
  useEffect(() => {
    if (shouldAutoScroll.current && listRef.current) {
      listRef.current.scrollTop = listRef.current.scrollHeight;
    }
  }, [messages]);
  
  return (
    <div ref={listRef} className="flex-1 overflow-y-auto">
      {messages.map((msg, i) => (
        <Message key={i} message={msg} />
      ))}
    </div>
  );
}
```

#### Message (`src/renderer/components/Message.tsx`)

Renders individual messages with markdown support:

```typescript
export default function Message({ message }: { message: Message }) {
  if (message.role === 'user') {
    return (
      <div className="user-message">
        <Markdown>{message.content}</Markdown>
        {message.attachments && <AttachmentPreviewList items={message.attachments} />}
      </div>
    );
  }
  
  if (message.role === 'assistant') {
    return (
      <div className="assistant-message">
        <Markdown remarkPlugins={[remarkGfm]}>{message.content}</Markdown>
      </div>
    );
  }
  
  // ... tool-use, thinking blocks, etc.
}
```

#### ChatInput (`src/renderer/components/ChatInput.tsx`)

Text input with file attachment support:

```typescript
export default function ChatInput({ onSend, disabled }: Props) {
  const [text, setText] = useState('');
  const [attachments, setAttachments] = useState<File[]>([]);
  
  const handleSubmit = () => {
    if (text.trim() || attachments.length > 0) {
      onSend(text, attachments);
      setText('');
      setAttachments([]);
    }
  };
  
  return (
    <div className="chat-input">
      <input
        type="file"
        multiple
        onChange={(e) => setAttachments([...e.target.files])}
      />
      <textarea
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => {
          if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            handleSubmit();
          }
        }}
        disabled={disabled}
      />
      <button onClick={handleSubmit} disabled={disabled}>Send</button>
    </div>
  );
}
```

#### BlockGroup (`src/renderer/components/BlockGroup.tsx`)

Groups related blocks (thinking, tool calls) with collapsible UI:

```typescript
export default function BlockGroup({ blocks, label }: Props) {
  const [expanded, setExpanded] = useState(true);
  
  return (
    <div className="block-group">
      <button onClick={() => setExpanded(!expanded)}>
        {expanded ? <ChevronDown /> : <ChevronRight />}
        {label} ({blocks.length})
      </button>
      
      {expanded && (
        <div className="block-content">
          {blocks.map((block, i) => (
            <BlockItem key={i} block={block} />
          ))}
        </div>
      )}
    </div>
  );
}
```

### Settings Panel (`src/renderer/pages/Settings.tsx`)

Provides configuration UI:

```typescript
export default function Settings({ onBack }: { onBack: () => void }) {
  const [workspaceDir, setWorkspaceDir] = useState('');
  const [apiKeyStatus, setApiKeyStatus] = useState({ configured: false });
  const [debugMode, setDebugMode] = useState(false);
  
  useEffect(() => {
    // Load current settings
    (async () => {
      const [dir, keyStatus, debug] = await Promise.all([
        window.electron.config.getWorkspaceDir(),
        window.electron.config.getApiKeyStatus(),
        window.electron.config.getDebugMode()
      ]);
      setWorkspaceDir(dir);
      setApiKeyStatus(keyStatus);
      setDebugMode(debug);
    })();
  }, []);
  
  const handleSaveWorkspaceDir = async (newDir: string) => {
    await window.electron.config.setWorkspaceDir(newDir);
    setWorkspaceDir(newDir);
  };
  
  return (
    <div className="settings-panel">
      <button onClick={onBack}>← Back</button>
      
      <section>
        <h2>Workspace Directory</h2>
        <input value={workspaceDir} onChange={(e) => setWorkspaceDir(e.target.value)} />
        <button onClick={() => handleSaveWorkspaceDir(workspaceDir)}>Save</button>
      </section>
      
      <section>
        <h2>API Key</h2>
        {apiKeyStatus.configured ?
          <p>✓ Configured (source: {apiKeyStatus.source})</p> :
          <p>⚠ Not configured</p>
        }
      </section>
      
      <section>
        <h2>Debug Mode</h2>
        <input
          type="checkbox"
          checked={debugMode}
          onChange={async (e) => {
            await window.electron.config.setDebugMode(e.target.checked);
            setDebugMode(e.target.checked);
          }}
        />
      </section>
    </div>
  );
}
```

## Configuration & State Management

### Configuration Hierarchy

1. **Environment Variables** (highest priority)
   - `ANTHROPIC_API_KEY`: API key
   - `UPDATE_FEED_URL`: Auto-update feed URL

2. **Local Storage** (`userData/config.json`)
   - `workspaceDir`: Workspace path
   - `debugMode`: Debug logging
   - `chatModelPreference`: 'fast' or 'smart'
   - `apiKey`: Stored API key (if not in environment)

3. **Defaults** (lowest priority)
   - Workspace: `~/Desktop/claude-agent`
   - Debug mode: `false`
   - Model: `'fast'` (Haiku)

### State Persistence

#### Window Bounds (`src/main/lib/window-state.ts`)

Persists window size and position across sessions:

```typescript
export function loadWindowBounds(): Electron.Rectangle | null {
  try {
    const statePath = join(app.getPath('userData'), 'window-state.json');
    if (existsSync(statePath)) {
      return JSON.parse(readFileSync(statePath, 'utf-8'));
    }
  } catch (error) {
    console.error('Failed to load window state:', error);
  }
  return null;
}

export function saveWindowBounds(bounds: Electron.Rectangle): void {
  try {
    const statePath = join(app.getPath('userData'), 'window-state.json');
    writeFileSync(statePath, JSON.stringify(bounds, null, 2));
  } catch (error) {
    console.error('Failed to save window state:', error);
  }
}
```

#### Conversation History

Stored as individual JSON files in `userData/conversations/<id>.json`:

```json
{
  "id": "uuid",
  "title": "Generated title",
  "messages": [
    { "role": "user", "content": "Hello" },
    { "role": "assistant", "content": "Hi!" }
  ],
  "sessionId": "session-uuid",
  "createdAt": "2025-01-01T00:00:00Z",
  "updatedAt": "2025-01-01T00:05:00Z"
}
```

## Auto-Update System

The application uses `electron-updater` for automatic updates.

### Update Flow

```
1. App starts → Check for updates (if UPDATE_FEED_URL set)
2. Update available → Notify user via banner
3. User clicks "Download" → Download in background
4. Download complete → Show "Ready to Install" banner
5. User clicks "Restart & Install" → Quit and install update
```

### Implementation (`src/main/lib/updater.ts`)

```typescript
import { autoUpdater } from 'electron-updater';

let updateCheckInterval: NodeJS.Timeout | null = null;

export function initializeUpdater(window: BrowserWindow | null) {
  const updateFeedUrl = process.env.UPDATE_FEED_URL;
  
  if (!updateFeedUrl) {
    console.log('UPDATE_FEED_URL not set, skipping auto-update');
    return;
  }
  
  autoUpdater.setFeedURL({ url: updateFeedUrl });
  
  autoUpdater.on('update-available', (info) => {
    updateStatus.updateAvailable = true;
    updateStatus.updateInfo = {
      version: info.version,
      releaseDate: info.releaseDate,
      releaseNotes: info.releaseNotes
    };
    notifyRenderer(window);
  });
  
  autoUpdater.on('download-progress', (progress) => {
    updateStatus.downloading = true;
    updateStatus.downloadProgress = progress.percent;
    notifyRenderer(window);
  });
  
  autoUpdater.on('update-downloaded', () => {
    updateStatus.downloading = false;
    updateStatus.readyToInstall = true;
    notifyRenderer(window);
  });
}

export function startPeriodicUpdateCheck() {
  // Check every 12 hours
  updateCheckInterval = setInterval(() => {
    checkForUpdates();
  }, 12 * 60 * 60 * 1000);
}

export async function checkForUpdates() {
  try {
    await autoUpdater.checkForUpdates();
  } catch (error) {
    console.error('Update check failed:', error);
  }
}
```

### Update UI Components

- **UpdateNotification**: Toast notification when update available
- **UpdateCheckFeedback**: Progress indicator during manual check
- **UpdateReadyBanner**: Persistent banner when update downloaded

## Development Workflow

### Initial Setup

```bash
# Install Bun (if not already installed)
curl -fsSL https://bun.sh/install | bash

# Install dependencies
bun install

# Run development server (downloads binaries, builds skills)
bun run dev
```

### Development Commands

```bash
# Start dev server with hot reload
bun run dev

# Type checking
bun run typecheck

# Linting
bun run lint

# Run tests
bun run test

# Format code
bun run format
```

### Code Quality

#### TypeScript Configuration

- **Strict mode**: Enabled for all files
- **Path aliases**: `@/*` → `src/renderer/*`
- **Includes**: Main source, scripts, skills, config files

#### ESLint Configuration (`eslint.config.js`)

- **Base**: `@eslint/js.recommended`
- **TypeScript**: `typescript-eslint.recommended`
- **React**: `eslint-plugin-react`, `react-hooks`
- **Prettier**: `eslint-config-prettier` (disables conflicting rules)

#### Testing

Tests use Bun's built-in test runner:

```typescript
// src/renderer/utils/parsePartialJson.test.ts
import { describe, expect, test } from 'bun:test';
import { parsePartialJson } from './parsePartialJson';

describe('parsePartialJson', () => {
  test('parses complete JSON', () => {
    expect(parsePartialJson('{"foo":"bar"}')).toEqual({ foo: 'bar' });
  });
  
  test('parses incomplete JSON', () => {
    expect(parsePartialJson('{"foo":"bar')).toEqual({ foo: 'bar' });
  });
});
```

### Debugging

#### Main Process

1. Add breakpoints in VS Code
2. Launch with `Electron: Main` debug configuration
3. Attach to main process

#### Renderer Process

1. In dev mode: Open DevTools with `Cmd+Option+I` (macOS) or `Ctrl+Shift+I` (Windows)
2. Use React DevTools extension
3. Inspect IPC communication via console

#### Debug Mode

Enable in Settings → Debug Mode:
- Logs detailed IPC events
- Shows SDK communication
- Displays full error stacks

### Project-Specific Conventions

#### Commit Messages

Follow Conventional Commits format:

```
feat: add conversation history sidebar
fix: resolve session persistence issue
docs: update technical documentation
chore: upgrade electron to v39
```

#### Code Style

- **Imports**: Sorted with `@ianvs/prettier-plugin-sort-imports`
- **Naming**: camelCase for functions/variables, PascalCase for components
- **Comments**: Only when necessary for complex logic
- **File Structure**: Group by feature (handlers/, components/, pages/)

### Adding a New Skill

1. Create skill directory: `.claude/skills/my-skill/`
2. Add `SKILL.md`:
   ```markdown
   ---
   name: my-skill
   description: Description of the skill
   ---
   
   # Tools
   ## my-tool
   - Purpose: What it does
   - Usage: `./scripts/my-tool/my-tool --arg value`
   ```
3. Create TypeScript tool: `.claude/skills/my-skill/scripts/my-tool/my-tool.ts`
4. Build skills: `node scripts/buildSkills.js`
5. Test in dev mode: `bun run dev`

### Production Build Checklist

Before building for distribution:

1. **Run quality checks**:
   ```bash
   bun run typecheck
   bun run lint
   bun run test
   bun run format:check
   ```

2. **Test skills compilation**:
   ```bash
   node scripts/buildSkills.js
   # Verify out/.claude/skills/ contains compiled binaries
   ```

3. **Build for target platform**:
   ```bash
   bun run build:mac    # macOS
   bun run build:win    # Windows
   ```

4. **Verify build output**:
   - Check `dist/` for packaged app
   - Test launch and basic functionality
   - Verify skills are accessible in workspace

5. **Code signing** (if applicable):
   - macOS: Set `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`
   - Windows: Set `CSC_LINK`, `CSC_KEY_PASSWORD`

---

## Summary

Claude Agent Desktop is a well-architected Electron application that:

- **Bridges the gap** between powerful AI capabilities and non-technical users
- **Bundles everything** needed for agent execution (bun, uv, git, skills)
- **Uses modern tooling** (Electron 39, React 19, Vite 7, TypeScript)
- **Maintains security** via process isolation and contextBridge
- **Provides extensibility** through the skills system
- **Ensures quality** with TypeScript, ESLint, and automated tests

The codebase follows Electron best practices with clear separation between processes, type-safe IPC, and a modular structure that makes it easy to understand and extend.
