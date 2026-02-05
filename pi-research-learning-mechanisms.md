# Pi Learning & Feedback Mechanisms - Deep Research

## Executive Summary

Pi implements "implicit RL" through three interconnected systems:
1. **Skills (SKILL.md)** - Agent-created capability packages loaded on-demand
2. **Memory (MEMORY.md)** - Persistent preference/instruction accumulation
3. **Sessions (JSONL tree)** - Branching conversation history with compaction

This is not traditional RL with reward signals. Instead, Pi accumulates behavioral modifications through:
- User corrections → stored in MEMORY.md
- Repeated patterns → extracted into Skills
- Long conversations → summarized via compaction, preserving decisions

---

## 1. Skills System (Self-Improvement Mechanism)

### 1.1 What Skills Are

Skills are self-contained capability packages the agent loads on-demand. A skill provides specialized workflows, setup instructions, helper scripts, and reference documentation.

**Key insight**: Pi can create skills for itself. If you ask "create a skill for managing my todo list," it writes a SKILL.md file that it will read and follow in future sessions.

### 1.2 SKILL.md Format

```markdown
---
name: my-skill
description: What this skill does and when to use it. Be specific.
---

# My Skill

## Setup
Run once before first use:
\`\`\`bash
cd /path/to/skill && npm install
\`\`\`

## Usage
\`\`\`bash
./scripts/process.sh <input>
\`\`\`
```

