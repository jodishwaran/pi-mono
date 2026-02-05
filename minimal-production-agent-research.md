# Minimal Production-Grade Agent Architecture

## Research Document for Architecture Guild

**Status**: Deep Research Phase Complete  
**Objective**: Design a minimal, composable, production-grade AI agent restricted to Azure AI Cloud Foundry  
**Reference Implementation**: Pi Agent Harness ([@mariozechner/pi-mono](https://github.com/badlogic/pi-mono))

---

## Executive Summary

This document synthesizes findings from deep analysis of the Pi agent framework to inform the design of a **minimal generic agent** with the following characteristics:

| Attribute | Target |
|-----------|--------|
| **Provider** | Azure AI Cloud Foundry only |
| **Core Size** | ~500 LOC (agent loop + types) |
| **Architecture** | Layered, composition-based |
| **Extensibility** | Runtime plugin injection |
| **Self-Improvement** | Instruction accumulation (Skills, Memory) |
| **Safety** | Schema validation, policy hooks, context management |

**Key Insight**: Pi achieves its "magical" quality through three principles:
1. **Tiny core, maximum composability** — The agent runtime is ~400 LOC; everything else is pluggable
2. **Self-modification via Skills/Memory** — The agent accumulates instructions based on feedback
3. **Event-driven architecture** — Fine-grained lifecycle events enable reactive UIs and behavior injection

---

## 1. Layered Architecture

### Proposed Layer Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    Generic Agent Application                     │
│           (Domain tools, UI, workflows, orchestration)          │
├─────────────────────────────────────────────────────────────────┤
│                    Extension/Plugin Layer                        │
│     (Guardrails, approvals, transformers, custom tools)         │
├─────────────────────────────────────────────────────────────────┤
│                    Policy & Safety Layer                         │
│     (Schema validation, rate limiting, context management)      │
├─────────────────────────────────────────────────────────────────┤
│                    Agent Core                                    │
│           (Agent loop, tool execution, events)                  │
├─────────────────────────────────────────────────────────────────┤
│                    AI/LLM Abstraction                            │
│           (Azure-only streaming, message types)                 │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility | LOC Estimate |
|-------|---------------|--------------|
| **AI/LLM** | Azure streaming, message types, tool schemas | ~800 |
| **Agent Core** | Agent loop, tool execution, EventStream | ~500 |
| **Policy/Safety** | Validation, retry, compaction, sanitization | ~600 |
| **Extension** | Plugin registration, event hooks, tool wrapping | ~400 |
| **Application** | Domain logic, UI bindings, session management | Variable |

**Total core framework**: ~2,300 LOC (excluding application layer)

---

## 2. AI/LLM Abstraction Layer

### 2.1 Core Type Definitions

The foundation rests on **provider-agnostic message types**:

```typescript
// Content block types (composable)
interface TextContent {
  type: "text";
  text: string;
  textSignature?: string;  // For replay/caching
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
  thinkingSignature?: string;
}

interface ImageContent {
  type: "image";
  data: string;       // base64
  mimeType: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, unknown>;
}

// Message types
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  usage: Usage;
  stopReason: StopReason;
  errorMessage?: string;
  timestamp: number;
}

interface ToolResultMessage<TDetails = unknown> {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: TDetails;
  isError: boolean;
  timestamp: number;
}

type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

### 2.2 Tool Schema Definition

Tools are defined with **TypeBox schemas** (compile-time TypeScript, runtime JSON Schema):

```typescript
import { Type, type TSchema, type Static } from "@sinclair/typebox";

interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}

// Example tool definition
const searchToolSchema = Type.Object({
  query: Type.String({ description: "Search query", minLength: 1 }),
  limit: Type.Optional(Type.Number({ minimum: 1, maximum: 100 })),
});

const searchTool: Tool<typeof searchToolSchema> = {
  name: "search",
  description: "Search the knowledge base",
  parameters: searchToolSchema,
};
```

### 2.3 EventStream Pattern

A **push/pull hybrid** for backpressure-free streaming:

```typescript
class EventStream<T, R = T> implements AsyncIterable<T> {
  private queue: T[] = [];
  private waiting: Array<(result: IteratorResult<T>) => void> = [];
  private done = false;
  private finalResultPromise: Promise<R>;

  constructor(
    private isComplete: (event: T) => boolean,
    private extractResult: (event: T) => R,
  ) {}

  push(event: T): void {
    if (this.isComplete(event)) {
      this.done = true;
      this.resolveFinalResult(this.extractResult(event));
    }
    const waiter = this.waiting.shift();
    if (waiter) {
      waiter({ value: event, done: false });
    } else {
      this.queue.push(event);
    }
  }

  async *[Symbol.asyncIterator](): AsyncIterator<T> {
    while (true) {
      if (this.queue.length > 0) {
        yield this.queue.shift()!;
      } else if (this.done) {
        return;
      } else {
        const result = await new Promise<IteratorResult<T>>(
          (resolve) => this.waiting.push(resolve)
        );
        if (result.done) return;
        yield result.value;
      }
    }
  }

  result(): Promise<R> {
    return this.finalResultPromise;
  }
}
```

### 2.4 Streaming Event Types

```typescript
type AssistantMessageEvent =
  | { type: "start"; partial: AssistantMessage }
  | { type: "text_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "text_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "text_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "thinking_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "thinking_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "thinking_end"; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: "toolcall_start"; contentIndex: number; partial: AssistantMessage }
  | { type: "toolcall_delta"; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: "toolcall_end"; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: "done"; reason: "stop" | "length" | "toolUse"; message: AssistantMessage }
  | { type: "error"; reason: "aborted" | "error"; error: AssistantMessage };

type AssistantMessageEventStream = EventStream<AssistantMessageEvent, AssistantMessage>;
```

### 2.5 Usage Tracking

```typescript
interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}

type StopReason = "stop" | "length" | "toolUse" | "error" | "aborted";
```

---

## 3. Azure AI Integration

### 3.1 Provider Architecture

Pi already has a production-ready **Azure OpenAI Responses** provider. For an Azure-only agent:

```typescript
// Simplified model interface (Azure-specific)
interface AzureModel {
  id: string;
  deploymentName: string;
  reasoning: boolean;
  maxTokens: number;
  contextWindow: number;
  cost: {
    input: number;   // $/million tokens
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };
}

// Azure-specific options
interface AzureStreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  baseUrl?: string;
  apiVersion?: string;
  deploymentName?: string;
  reasoningEffort?: "minimal" | "low" | "medium" | "high" | "xhigh";
  sessionId?: string;  // For prompt caching
}
```

### 3.2 Environment Configuration

```bash
# Required
AZURE_OPENAI_API_KEY=your-api-key
AZURE_OPENAI_BASE_URL=https://your-resource.openai.azure.com/openai/v1

