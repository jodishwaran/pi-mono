# Pi Agent Harness: Comprehensive Research Report

## Executive Summary

Pi is a **minimalist, composable agent framework** that achieves its "magical" quality through three core principles:

1. **Tiny Core, Maximum Composability** - The actual agent runtime is ~400 lines; everything else is pluggable
2. **Self-Modification via Skills/Extensions** - The agent can write tools and instructions for itself
3. **Event-Driven Architecture** - Fine-grained lifecycle events enable reactive UIs and behavior injection

The quote *"able to write plugins for itself as you use it"* is accurate - Pi's extension system + skill creation allows the agent to modify its own capabilities at runtime.

---

## 1. Architecture Overview

### Package Dependency Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                    @mariozechner/pi-ai                           │
│           (Unified LLM API - Foundation Layer)                   │
│   - 20+ providers (Anthropic, OpenAI, Google, Bedrock, etc.)    │
│   - Streaming with standardized events                           │
│   - Cross-provider message transformation                        │
│   - Token counting and cost tracking                             │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────────┐
│               @mariozechner/pi-agent-core                        │
│              (Agent Runtime - Core Primitives)                   │
│   - AgentLoop (pure functions, stateless)                        │
│   - Agent class (stateful wrapper)                               │
│   - Tool execution with validation                               │
│   - Event streaming                                              │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │
┌─────────────────────────────────────────────────────────────────┐
│             @mariozechner/pi-coding-agent                        │
│                  (Full Application Layer)                        │
│   - Extension system (runtime plugin injection)                  │
│   - Skill system (declarative capabilities)                      │
│   - Session persistence (JSONL trees)                            │
│   - Context compaction (auto-summarization)                      │
│   - Tool implementations (read, bash, edit, write, etc.)         │
└─────────────────────────────────────────────────────────────────┘
```

### Minimal Set for Generic Agent

For a **non-coding generic agent**, you need only:

| Package | Size | Purpose |
|---------|------|---------|
| `pi-ai` | Foundation | LLM streaming, providers, message types |
| `pi-agent-core` | ~400 LOC core | Agent loop, tool execution, events |

The `coding-agent` package is application-specific and demonstrates how to compose the primitives.

---

## 2. Core Design Patterns

### Pattern 1: Composition over Inheritance

Pi never subclasses. Everything is composed via **configuration injection**:

```typescript
// The AgentLoopConfig is the composition interface
interface AgentLoopConfig {
  model: Model<any>;
  
  // Transform pipeline (injected functions)
  convertToLlm: (messages: AgentMessage[]) => Message[];
  transformContext?: (messages: AgentMessage[]) => Promise<AgentMessage[]>;
  
  // Interruption hooks
  getSteeringMessages?: () => Promise<AgentMessage[]>;
  getFollowUpMessages?: () => Promise<AgentMessage[]>;
}
```

**Why this matters for generic agents:** You inject behavior through functions, not inheritance. Want different context management? Inject a different `transformContext`. Want approval flows? Inject `getSteeringMessages`.

### Pattern 2: Pure Functions + Stateful Wrapper

Two ways to use the agent:

```typescript
// 1. Pure functions (agentLoop) - full control, no state management
const stream = agentLoop(messages, context, config, signal);
for await (const event of stream) { /* handle manually */ }

// 2. Stateful wrapper (Agent class) - convenience API
const agent = new Agent({ initialState, convertToLlm, ... });
agent.subscribe((event) => { /* reactive updates */ });
await agent.prompt("Do something");
```

**Why this matters:** Pure functions for testing/embedding, class for applications.

### Pattern 3: EventStream (Push/Pull Hybrid)

```typescript
class EventStream<T, R> implements AsyncIterable<T> {
  push(event: T): void {
    // If consumer waiting, deliver immediately
    // Otherwise queue for later
    const waiter = this.waiting.shift();
    if (waiter) waiter({ value: event, done: false });
    else this.queue.push(event);
  }
  
  // Consumer pulls via async iteration
  async *[Symbol.asyncIterator]() {
    while (!this.done) {
      if (this.queue.length > 0) yield this.queue.shift()!;
      else await new Promise(resolve => this.waiting.push(resolve));
    }
  }
}
```

**Why this matters:** Enables streaming without backpressure issues. UI can consume at its own pace.

### Pattern 4: Declaration Merging for Extensibility

Pi allows apps to add custom message types without modifying core:

```typescript
// In pi-agent-core (empty by default)
interface CustomAgentMessages {}