**Frontmatter fields** (per [Agent Skills spec](https://agentskills.io/specification)):

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Max 64 chars. Lowercase a-z, 0-9, hyphens. Must match parent directory. |
| `description` | Yes | Max 1024 chars. What the skill does and when to use it. |
| `license` | No | License name or reference |
| `compatibility` | No | Max 500 chars. Environment requirements. |
| `metadata` | No | Arbitrary key-value mapping. |
| `allowed-tools` | No | Space-delimited list of pre-approved tools (experimental). |
| `disable-model-invocation` | No | When `true`, skill is hidden from system prompt (user must use `/skill:name`). |

### 1.3 Skill Loading Pipeline

**Source:** [packages/coding-agent/src/core/skills.ts](packages/coding-agent/src/core/skills.ts)

```typescript
// Loading locations (in priority order):
// 1. Global: ~/.pi/agent/skills/
// 2. Project: .pi/skills/
// 3. Packages: skills/ directories or pi.skills in package.json
// 4. Settings: skills array with files or directories
// 5. CLI: --skill <path>

export function loadSkills(options: LoadSkillsOptions = {}): LoadSkillsResult {
  const skillMap = new Map<string, Skill>();
  // ... loads from all sources, later sources override earlier on name collision
}
```

**Discovery rules:**
- Direct `.md` files in skills directory root
- Recursive `SKILL.md` files under subdirectories
- Symlinks are followed (realpath deduplication)

### 1.4 Progressive Disclosure

Skills are injected into the system prompt in XML format:

```typescript
export function formatSkillsForPrompt(skills: Skill[]): string {
  const visibleSkills = skills.filter((s) => !s.disableModelInvocation);
  // Only names and descriptions go in system prompt
  // Full content loaded on-demand via read tool
}
```

**System prompt injection:**
```xml
<available_skills>
  <skill>
    <name>my-skill</name>
    <description>What this skill does</description>
    <location>/path/to/SKILL.md</location>
  </skill>
</available_skills>
```

**On-demand loading:** When the task matches a skill's description, the agent uses the `read` tool to load the full SKILL.md. This is progressive disclosure - only descriptions are always in context.

### 1.5 Skill Commands

Skills register as `/skill:name` commands:
```bash
/skill:brave-search           # Load and execute the skill
/skill:pdf-tools extract      # Load skill with arguments
```

---

## 2. MEMORY.md (Persistent Preferences)

### 2.1 Two-Level Memory Hierarchy

**Source:** [packages/mom/src/agent.ts](packages/mom/src/agent.ts#L73-L99)

```typescript
function getMemory(channelDir: string): string {
  const parts: string[] = [];

  // 1. Global workspace memory (shared across all channels)
  const workspaceMemoryPath = join(channelDir, "..", "MEMORY.md");
  if (existsSync(workspaceMemoryPath)) {
    parts.push(`### Global Workspace Memory\n${content}`);
  }

  // 2. Channel-specific memory
  const channelMemoryPath = join(channelDir, "MEMORY.md");
  if (existsSync(channelMemoryPath)) {
    parts.push(`### Channel-Specific Memory\n${content}`);
  }
}
```

**Storage locations:**
- `workspace/MEMORY.md` - Global (shared across channels): project architecture, coding conventions, communication preferences
- `workspace/<channel>/MEMORY.md` - Channel-specific: context, decisions, ongoing work

### 2.2 Memory Content Examples

Typical MEMORY.md content:
- Email writing tone preferences
- Coding conventions (tabs vs spaces, naming)
- Team member responsibilities
- Common troubleshooting steps
- Workflow patterns

### 2.3 Implicit RL Mechanism

This is the core "implicit RL" pattern:
1. User corrects agent: "remember that we use tabs not spaces"
2. Agent writes to MEMORY.md
3. Future sessions read MEMORY.md before responding
4. Behavior is modified without explicit reward signals

**System prompt instruction:**
```
Write to MEMORY.md files to persist context across conversations.
- Global (${workspacePath}/MEMORY.md): skills, preferences, project info
- Channel (${channelPath}/MEMORY.md): channel-specific decisions, ongoing work
Update when you learn something important or when asked to remember something.
```

---

## 3. Session Persistence (JSONL Tree Format)

### 3.1 File Format

**Source:** [packages/coding-agent/src/core/session-manager.ts](packages/coding-agent/src/core/session-manager.ts)

Sessions are stored as append-only JSONL files with tree structure:

```typescript
// Header (first line)
interface SessionHeader {
  type: "session";
  version: number;  // Current: 3
  id: string;
  timestamp: string;
  cwd: string;
  parentSession?: string;  // For forked sessions
}

// Entry types (subsequent lines)
type SessionEntry =
  | SessionMessageEntry      // User/assistant messages
  | ThinkingLevelChangeEntry // Model thinking level changes
  | ModelChangeEntry         // Model switches
  | CompactionEntry          // Summarization markers
  | BranchSummaryEntry       // Context when branching
  | CustomEntry              // Extension data (not in LLM context)
  | CustomMessageEntry       // Extension messages (in LLM context)
  | LabelEntry               // User-defined bookmarks
  | SessionInfoEntry;        // Session metadata
```

### 3.2 Tree Structure

Each entry has `id` and `parentId` forming a tree:

```typescript
interface SessionEntryBase {
  type: string;
  id: string;           // Unique 8-char hex
  parentId: string | null;  // Parent entry ID
  timestamp: string;
}
```

**Branching mechanism:**
```typescript
// Move leaf pointer to earlier entry - next append creates new branch
branch(branchFromId: string): void {
  this.leafId = branchFromId;
}

// Append creates child of current leaf
appendMessage(message: Message): string {
  const entry = {
    id: generateId(this.byId),
    parentId: this.leafId,  // Child of current leaf
    ...
  };
  this.leafId = entry.id;  // Advance leaf
}
```

### 3.3 Context Building from Tree

```typescript
export function buildSessionContext(
  entries: SessionEntry[],
  leafId?: string | null,
): SessionContext {
  // Walk from leaf to root, collecting path
  const path: SessionEntry[] = [];
  let current = leaf;
  while (current) {
    path.unshift(current);
    current = current.parentId ? byId.get(current.parentId) : undefined;
  }

  // Handle compaction summaries
  if (compaction) {
    // 1. Emit summary first
    messages.push(createCompactionSummaryMessage(compaction.summary, ...));
    // 2. Emit kept messages (from firstKeptEntryId)
    // 3. Emit messages after compaction
  }
}
```

---

## 4. Compaction Algorithm

### 4.1 Settings

**Source:** [packages/coding-agent/src/core/compaction/compaction.ts](packages/coding-agent/src/core/compaction/compaction.ts)

```typescript
export interface CompactionSettings {
  enabled: boolean;
  reserveTokens: number;      // Default: 16384 (for prompt + response)
  keepRecentTokens: number;   // Default: 20000 (recent context to preserve)
}
```

### 4.2 Trigger Condition

```typescript
export function shouldCompact(
  contextTokens: number,
  contextWindow: number,
  settings: CompactionSettings
): boolean {
  if (!settings.enabled) return false;
  return contextTokens > contextWindow - settings.reserveTokens;
}
```

### 4.3 Cut Point Detection

```typescript
export function findCutPoint(
  entries: SessionEntry[],
  startIndex: number,
  endIndex: number,
  keepRecentTokens: number,
): CutPointResult {
  // Walk backwards from newest, accumulating estimated message sizes
  // Stop when we've accumulated >= keepRecentTokens
  // Can cut at user OR assistant messages (never tool results)
}
```

**Valid cut points:** user, assistant, custom, bashExecution, branchSummary, compactionSummary
**Never cut at:** toolResult (must follow their tool call)

### 4.4 Token Estimation

```typescript
export function estimateTokens(message: AgentMessage): number {
  // Uses chars/4 heuristic (conservative, overestimates)
  let chars = 0;
  // ... extract text from message content
  return Math.ceil(chars / 4);
}
```

### 4.5 Summarization Prompts

**Initial summary prompt:**
```
Create a structured context checkpoint summary:

## Goal
[What is the user trying to accomplish?]

## Constraints & Preferences
- [Any constraints mentioned by user]

## Progress
### Done
- [x] [Completed tasks]
### In Progress
- [ ] [Current work]
### Blocked
- [Issues preventing progress]

## Key Decisions
- **[Decision]**: [Brief rationale]

## Next Steps
1. [Ordered list]

## Critical Context
- [Data, examples, references needed to continue]
```

**Update summary prompt (iterative compaction):**
```
Update the existing structured summary with new information. RULES:
- PRESERVE all existing information
- ADD new progress, decisions, context
- UPDATE Progress: move items from "In Progress" to "Done"
- UPDATE "Next Steps" based on what was accomplished
- PRESERVE exact file paths, function names, error messages
```

### 4.6 File Operation Tracking

Compaction tracks which files were read/modified for context preservation:

```typescript
export interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}