# Alternative to BASE_URL
AZURE_OPENAI_RESOURCE_NAME=your-resource

# Optional
AZURE_OPENAI_API_VERSION=2024-12-01-preview
AZURE_OPENAI_DEPLOYMENT_NAME_MAP=gpt-4o=my-gpt4o,gpt-4o-mini=my-mini
```

### 3.3 Stream Function Signature

```typescript
function streamAzure(
  model: AzureModel,
  context: Context,
  options?: AzureStreamOptions,
): AssistantMessageEventStream;

// High-level API
async function complete(
  model: AzureModel,
  context: Context,
  options?: AzureStreamOptions,
): Promise<AssistantMessage>;
```

### 3.4 Azure OpenAI Event Mapping

| Azure OpenAI Event | Agent Event |
|-------------------|-------------|
| `response.output_item.added` (reasoning) | `thinking_start` |
| `response.output_text.delta` (reasoning) | `thinking_delta` |
| `response.output_item.done` (reasoning) | `thinking_end` |
| `response.output_item.added` (message) | `text_start` |
| `response.output_text.delta` | `text_delta` |
| `response.output_item.done` (message) | `text_end` |
| `response.output_item.added` (function_call) | `toolcall_start` |
| `response.function_call_arguments.delta` | `toolcall_delta` |
| `response.output_item.done` (function_call) | `toolcall_end` |
| `response.completed` | `done` |
| Error/abort | `error` |

### 3.5 Authentication Patterns

For Azure AI Foundry expansion, consider:

| Auth Method | Use Case |
|-------------|----------|
| **API Key** | Simple deployments, dev/test |
| **Managed Identity** | Production Azure workloads |
| **Entra ID (Azure AD)** | Enterprise SSO integration |
| **Key Vault** | Secure credential rotation |

```typescript
// Credential resolution interface
interface CredentialResolver {
  getApiKey(): Promise<string | undefined>;
  getAccessToken?(): Promise<string | undefined>;
}

// Azure-specific implementation
class AzureCredentialResolver implements CredentialResolver {
  async getApiKey(): Promise<string | undefined> {
    return process.env.AZURE_OPENAI_API_KEY;
  }
  
  async getAccessToken(): Promise<string | undefined> {
    // Use @azure/identity DefaultAzureCredential for Managed Identity
    const credential = new DefaultAzureCredential();
    const token = await credential.getToken("https://cognitiveservices.azure.com/.default");
    return token?.token;
  }
}
```

---

## 4. Agent Core Architecture

### 4.1 Nested Loop Pattern

The agent loop uses a **two-level nested structure**:

```
OUTER LOOP (follow-up messages)
│
└── INNER LOOP (tool calls + steering)
    │
    ├── Process pending steering messages
    ├── Stream assistant response
    ├── Execute tool calls (with mid-execution steering check)
    └── Get more steering messages