// In your app (via declaration merging)
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string };
    approval: { role: "approval"; toolName: string; approved: boolean };
  }
}
```

**Why this matters:** You can add domain-specific message types that flow through the entire system.

### Pattern 5: Extension System (Runtime Behavior Injection)

Extensions can intercept and modify any lifecycle event:

```typescript
const myExtension: ExtensionFactory = (pi) => {
  // Register custom tools at runtime
  pi.registerTool({
    name: "search_knowledge_base",
    execute: async (id, params, signal) => { /* ... */ }
  });
  
  // Intercept tool calls (guardrails)
  pi.on("tool_call", async (event, ctx) => {
    if (isDangerous(event)) return { block: true, reason: "Blocked" };
  });
  
  // Modify context before LLM calls
  pi.on("context", async (event) => {
    return event.messages.filter(m => !isStale(m));
  });
  
  // Transform user input
  pi.on("input", async (event) => {
    return { action: "transform", text: expandAliases(event.text) };
  });
};
```

**Why this matters:** This is what makes Pi "malleable" - extensions can completely reshape agent behavior.

---

## 3. The "Self-Modification" Capability

The quote mentions Pi can "write plugins for itself." Here's how:

### Skills System

Skills are markdown files following the [Agent Skills standard](https://agentskills.io):

```markdown
---
name: database-query
description: Query the company database
---
# Instructions
When user asks about data, use the database_query tool with SQL...

# Examples
User: "How many users signed up yesterday?"
→ Use: database_query("SELECT COUNT(*) FROM users WHERE created_at > ...")
```

**Pi can create skills for itself** - if you ask "create a skill for managing my todo list," it writes a SKILL.md file that it will then read and follow in future sessions.

### MEMORY.md Files

Pi uses memory files for persistent preferences:

```markdown
# Memory
- User prefers TypeScript over JavaScript
- Always run tests before committing
- Use pnpm, not npm
```

This is **implicit RL** - the agent accumulates instructions based on feedback.

### Dynamic Tool Registration

Extensions can register tools at runtime:

```typescript
pi.registerTool({
  name: "query_jira",
  // ...
});
```

Combined with skill creation, the agent can literally write the code for a new tool, then ask to reload extensions.

---

## 4. Guardrails and Verification

### Tool Interception

Every tool call passes through extension handlers:

```typescript
pi.on("tool_call", async (event, ctx) => {
  // Block dangerous operations
  if (event.toolName === "bash" && /rm -rf/.test(event.input.command)) {
    const ok = await ctx.ui.confirm("Allow dangerous command?");
    if (!ok) return { block: true, reason: "Blocked by user" };
  }
});
```

### Argument Validation

Tools use TypeBox schemas validated by AJV:

```typescript
const myToolSchema = Type.Object({
  query: Type.String({ minLength: 1, maxLength: 1000 }),
  limit: Type.Optional(Type.Number({ minimum: 1, maximum: 100 })),
});

// Validation happens automatically before execute()
```

### Error Recovery

Auto-retry with exponential backoff for transient errors:

```typescript
// Retryable: 429, 500, 502, 503, 504, rate limits, overloaded
// Not retryable: Context overflow (handled by compaction instead)
```

### Context Overflow Handling

Automatic compaction when approaching token limits:

```typescript
// 1. Detect overflow (pattern matching error messages or usage > context window)
// 2. Find cut point preserving tool call/result pairs
// 3. Generate summary using separate LLM call
// 4. Replace old messages with summary message
// 5. Retry
```

---

## 5. Event Taxonomy

### LLM Stream Events

```typescript
type AssistantMessageEvent =
  | { type: "text_start" | "text_delta" | "text_end" }
  | { type: "thinking_start" | "thinking_delta" | "thinking_end" }
  | { type: "toolcall_start" | "toolcall_delta" | "toolcall_end" }
  | { type: "done"; reason: "stop" | "length" | "toolUse" }
  | { type: "error"; reason: "aborted" | "error" }
```

### Agent Events

```typescript
type AgentEvent =
  | { type: "agent_start" | "agent_end" }
  | { type: "turn_start" | "turn_end" }
  | { type: "message_start" | "message_update" | "message_end" }
  | { type: "tool_execution_start" | "tool_execution_update" | "tool_execution_end" }