// Extracted from tool calls in messages
function extractFileOpsFromMessage(message: AgentMessage, fileOps: FileOperations): void {
  // Look for read, write, edit tool calls and extract paths
}
```

Appended to summary as XML:
```xml
<read-files>
/path/to/file1.ts
/path/to/file2.ts
</read-files>

<modified-files>
/path/to/modified.ts
</modified-files>
```

### 4.7 Split Turn Handling

When cutting in the middle of a turn (not at a user message):

```typescript
interface CutPointResult {
  firstKeptEntryIndex: number;
  turnStartIndex: number;      // User message that started the turn
  isSplitTurn: boolean;        // True if cut is mid-turn
}
```

Generates additional "turn prefix summary" to preserve context from the split:
```
This is the PREFIX of a turn that was too large to keep.
The SUFFIX (recent work) is retained.

## Original Request
[What did the user ask for in this turn?]

## Early Progress
- [Key decisions and work done in the prefix]

## Context for Suffix
- [Information needed to understand the retained recent work]
```

---

## 5. Branch Summarization

**Source:** [packages/coding-agent/src/core/compaction/branch-summarization.ts](packages/coding-agent/src/core/compaction/branch-summarization.ts)

When navigating to a different point in the session tree, generates a summary of the abandoned branch so context isn't lost:

```typescript
export function collectEntriesForBranchSummary(
  session: ReadonlySessionManager,
  oldLeafId: string | null,
  targetId: string,
): CollectEntriesResult {
  // Walks from oldLeafId back to common ancestor with targetId
  // Collects entries for summarization
}
```

---

## 6. Patterns for Generic Agents

### 6.1 Storing User Preferences/Feedback

**Pattern: Hierarchical Markdown Files**
```
workspace/
├── MEMORY.md           # Global preferences
└── <context>/
    └── MEMORY.md       # Context-specific preferences