```

**Algorithm**:

```typescript
async function* agentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
): AgentEventStream {
  yield { type: "agent_start" };
  
  let pendingMessages = prompts;
  
  // Outer loop: follow-up messages
  while (true) {
    let hasMoreToolCalls = true;
    
    // Inner loop: tool calls + steering
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      yield { type: "turn_start" };
      
      // Inject pending messages
      if (pendingMessages.length > 0) {
        context.messages.push(...pendingMessages);
        pendingMessages = [];
      }
      
      // Stream assistant response
      const assistantMessage = await streamAssistantResponse(context, config, signal);
      
      // Execute tool calls
      if (hasToolCalls(assistantMessage)) {
        const { toolResults, steeringMessages } = await executeToolCalls(
          context.tools,
          assistantMessage,
          signal,
          config.getSteeringMessages,
        );
        context.messages.push(...toolResults);
        pendingMessages = steeringMessages;
        hasMoreToolCalls = hasMoreToolCalls(toolResults);
      } else {
        hasMoreToolCalls = false;
      }
      
      yield { type: "turn_end", message: assistantMessage, toolResults };
      
      // Check for steering messages
      if (pendingMessages.length === 0) {
        pendingMessages = await config.getSteeringMessages?.() ?? [];
      }
    }
    
    // Check for follow-up messages
    const followUp = await config.getFollowUpMessages?.() ?? [];
    if (followUp.length > 0) {
      pendingMessages = followUp;
      continue;
    }
    break;
  }
  
  yield { type: "agent_end", messages: context.messages };
}
```

### 4.2 Configuration Interface

```typescript
interface AgentLoopConfig {
  model: AzureModel;
  
  // REQUIRED: Transform agent messages to LLM-compatible format
  convertToLlm: (messages: AgentMessage[]) => Message[] | Promise<Message[]>;
  
  // OPTIONAL: Modify context before LLM call (pruning, injection)
  transformContext?: (messages: AgentMessage[], signal?: AbortSignal) => Promise<AgentMessage[]>;
  
  // OPTIONAL: Dynamic credential resolution
  getApiKey?: () => Promise<string | undefined>;
  
  // OPTIONAL: Mid-turn interruption
  getSteeringMessages?: () => Promise<AgentMessage[]>;
  
  // OPTIONAL: Post-completion continuation
  getFollowUpMessages?: () => Promise<AgentMessage[]>;
}
```

### 4.3 Steering vs Follow-Up Messages

| Mechanism | When Called | Purpose | Effect |
|-----------|-------------|---------|--------|
| `getSteeringMessages` | After each tool execution | Interrupt mid-run | Skips remaining tools, injects new context |
| `getFollowUpMessages` | Only when agent would stop | Queue additional work | Re-enters loop with new prompts |

### 4.4 Tool Execution Interface

```typescript
interface AgentTool<TParameters extends TSchema, TDetails = unknown> extends Tool<TParameters> {
  label: string;  // Human-readable for UI
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: (partialResult: Partial<AgentToolResult<TDetails>>) => void,
  ) => Promise<AgentToolResult<TDetails>>;
}

interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];
  details: T;  // Typed metadata for UI/logging
}
```

### 4.5 Agent Events

```typescript
type AgentEvent =
  // Agent lifecycle
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  
  // Turn lifecycle
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  
  // Message streaming
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; event: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  
  // Tool execution
  | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: unknown }
  | { type: "tool_execution_update"; toolCallId: string; partialResult: unknown }
  | { type: "tool_execution_end"; toolCallId: string; result: AgentToolResult<unknown>; isError: boolean };

type AgentEventStream = EventStream<AgentEvent, AgentMessage[]>;
```

### 4.6 Message Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    AgentMessage[]                                │
│  (user, assistant, toolResult, + custom application messages)   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   transformContext()                             │
│  • Context window pruning                                        │
│  • Inject external context (skills, memory)                      │
│  • Filter stale/irrelevant messages                              │
│  → Still AgentMessage[]                                          │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     convertToLlm()                               │
│  • Filter to LLM-compatible messages                             │
│  • Convert custom message types                                  │
│  → Message[]                                                     │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│            Context { systemPrompt, messages, tools }             │
│                         → Azure LLM                              │
└─────────────────────────────────────────────────────────────────┘
```

### 4.7 Composition via Declaration Merging

Allow applications to extend message types without modifying core:

```typescript
// In agent-core (empty by default)
interface CustomAgentMessages {}

// In application code (via declaration merging)
declare module "agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string; severity: "info" | "warn" | "error" };
    approval: { role: "approval"; toolName: string; approved: boolean };
    artifact: { role: "artifact"; type: string; data: unknown };
  }
}

// AgentMessage automatically includes custom types
type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

---

## 5. Policy & Safety Layer

### 5.1 Schema Validation (TypeBox + AJV)

```typescript
import Ajv from "ajv";
import addFormats from "ajv-formats";

const ajv = new Ajv({
  allErrors: true,    // Report all errors
  strict: false,      // Lenient schema mode
  coerceTypes: true,  // "123" → 123
});
addFormats(ajv);

function validateToolArguments<T extends TSchema>(
  tool: Tool<T>,
  toolCall: ToolCall,
): Static<T> {
  const args = structuredClone(toolCall.arguments);  // AJV mutates
  const validate = ajv.compile(tool.parameters);
  
  if (!validate(args)) {
    const errors = validate.errors?.map(e => `${e.instancePath}: ${e.message}`).join("\n");
    throw new Error(`Validation failed for tool "${toolCall.name}":\n${errors}`);
  }
  
  return args as Static<T>;
}
```

### 5.2 Retryable Error Detection

```typescript
const RETRYABLE_PATTERNS = [
  /overloaded|rate.?limit|too many requests/i,
  /429|500|502|503|504/,
  /service.?unavailable|server error|internal error/i,
  /connection.?error|connection.?refused|fetch failed/i,
  /upstream.?connect|reset before headers|terminated/i,
  /retry delay/i,
];