```

### Extension Events

```typescript
// Hook points for behavior modification
type ExtensionEvent =
  | "session_start" | "session_switch"
  | "context"           // Modify messages before LLM
  | "before_agent_start" // Inject context
  | "tool_call"         // Block/modify tools
  | "tool_result"       // Transform results
  | "input"             // Transform user input
```

---

## 6. Code You Can Leverage

### For LLM Abstraction (pi-ai)

**Direct reuse:**
- Multi-provider streaming (`packages/ai/src/stream.ts`)
- Message type definitions (`packages/ai/src/types.ts`)
- Cross-provider message transformation (`packages/ai/src/providers/transform-messages.ts`)
- Provider registry pattern (`packages/ai/src/api-registry.ts`)

```typescript
// Example: Use pi-ai directly for any agent
import { stream, getModel } from "@mariozechner/pi-ai";

const model = getModel("anthropic", "claude-sonnet-4-20250514");
for await (const event of stream(model, context)) {
  // Standardized events regardless of provider
}
```

### For Agent Core (pi-agent-core)

**Direct reuse:**
- Agent loop implementation (`packages/agent/src/agent-loop.ts`)
- EventStream class (`packages/agent/src/types.ts`)
- Tool validation (`packages/ai/src/utils/validate-tool-arguments.ts`)

```typescript
// Minimal agent in ~20 lines
import { Agent } from "@mariozechner/pi-agent-core";
import { getModel, Type } from "@mariozechner/pi-ai";

const agent = new Agent({
  initialState: {
    systemPrompt: "You are a helpful assistant.",
    model: getModel("openai", "gpt-4o"),
    tools: [myTool],
    messages: [],
  },
  convertToLlm: (msgs) => msgs.filter(m => m.role !== "custom"),
});

agent.subscribe(console.log);
await agent.prompt("Hello!");
```

### For Extension System

**Study and adapt:**
- Extension types (`packages/coding-agent/src/extensions/types.ts`)
- Runtime implementation (`packages/coding-agent/src/extensions/runtime.ts`)
- Tool wrapping pattern (`packages/coding-agent/src/extensions/wrap-tool-with-extensions.ts`)

### For Context Management

**Study and adapt:**
- Compaction logic (`packages/coding-agent/src/core/compaction.ts`)
- Token estimation (`packages/coding-agent/src/core/context-stats.ts`)
- Session persistence (`packages/coding-agent/src/core/session-manager.ts`)

---

## 7. Detailed Component Analysis

### 7.1 Agent Loop Architecture

The agent loop follows a **nested loop pattern**:

```typescript
async function runLoop(...): Promise<void> {
  // Outer loop: handles follow-up messages after agent would stop
  while (true) {
    let hasMoreToolCalls = true;
    let steeringAfterTools: AgentMessage[] | null = null;

    // Inner loop: processes tool calls and steering messages
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      // 1. Emit turn_start
      // 2. Inject pending messages (steering/follow-up)
      // 3. Stream assistant response
      // 4. Execute tool calls (with steering check between each)
      // 5. Emit turn_end
      // 6. Check for new steering messages
    }

    // Check for follow-up messages when agent would stop
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) {
      pendingMessages = followUpMessages;
      continue;
    }
    break;
  }
}
```

**Key characteristics:**
- **Event-driven**: Every action emits events via EventStream
- **Interruptible**: Steering messages can interrupt between tool executions
- **Continuable**: Follow-up messages can restart the loop after completion

### 7.2 Message Transform Pipeline

```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
```

This pipeline enables:
1. **Context pruning** (via `transformContext`)
2. **Custom message types** (via `convertToLlm` filtering)
3. **Message injection** (external context)

### 7.3 Tool System

**Tool Interface:**

```typescript
interface AgentTool<TParameters extends TSchema = TSchema, TDetails = any> extends Tool<TParameters> {
  label: string;
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ) => Promise<AgentToolResult<TDetails>>;
}

interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];
  details: T;
}
```

**Tool Execution Flow:**

```typescript
async function executeToolCalls(tools, assistantMessage, signal, stream, getSteeringMessages) {
  const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
  const results: ToolResultMessage[] = [];

  for (const toolCall of toolCalls) {
    // 1. Emit tool_execution_start
    // 2. Validate arguments against TypeBox schema
    // 3. Execute tool with streaming updates
    // 4. Emit tool_execution_end
    // 5. Check for steering messages (can interrupt remaining tools)
  }
  
  return { toolResults: results, steeringMessages };
}
```

### 7.4 Cross-Provider Compatibility

The `transformMessages` function handles cross-provider compatibility:

```typescript
function transformMessages<TApi extends Api>(
  messages: Message[],
  model: Model<TApi>,
  normalizeToolCallId?: (id: string, model: Model<TApi>, source: AssistantMessage) => string,
): Message[] {
  // 1. Build tool call ID mapping for normalization
  // 2. For assistant messages:
  //    - Same provider/model: preserve thinking blocks with signatures
  //    - Different provider: convert thinking to plain text, strip signatures
  //    - Normalize tool call IDs if needed (e.g., Mistral's 9-char requirement)
  // 3. Handle orphaned tool calls by inserting synthetic results
  // 4. Skip errored/aborted assistant messages entirely
}
```

Key transformations:
- **Thinking blocks**: Preserved for same model, converted to text for cross-provider
- **Tool call IDs**: Normalized (e.g., OpenAI's 450+ char IDs → Anthropic's 64-char max)
- **Signatures**: Only preserved for same provider/model replay
- **Error handling**: Errored/aborted messages are filtered out

---

## 8. Session and Persistence

### Session File Format (JSONL Tree)

```
{"type":"session","version":3,"id":"...","timestamp":"...","cwd":"..."}
{"type":"message","id":"abc","parentId":null,"timestamp":"...","message":{...}}
{"type":"message","id":"def","parentId":"abc","timestamp":"...","message":{...}}
{"type":"thinking_level_change","id":"...","parentId":"def","thinkingLevel":"medium"}
{"type":"compaction","id":"...","summary":"...","firstKeptEntryId":"...","tokensBefore":50000}
```

**Key features:**
- Tree structure with `id`/`parentId` for branching
- `leafId` pointer tracks current position
- Branching moves leaf without modifying history
- `getMessagesToLeaf()` resolves messages from root to leaf

### Compaction Settings

```typescript
interface CompactionSettings {
  enabled: boolean;
  reserveTokens: number;     // Default: 16384
  keepRecentTokens: number;  // Default: 20000
}

function shouldCompact(contextTokens: number, contextWindow: number, settings: CompactionSettings): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

---

## 9. SDK API Surface

### Minimal Usage

```typescript
import { createAgentSession } from "@mariozechner/pi-coding-agent";

const { session } = await createAgentSession();

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("What files are in the current directory?");
```

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `cwd` | `process.cwd()` | Working directory |
| `agentDir` | `~/.pi` | Global config directory |
| `model` | From settings/first available | Model to use |
| `thinkingLevel` | From settings/"medium" | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `tools` | `[]` | Built-in tools: `read`, `bash`, `edit`, `write` |
| `customTools` | `[]` | Additional tool definitions |
| `resourceLoader` | `DefaultResourceLoader` | For extensions, skills, prompts, themes |
| `sessionManager` | `SessionManager.create(cwd)` | Session persistence |

### Steering API

```typescript
// Interrupt agent with new message (delivered after current tool, skips remaining)
await session.steer("Stop and do this instead");

// Queue follow-up (delivered only when agent finishes)
await session.followUp("When you're done, also do this");

// Abort current operation
await session.abort();

// Clear all queued messages
const { steering, followUp } = session.clearQueue();
```

---

## 10. Recommendations for Your Generic Agent

### Architecture

```
┌──────────────────────────────────────────┐
│         Your Generic Agent App           │
│  (Domain-specific tools, UI, workflows)  │
├──────────────────────────────────────────┤
│         Extension Layer (optional)       │
│  (Guardrails, approvals, transformers)   │
├──────────────────────────────────────────┤
│         pi-agent-core (direct use)       │
│  (Agent loop, tool execution, events)    │
├──────────────────────────────────────────┤
│         pi-ai (direct use)               │
│  (LLM streaming, multi-provider)         │
└──────────────────────────────────────────┘
```

### Key Interfaces to Implement

1. **Tools** - Use `AgentTool` interface with TypeBox schemas
2. **Context Transform** - Inject `transformContext` for domain-specific filtering
3. **Steering** - Implement `getSteeringMessages` for human-in-the-loop
4. **Events** - Subscribe to agent events for UI updates