```

Implementation:
1. Read all MEMORY.md files at session start
2. Inject into system prompt
3. When user gives correction, append to appropriate file
4. Future sessions automatically inherit

**Code pattern:**
```typescript
function getMemory(contextDir: string): string {
  const globalPath = join(contextDir, "..", "MEMORY.md");
  const localPath = join(contextDir, "MEMORY.md");
  // Read both, format as sections
}
```

### 6.2 Creating New Capabilities from Usage Patterns

**Pattern: Self-Created Skills**
1. Agent observes repeated workflow
2. User asks to "create a skill for X"
3. Agent writes SKILL.md with instructions + helper scripts
4. Future sessions see skill in system prompt
5. Agent loads full instructions on-demand

**Code pattern:**
```typescript
// Discovery
const skills = loadSkillsFromDir({ dir: skillsPath, source: "project" });

// Injection (just metadata)
const prompt = formatSkillsForPrompt(skills);

// On-demand loading (agent uses read tool)
// Agent: read("/path/to/SKILL.md")
```

### 6.3 Context Management for Long Sessions

**Pattern: Structured Summarization with Checkpoints**

1. **Monitor context size:**
```typescript
const usage = estimateContextTokens(messages);
if (shouldCompact(usage.tokens, model.contextWindow, settings)) {
  // Trigger compaction
}
```

2. **Find optimal cut point:**
- Keep recent `N` tokens
- Never cut at tool results
- Prefer cutting at turn boundaries

3. **Generate structured summary:**
- Goal + Progress tracking
- Key decisions preserved
- File operations tracked
- Previous summary updated (not replaced)

4. **Store as special entry:**
```typescript
interface CompactionEntry {
  type: "compaction";
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  details: { readFiles, modifiedFiles };
}
```

5. **Reconstruct context:**
```typescript
// Summary → kept messages → new messages
messages.push(createCompactionSummaryMessage(...));
for (let i = firstKeptIndex; i < path.length; i++) {
  messages.push(getMessageFromEntry(path[i]));
}
```

### 6.4 The "Implicit RL" Pattern

Pi's self-improvement is not traditional RL. It's **instruction accumulation**:

| Mechanism | Storage | Trigger | Effect |
|-----------|---------|---------|--------|
| User correction | MEMORY.md | User says "remember X" | Future sessions see instruction |
| Workflow pattern | SKILL.md | User says "create skill for X" | New capability available |
| Session history | JSONL | Automatic | Context preserved across sessions |
| Compaction | Summary entry | Context overflow | Key decisions preserved |

**The feedback loop:**
1. Agent makes mistake
2. User corrects: "No, always use X instead of Y"
3. Agent writes to MEMORY.md: "Always use X instead of Y"
4. Next session: Agent reads MEMORY.md, follows instruction
5. Behavior is modified without explicit reward

This is "implicit RL" because:
- No explicit reward signal
- No gradient updates
- No model fine-tuning
- Just persistent natural language instructions that modify behavior

---

## 7. Key Implementation Details

### 7.1 Session Migration

Sessions are versioned. Migrations run on load:
```typescript
function migrateToCurrentVersion(entries: FileEntry[]): boolean {
  if (version < 2) migrateV1ToV2(entries);  // Add id/parentId
  if (version < 3) migrateV2ToV3(entries);  // Rename hookMessage to custom
}
```

### 7.2 Append-Only Semantics

Sessions are append-only for safety:
- Entries cannot be modified or deleted
- Branching changes the leaf pointer, doesn't modify history
- Compaction adds a summary entry, doesn't remove old entries
- Tree traversal skips summarized entries

### 7.3 Label System

Users can add bookmarks:
```typescript
interface LabelEntry {
  type: "label";
  targetId: string;
  label: string | undefined;  // undefined to clear
}
```

### 7.4 Custom Entries for Extensions

Extensions can store data without affecting LLM context:
```typescript
// Not in LLM context
interface CustomEntry<T> {
  type: "custom";
  customType: string;  // Extension identifier
  data?: T;
}