function isRetryableError(message: AssistantMessage, contextWindow: number): boolean {
  if (message.stopReason !== "error" || !message.errorMessage) return false;
  
  // Context overflow → compaction, NOT retry
  if (isContextOverflow(message.errorMessage, contextWindow)) return false;
  
  return RETRYABLE_PATTERNS.some(p => p.test(message.errorMessage!));
}
```

### 5.3 Exponential Backoff with Server Hints

```typescript
interface RetrySettings {
  enabled: boolean;
  maxRetries: number;      // Default: 3
  baseDelayMs: number;     // Default: 1000
  maxDelayMs: number;      // Default: 60000
}

function extractRetryDelay(errorText: string, headers?: Headers): number | undefined {
  // Priority 1: Retry-After header
  const retryAfter = headers?.get("retry-after");
  if (retryAfter) {
    const seconds = parseInt(retryAfter, 10);
    if (!isNaN(seconds)) return seconds * 1000;
  }
  
  // Priority 2: x-ratelimit-reset header
  const rateLimitReset = headers?.get("x-ratelimit-reset");
  if (rateLimitReset) {
    const resetTime = parseInt(rateLimitReset, 10) * 1000;
    return Math.max(0, resetTime - Date.now());
  }
  
  // Priority 3: Parse from error text
  const match = errorText.match(/retry\s*(?:in|after)\s*(\d+)\s*s/i);
  if (match) return parseInt(match[1], 10) * 1000;
  
  return undefined;
}

async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  settings: RetrySettings,
  signal?: AbortSignal,
): Promise<T> {
  let attempt = 0;
  
  while (true) {
    try {
      return await fn();
    } catch (error) {
      attempt++;
      if (attempt > settings.maxRetries) throw error;
      if (!isRetryableError(error)) throw error;
      
      const serverDelay = extractRetryDelay(error.message);
      const backoffDelay = settings.baseDelayMs * Math.pow(2, attempt - 1);
      const delay = Math.min(serverDelay ?? backoffDelay, settings.maxDelayMs);
      
      await sleep(delay, signal);
    }
  }
}
```

### 5.4 Context Overflow Detection

```typescript
const OVERFLOW_PATTERNS = [
  /prompt is too long/i,                     // Anthropic
  /input is too long for requested model/i,  // Bedrock
  /exceeds the context window/i,             // OpenAI
  /maximum context length is \d+ tokens/i,   // Various
  /reduce the length of the messages/i,      // Groq
];

function isContextOverflow(errorMessage: string, contextWindow: number): boolean {
  return OVERFLOW_PATTERNS.some(p => p.test(errorMessage));
}

interface CompactionSettings {
  enabled: boolean;
  reserveTokens: number;     // Default: 16384
  keepRecentTokens: number;  // Default: 20000
}

function shouldCompact(
  contextTokens: number,
  contextWindow: number,
  settings: CompactionSettings,
): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

### 5.5 Message Sanitization

```typescript
// Handle orphaned tool calls (tool call without result)
function insertSyntheticToolResults(messages: Message[]): Message[] {
  const toolCallIds = new Set<string>();
  const toolResultIds = new Set<string>();
  
  for (const msg of messages) {
    if (msg.role === "assistant") {
      for (const c of msg.content) {
        if (c.type === "toolCall") toolCallIds.add(c.id);
      }
    } else if (msg.role === "toolResult") {
      toolResultIds.add(msg.toolCallId);
    }
  }
  
  const orphanedIds = [...toolCallIds].filter(id => !toolResultIds.has(id));
  const syntheticResults: ToolResultMessage[] = orphanedIds.map(id => ({
    role: "toolResult",
    toolCallId: id,
    toolName: "unknown",
    content: [{ type: "text", text: "Tool execution was interrupted" }],
    isError: true,
    timestamp: Date.now(),
  }));
  
  return [...messages, ...syntheticResults];
}

// Remove unpaired Unicode surrogates (breaks JSON serialization)
function sanitizeSurrogates(text: string): string {
  return text.replace(
    /[\uD800-\uDBFF](?![\uDC00-\uDFFF])|(?<![\uD800-\uDBFF])[\uDC00-\uDFFF]/g,
    ""
  );
}

// Filter errored/aborted messages (can cause replay issues)
function filterErroredMessages(messages: Message[]): Message[] {
  return messages.filter(msg => {
    if (msg.role === "assistant") {
      return msg.stopReason !== "error" && msg.stopReason !== "aborted";
    }
    return true;
  });
}
```

---

## 6. Extension Layer

### 6.1 Extension Factory Pattern

```typescript
type ExtensionFactory = (api: ExtensionAPI) => void | Promise<void>;

// Extensions are simple functions that register handlers
const myExtension: ExtensionFactory = (api) => {
  // Register event handlers
  api.on("tool_call", async (event, ctx) => { /* ... */ });
  
  // Register custom tools
  api.registerTool({ /* ... */ });
  
  // Register commands/shortcuts
  api.registerCommand("my-command", { /* ... */ });
};
```

### 6.2 Extension Hook Points