### Minimal Viable Generic Agent

```typescript
import { Agent, AgentTool } from "@mariozechner/pi-agent-core";
import { getModel, Type, stream } from "@mariozechner/pi-ai";

// 1. Define domain tools
const searchKnowledgeBase: AgentTool = {
  name: "search_kb",
  label: "Search Knowledge Base",
  description: "Search internal documents",
  parameters: Type.Object({ query: Type.String() }),
  execute: async (id, { query }) => ({
    content: [{ type: "text", text: await myKBSearch(query) }],
    details: {},
  }),
};

// 2. Create agent
const agent = new Agent({
  initialState: {
    systemPrompt: "You are a knowledge assistant...",
    model: getModel("anthropic", "claude-sonnet-4-20250514"),
    tools: [searchKnowledgeBase],
    messages: [],
  },
  getApiKey: async (provider) => process.env[`${provider.toUpperCase()}_API_KEY`],
});

// 3. Subscribe to events
agent.subscribe((event) => {
  if (event.type === "message_update") {
    const delta = event.assistantMessageEvent;
    if (delta.type === "text_delta") process.stdout.write(delta.delta);
  }
});

// 4. Run
await agent.prompt("What do we know about customer X?");
```

---

## 11. Summary: What Makes Pi Special

| Aspect | Pi's Approach | Benefit |
|--------|---------------|---------|
| **Core Size** | ~400 LOC agent loop | Easy to understand, audit, modify |
| **Composition** | Injected functions, no inheritance | Swap any behavior without subclassing |
| **Extensibility** | Runtime extensions + declaration merging | Agents can modify themselves |
| **Self-improvement** | Skills + MEMORY.md | Accumulates instructions over time |
| **Provider Abstraction** | Registry with unified events | Switch models mid-conversation |
| **Error Recovery** | Auto-retry + auto-compaction | Robust to transient failures |
| **Events** | Fine-grained lifecycle events | Reactive UIs, behavior injection |

The "magic" is that Pi provides just enough structure (event types, tool interface, agent loop) while leaving all behavior injectable. This lets the agent literally evolve based on how you use it - the "dawn of malleable software" indeed.

---

## 12. Next Steps for Implementation Phase

1. **Install and experiment** with `pi-ai` and `pi-agent-core` directly
2. **Build a minimal generic agent** with your domain tools
3. **Add extension system** for guardrails/approvals
4. **Implement skill-like capability** for self-improvement
5. **Add persistence layer** appropriate for your domain

---

## Appendix A: Key File Locations

| Component | Location |
|-----------|----------|
| Agent Loop | `packages/agent/src/agent-loop.ts` |
| Agent Class | `packages/agent/src/agent.ts` |
| Agent Types | `packages/agent/src/types.ts` |
| LLM Streaming | `packages/ai/src/stream.ts` |
| LLM Types | `packages/ai/src/types.ts` |
| Provider Registry | `packages/ai/src/api-registry.ts` |
| Message Transform | `packages/ai/src/providers/transform-messages.ts` |
| Tool Validation | `packages/ai/src/utils/validate-tool-arguments.ts` |
| Extension Types | `packages/coding-agent/src/extensions/types.ts` |
| Extension Runtime | `packages/coding-agent/src/extensions/runtime.ts` |
| Compaction | `packages/coding-agent/src/core/compaction.ts` |
| Session Manager | `packages/coding-agent/src/core/session-manager.ts` |
| SDK Entry | `packages/coding-agent/src/sdk/index.ts` |

## Appendix B: Package Summary

| Package | npm Name | Purpose |
|---------|----------|---------|
| **ai** | `@mariozechner/pi-ai` | Unified multi-provider LLM API |
| **agent** | `@mariozechner/pi-agent-core` | Stateful agent runtime with tool execution |
| **tui** | `@mariozechner/pi-tui` | Terminal UI framework |
| **coding-agent** | `@mariozechner/pi-coding-agent` | Interactive CLI coding agent |
| **mom** | `@mariozechner/pi-mom` | Slack bot using coding agent |
| **web-ui** | `@mariozechner/pi-web-ui` | Web components for AI chat |
| **pods** | `@mariozechner/pi` | GPU pod management CLI |