// In LLM context
interface CustomMessageEntry<T> {
  type: "custom_message";
  customType: string;
  content: string | (TextContent | ImageContent)[];
  display: boolean;  // TUI visibility
  details?: T;       // Not sent to LLM
}
```

---

## 8. Summary Table

| Feature | Implementation | Location |
|---------|---------------|----------|
| Skills loading | `loadSkills()` | [skills.ts](packages/coding-agent/src/core/skills.ts) |
| Skill prompt format | `formatSkillsForPrompt()` | [skills.ts](packages/coding-agent/src/core/skills.ts#L247) |
| Memory loading | `getMemory()` | [agent.ts](packages/mom/src/agent.ts#L73) |
| Session tree | `SessionManager` | [session-manager.ts](packages/coding-agent/src/core/session-manager.ts) |
| Context building | `buildSessionContext()` | [session-manager.ts](packages/coding-agent/src/core/session-manager.ts#L308) |
| Compaction trigger | `shouldCompact()` | [compaction.ts](packages/coding-agent/src/core/compaction/compaction.ts#L186) |
| Cut point finding | `findCutPoint()` | [compaction.ts](packages/coding-agent/src/core/compaction/compaction.ts#L355) |
| Summary generation | `generateSummary()` | [compaction.ts](packages/coding-agent/src/core/compaction/compaction.ts#L449) |
| File tracking | `extractFileOpsFromMessage()` | [utils.ts](packages/coding-agent/src/core/compaction/utils.ts#L28) |
| Branch summarization | `collectEntriesForBranchSummary()` | [branch-summarization.ts](packages/coding-agent/src/core/compaction/branch-summarization.ts#L85) |

---

## 9. Applying to a Generic Agent

To implement Pi's learning mechanisms in a generic agent:

### Minimal Implementation

1. **Memory file handling:**
```typescript
// At session start
const globalMemory = readFileIfExists("~/.agent/MEMORY.md");
const projectMemory = readFileIfExists(".agent/MEMORY.md");
systemPrompt += `\n\nMemory:\n${globalMemory}\n${projectMemory}`;

// Instruct agent to write when asked to remember
// "When user says 'remember X', append to MEMORY.md"
```

2. **Skill loading:**
```typescript
interface Skill {
  name: string;
  description: string;
  filePath: string;
}

// Scan directories, extract frontmatter
const skills = loadSkillsFromDir("~/.agent/skills");

// Inject into system prompt (just metadata)
systemPrompt += formatSkillsAsXML(skills);

// Agent reads full file when needed via read tool
```

3. **Session persistence:**
```typescript
// Append-only JSONL with tree structure
interface Entry { id: string; parentId: string | null; ... }

// On message: append entry
appendEntry({ ...message, id: generateId(), parentId: currentLeafId });

// On branch: change leaf pointer
currentLeafId = targetEntryId;
```

4. **Compaction:**
```typescript
// Monitor token usage
if (contextTokens > contextWindow - reserveTokens) {
  // Find cut point (keep ~20k recent tokens)
  // Generate summary of older messages
  // Store as CompactionEntry
  // Rebuild context: summary + kept + new
}
```

### Key Takeaways

1. **Memory is just text files** - No database, no vector store, just markdown
2. **Skills are progressive disclosure** - Metadata in prompt, full content on-demand
3. **Sessions are append-only trees** - Never modify, only append and branch
4. **Compaction preserves decisions** - Structured summary with goal/progress/decisions
5. **"RL" is instruction accumulation** - User feedback becomes persistent instructions