| Event | Description | Result Type |
|-------|-------------|-------------|
| `context` | Before LLM call — modify messages | `{ messages?: AgentMessage[] }` |
| `before_agent_start` | After user submits — modify prompt | `{ systemPrompt?: string; message?: CustomMessage }` |
| `agent_start` | Agent loop started | – |
| `agent_end` | Agent loop ended | – |
| `turn_start` | LLM turn started | – |
| `turn_end` | LLM turn ended | – |
| `input` | User input received | `{ action: "continue" \| "transform" \| "handled"; text?: string }` |
| `tool_call` | Before tool executes | `{ block?: boolean; reason?: string }` |
| `tool_result` | After tool executes | `{ content?: Content[]; details?: unknown }` |
| `session_start` | Session initialized | – |
| `session_shutdown` | Process exiting | – |

### 6.3 Extension API

```typescript
interface ExtensionAPI {
  // Event subscription (strongly typed)
  on<E extends ExtensionEvent>(
    event: E,
    handler: ExtensionHandler<E>,
  ): void;

  // Tool registration
  registerTool<TParams extends TSchema, TDetails>(
    tool: ToolDefinition<TParams, TDetails>,
  ): void;

  // Command/shortcut registration
  registerCommand(name: string, options: CommandOptions): void;
  registerShortcut(key: string, options: ShortcutOptions): void;

  // Runtime actions
  sendMessage(message: CustomMessage): void;
  abort(): void;

  // State access
  getActiveTools(): string[];
  setActiveTools(names: string[]): void;
  getThinkingLevel(): ThinkingLevel;
  setThinkingLevel(level: ThinkingLevel): void;
}
```

### 6.4 Extension Context (Passed to Handlers)

```typescript
interface ExtensionContext {
  ui: ExtensionUI;           // Dialogs, notifications
  hasUI: boolean;            // false in headless mode
  cwd: string;               // Working directory
  isIdle(): boolean;         // Agent streaming?
  abort(): void;             // Abort current operation
  getContextUsage(): Usage;  // Token usage
  getSystemPrompt(): string;
}

interface ExtensionUI {
  confirm(message: string): Promise<boolean>;
  select<T>(options: SelectOptions<T>): Promise<T | undefined>;
  input(options: InputOptions): Promise<string | undefined>;
  notify(message: string, severity: "info" | "warn" | "error"): void;
}
```

### 6.5 Tool Wrapping (Interception Pattern)

```typescript
function wrapToolWithExtensions<T>(
  tool: AgentTool<TSchema, T>,
  runner: ExtensionRunner,
): AgentTool<TSchema, T> {
  return {
    ...tool,
    execute: async (toolCallId, params, signal, onUpdate) => {
      // Before hook — can block execution
      if (runner.hasHandlers("tool_call")) {
        const result = await runner.emit("tool_call", {
          toolName: tool.name,
          toolCallId,
          input: params,
        });
        
        if (result?.block) {
          throw new Error(result.reason ?? "Blocked by extension");
        }
      }

      // Execute actual tool
      const result = await tool.execute(toolCallId, params, signal, onUpdate);

      // After hook — can modify result
      if (runner.hasHandlers("tool_result")) {
        const modified = await runner.emit("tool_result", {
          toolName: tool.name,
          toolCallId,
          input: params,
          content: result.content,
          details: result.details,
          isError: false,
        });
        
        if (modified) {
          return {
            content: modified.content ?? result.content,
            details: (modified.details ?? result.details) as T,
          };
        }
      }

      return result;
    },
  };
}
```

### 6.6 Guardrail Patterns

**Blocking Dangerous Operations:**

```typescript
api.on("tool_call", async (event, ctx) => {
  if (event.toolName === "execute" && isDangerous(event.input.command)) {
    const confirmed = await ctx.ui.confirm(
      `Allow potentially dangerous command?\n${event.input.command}`
    );
    if (!confirmed) {
      return { block: true, reason: "User declined dangerous operation" };
    }
  }
  return undefined;  // Allow
});
```

**Input Transformation:**

```typescript
api.on("input", async (event, ctx) => {
  // Expand aliases
  if (event.text.startsWith("/")) {
    const expanded = expandAlias(event.text);
    if (expanded) {
      return { action: "transform", text: expanded };
    }
  }
  
  // Handle without LLM
  if (event.text === "/help") {
    ctx.ui.notify(HELP_TEXT, "info");
    return { action: "handled" };
  }
  
  return { action: "continue" };
});
```

**Context Modification:**

```typescript
api.on("context", async (event) => {
  return {
    messages: event.messages.filter(m => {
      // Remove stale context markers
      if (m.customType === "temporary-context") return false;
      return true;
    }),
  };
});
```

---

## 7. Learning & Feedback Mechanisms

### 7.1 Overview: "Implicit RL" Pattern

Pi achieves self-improvement through **instruction accumulation**, not traditional RL:

| Mechanism | Storage | Trigger | Effect |
|-----------|---------|---------|--------|
| User correction | MEMORY.md | "remember X" | Future sessions see instruction |
| Workflow pattern | SKILL.md | "create skill for X" | New capability available |
| Session history | JSONL | Automatic | Context preserved |
| Compaction | Summary | Context overflow | Key decisions preserved |

**No reward signals, no gradients, no fine-tuning** — just persistent natural language instructions.

### 7.2 Memory System

**Location**: `MEMORY.md` files at project and global levels

**Format**:

```markdown
# Memory

## Preferences
- User prefers concise responses
- Always confirm before destructive operations

## Context
- Primary language: TypeScript
- Framework: Azure Functions

## Learned Patterns
- When user says "deploy", they mean Azure deployment
- "Quick check" means skip detailed explanations
```

**Integration**:

```typescript
async function loadMemory(cwd: string, globalDir: string): Promise<string> {
  const projectMemory = await readFile(join(cwd, "MEMORY.md")).catch(() => "");
  const globalMemory = await readFile(join(globalDir, "MEMORY.md")).catch(() => "");
  
  return [globalMemory, projectMemory].filter(Boolean).join("\n\n");
}

// Inject into system prompt
function buildSystemPrompt(basePrompt: string, memory: string): string {
  if (!memory) return basePrompt;
  return `${basePrompt}\n\n## Remembered Context\n\n${memory}`;
}
```

**Accumulation Pattern**:

When user says "remember that I prefer X", the agent appends to MEMORY.md:

```typescript
api.on("tool_call", async (event, ctx) => {
  if (event.toolName === "remember") {
    const content = `- ${event.input.text}\n`;
    await appendFile(join(ctx.cwd, "MEMORY.md"), content);
    return { content: [{ type: "text", text: "Remembered." }], details: {} };
  }
});
```

### 7.3 Skill System

**Format**: Self-contained capability packages

```yaml
---
name: data-analysis
description: Analyze datasets and generate insights
tools:
  - query_database
  - generate_chart
---
# Instructions

When analyzing data:
1. First understand the schema
2. Form hypotheses
3. Query to validate
4. Visualize findings

# Examples

User: "What's our monthly trend?"
→ Query sales data, group by month, generate line chart
```

**Discovery**:

```typescript
interface Skill {
  name: string;
  description: string;
  tools?: string[];
  content: string;  // Full markdown content
}

async function discoverSkills(paths: string[]): Promise<Skill[]> {
  const skills: Skill[] = [];
  
  for (const dir of paths) {
    const files = await glob(join(dir, "*.skill.md"));
    for (const file of files) {
      const content = await readFile(file, "utf-8");
      const { data, content: body } = matter(content);  // Parse YAML frontmatter
      skills.push({
        name: data.name,
        description: data.description,
        tools: data.tools,
        content: body,
      });
    }
  }
  
  return skills;
}
```

**Progressive Disclosure**:

Only skill names/descriptions in system prompt; full content loaded on-demand:

```typescript
// System prompt includes skill index
const skillIndex = skills.map(s => `- ${s.name}: ${s.description}`).join("\n");
const prompt = `${basePrompt}\n\nAvailable skills:\n${skillIndex}\n\nUse load_skill(name) to access full instructions.`;

// Tool loads full content
const loadSkillTool: AgentTool = {
  name: "load_skill",
  description: "Load full instructions for a skill",
  parameters: Type.Object({ name: Type.String() }),
  async execute(_, { name }) {
    const skill = skills.find(s => s.name === name);
    if (!skill) return { content: [{ type: "text", text: "Skill not found" }], details: {}, isError: true };
    return { content: [{ type: "text", text: skill.content }], details: { skill } };
  },
};
```

### 7.4 Session Persistence

**Format**: Append-only JSONL with tree structure

```jsonl
{"type":"session","version":1,"id":"abc123","timestamp":"2026-02-03T10:00:00Z"}
{"type":"message","id":"m1","parentId":null,"message":{"role":"user","content":"Hello"}}
{"type":"message","id":"m2","parentId":"m1","message":{"role":"assistant","content":"Hi!"}}
{"type":"branch","id":"b1","parentId":"m2","label":"Alternative approach"}
{"type":"message","id":"m3","parentId":"m2","message":{"role":"user","content":"Try again"}}
{"type":"compaction","id":"c1","parentId":"m3","summary":"...","firstKeptId":"m3"}
```

**Tree Operations**:

```typescript
interface SessionEntry {
  type: "session" | "message" | "branch" | "compaction" | "custom";
  id: string;
  parentId: string | null;
  timestamp: string;
  // Type-specific fields
}

// Get path from root to leaf
function getPathToLeaf(entries: SessionEntry[], leafId: string): SessionEntry[] {
  const byId = new Map(entries.map(e => [e.id, e]));
  const path: SessionEntry[] = [];
  
  let current = byId.get(leafId);
  while (current) {
    path.unshift(current);
    current = current.parentId ? byId.get(current.parentId) : undefined;
  }
  
  return path;
}

// Branch: set new leaf without modifying history
function branch(entries: SessionEntry[], fromId: string, label: string): SessionEntry {
  return {
    type: "branch",
    id: generateId(),
    parentId: fromId,
    timestamp: new Date().toISOString(),
    label,
  };
}
```

### 7.5 Compaction Algorithm

**Trigger**: Context tokens > (context window - reserve tokens)

**Algorithm**:

```typescript
interface CompactionResult {
  summary: string;
  firstKeptId: string;
  tokensBefore: number;
  tokensAfter: number;
}

async function compact(
  messages: AgentMessage[],
  model: AzureModel,
  settings: CompactionSettings,
): Promise<CompactionResult> {
  // Find cut point that keeps ~keepRecentTokens
  const cutIndex = findCutPoint(messages, settings.keepRecentTokens);
  const toSummarize = messages.slice(0, cutIndex);
  const toKeep = messages.slice(cutIndex);
  
  // Generate structured summary
  const summary = await generateSummary(toSummarize, model);
  
  return {
    summary,
    firstKeptId: toKeep[0].id,
    tokensBefore: countTokens(messages),
    tokensAfter: countTokens(toKeep) + countTokens(summary),
  };
}
```

**Summary Format**:

```markdown
## Conversation Summary

### Goal
Build a data pipeline for customer analytics

### Progress
**Done:**
- Set up Azure Data Factory
- Created initial schema

**In Progress:**
- Implementing transformation logic

**Blocked:**
- Waiting for API credentials

### Key Decisions
- Using Parquet format for storage
- Incremental load strategy

### Next Steps
1. Complete transformations
2. Set up monitoring
```

---

## 8. MCP (Model Context Protocol) Integration

### 8.1 Overview

MCP provides a standardized way to expose tools to AI agents:

```typescript
interface MCPServer {
  listTools(): Promise<MCPTool[]>;
  callTool(name: string, args: Record<string, unknown>): Promise<MCPToolResult>;
}

interface MCPTool {
  name: string;
  description: string;
  inputSchema: JSONSchema;
}
```

### 8.2 Integration Pattern

```typescript
async function loadMCPTools(serverConfig: MCPServerConfig): Promise<AgentTool[]> {
  const client = await connectMCP(serverConfig);
  const mcpTools = await client.listTools();
  
  return mcpTools.map(tool => ({
    name: tool.name,
    label: tool.name,
    description: tool.description,
    parameters: tool.inputSchema as TSchema,
    async execute(toolCallId, params, signal) {
      const result = await client.callTool(tool.name, params);
      return {
        content: [{ type: "text", text: JSON.stringify(result) }],
        details: { mcpResult: result },
      };
    },
  }));
}
```

### 8.3 Server Discovery

```typescript
interface MCPServerConfig {
  name: string;
  transport: "stdio" | "http";
  command?: string;     // For stdio
  args?: string[];
  url?: string;         // For http
  env?: Record<string, string>;
}

// Configuration file: mcp.json
{
  "servers": {
    "database": {
      "transport": "stdio",
      "command": "npx",
      "args": ["@my-org/db-mcp-server"],
      "env": { "DB_CONNECTION": "..." }
    },
    "api": {
      "transport": "http",
      "url": "http://localhost:3001/mcp"
    }
  }
}
```

---

## 9. Implementation Roadmap

### Phase 1: Core Foundation (Week 1-2)

| Component | Deliverable |
|-----------|-------------|
| Types | Message types, tool schemas, events |
| Azure Provider | Streaming, auth, error handling |
| EventStream | Push/pull implementation |
| Validation | TypeBox + AJV integration |

### Phase 2: Agent Core (Week 3-4)

| Component | Deliverable |
|-----------|-------------|
| Agent Loop | Nested loop with steering/follow-up |
| Tool Execution | Validation, execution, result handling |
| Error Recovery | Retry logic, overflow detection |
| Events | Full event taxonomy |

### Phase 3: Extension System (Week 5-6)

| Component | Deliverable |
|-----------|-------------|
| Extension API | Registration, hooks, actions |
| Tool Wrapping | Interception pattern |
| Extension Runner | Event dispatch, handler management |
| Built-in Extensions | Basic guardrails |

### Phase 4: Learning & Persistence (Week 7-8)

| Component | Deliverable |
|-----------|-------------|
| Memory System | MEMORY.md read/write |
| Skill System | Discovery, loading, progressive disclosure |
| Session Persistence | JSONL tree format |
| Compaction | Summary generation, threshold management |

### Phase 5: Integration & Polish (Week 9-10)

| Component | Deliverable |
|-----------|-------------|
| MCP Integration | Server discovery, tool loading |
| Testing | Unit tests, integration tests |
| Documentation | API docs, usage guides |
| Examples | Reference implementations |

---

## 10. Key Design Decisions

### 10.1 Single Provider (Azure) Simplification

**Rationale**: Reduces complexity by ~60%

| Removed | Impact |
|---------|--------|
| Provider registry | No dispatch logic |
| Cross-provider transforms | No message normalization |
| Model generation scripts | Static model definitions |
| Multiple auth patterns | Azure-only credentials |

### 10.2 Composition Over Inheritance

**Rationale**: Maximum flexibility without class hierarchies

```typescript
// ❌ Inheritance (avoided)
class MyAgent extends BaseAgent {
  override transformContext() { /* ... */ }
}

// ✅ Composition (preferred)
const agent = createAgent({
  transformContext: myTransformer,
  getSteeringMessages: mySteeringFn,
});
```

### 10.3 Event-Driven Architecture

**Rationale**: Decouples UI from agent logic

- UI subscribes to events, renders accordingly
- Extensions hook into events, modify behavior
- Agent core emits events, remains pure

### 10.4 Stateless Loop + Stateful Wrapper

**Rationale**: Testability + Convenience

```typescript
// Stateless for testing
const stream = agentLoop(prompts, context, config);
for await (const event of stream) { /* verify */ }

// Stateful for applications
const agent = new Agent(config);
agent.subscribe(renderUI);
await agent.prompt("Hello");
```

### 10.5 Instruction Accumulation (Not RL)

**Rationale**: Predictable, auditable, no training required

- Users see exactly what the agent "learned"
- No opaque model changes
- Easy to edit/remove learned behaviors
- Works across sessions without state servers

---

## 11. Risk Analysis

| Risk | Mitigation |
|------|------------|
| Azure API changes | Pin API version, maintain compatibility layer |
| Context overflow at scale | Aggressive compaction, structured summaries |
| Extension security | Sandboxing, permission model, code review |
| Memory/skill injection attacks | Sanitization, validation, user confirmation |
| Cost overruns | Usage tracking, budgets, alerts |

---

## 12. References

### Pi Codebase (Key Files)

| Component | Location |
|-----------|----------|
| Agent Loop | `packages/agent/src/agent-loop.ts` |
| Agent Class | `packages/agent/src/agent.ts` |
| Types | `packages/agent/src/types.ts` |
| LLM Streaming | `packages/ai/src/stream.ts` |
| Azure Provider | `packages/ai/src/providers/azure-openai-responses.ts` |
| Message Transform | `packages/ai/src/providers/transform-messages.ts` |
| Validation | `packages/ai/src/utils/validate-tool-arguments.ts` |
| Extension Types | `packages/coding-agent/src/extensions/types.ts` |
| Extension Runtime | `packages/coding-agent/src/extensions/runtime.ts` |
| Tool Wrapping | `packages/coding-agent/src/extensions/wrap-tool-with-extensions.ts` |
| Compaction | `packages/coding-agent/src/core/compaction.ts` |
| Session Manager | `packages/coding-agent/src/core/session-manager.ts` |
| Skill System | `packages/coding-agent/src/core/skills.ts` |

### External References

- [TypeBox](https://github.com/sinclairzx81/typebox) — Runtime type validation
- [AJV](https://ajv.js.org/) — JSON Schema validator
- [MCP Specification](https://modelcontextprotocol.io/) — Tool protocol
- [Azure OpenAI](https://learn.microsoft.com/en-us/azure/ai-services/openai/) — Provider docs

---

## Appendix A: Minimal Code Skeleton

```typescript
// ai/types.ts — ~200 LOC
export interface TextContent { type: "text"; text: string; }
export interface ToolCall { type: "toolCall"; id: string; name: string; arguments: Record<string, unknown>; }
export interface UserMessage { role: "user"; content: string | Content[]; timestamp: number; }
export interface AssistantMessage { role: "assistant"; content: Content[]; usage: Usage; stopReason: StopReason; timestamp: number; }
export interface ToolResultMessage { role: "toolResult"; toolCallId: string; content: Content[]; isError: boolean; timestamp: number; }
export type Message = UserMessage | AssistantMessage | ToolResultMessage;

// ai/stream.ts — ~300 LOC
export class EventStream<T, R> implements AsyncIterable<T> { /* ... */ }
export function streamAzure(model: AzureModel, context: Context, options?: AzureOptions): AssistantMessageEventStream { /* ... */ }

// agent/loop.ts — ~400 LOC
export async function* agentLoop(prompts: AgentMessage[], context: AgentContext, config: AgentLoopConfig): AgentEventStream { /* ... */ }

// agent/agent.ts — ~300 LOC (optional)
export class Agent { /* Stateful wrapper */ }

// extensions/types.ts — ~150 LOC
export type ExtensionFactory = (api: ExtensionAPI) => void | Promise<void>;
export interface ExtensionAPI { on, registerTool, registerCommand, ... }

// extensions/runner.ts — ~250 LOC
export class ExtensionRunner { emit, hasHandlers, wrap, ... }
```

**Total Core**: ~1,600 LOC

---

## Appendix B: Example Extension

```typescript
// guardrails.extension.ts
import type { ExtensionFactory } from "agent-core";

const guardrailsExtension: ExtensionFactory = (api) => {
  // Block dangerous patterns
  api.on("tool_call", async (event, ctx) => {
    const blockedPatterns = [
      /rm\s+-rf\s+\//,
      /DROP\s+DATABASE/i,
      /DELETE\s+FROM\s+\w+\s*;/i,
    ];
    
    const input = JSON.stringify(event.input);
    for (const pattern of blockedPatterns) {
      if (pattern.test(input)) {
        const confirmed = await ctx.ui.confirm(
          `⚠️ Potentially dangerous operation detected:\n${input}\n\nProceed?`
        );
        if (!confirmed) {
          return { block: true, reason: "Blocked by guardrails" };
        }
      }
    }
    
    return undefined;
  });

  // Log all tool executions
  api.on("tool_result", async (event) => {
    console.log(`[AUDIT] ${event.toolName}: ${event.isError ? "FAILED" : "OK"}`);
    return undefined;
  });
};

export default guardrailsExtension;
```

---

*Document generated: 2026-02-03*  
*Research phase: Complete*  
*Next phase: Planning*
