# Systematic Plan: Adding SLM (Small Language Model) Functionality to Codex CLI

## Overview

This plan outlines the systematic approach to integrate task-specific Small Language Models (SLMs) into the Codex CLI, allowing the system to use specialized models for specific tasks while maintaining compatibility with the existing OpenAI-based architecture.

## Detailed Agent-Loop Task Execution Analysis

### Task Types and Execution Flow in agent-loop.ts

The `agent-loop.ts` file orchestrates the entire conversation flow and task execution pipeline. Understanding each component is crucial for integrating SLMs effectively.

#### 1. **Core Task Categories**

**1.1 Shell Command Execution (`shell` function)**

- **Purpose**: Execute arbitrary shell commands in a sandboxed environment
- **Flow**: `handleFunctionCall()` → `handleExecCommand()` → `exec()` with platform-specific sandboxing
- **Security**: Uses Landlock (Linux) or Seatbelt (macOS) sandboxing
- **Input**: Command array, working directory, timeout
- **Output**: stdout, stderr, exit code, execution metadata
- **SLM Integration Point**: Command generation, validation, and explanation tasks

**1.2 Apply Patch Operations (`apply_patch`)**

- **Purpose**: Apply structured code changes using a specialized diff format
- **Flow**: `handleFunctionCall()` → `execApplyPatch()` → file system modifications
- **Format**: V4A diff format with context lines and @@ markers
- **Security**: Controlled file modifications within writable roots
- **SLM Integration Point**: Code generation, refactoring, and bug fixing tasks

**1.3 Local Shell Calls (`local_shell_call`)**

- **Purpose**: Enhanced shell execution for codex models with structured actions
- **Flow**: `handleLocalShellCall()` → normalized to standard exec flow
- **Enhancement**: Structured action types with timeout and working directory
- **SLM Integration Point**: Task-specific command execution with better context

#### 2. **Task Detection and Classification**

**2.1 Natural Language Analysis**

```typescript
// Current detection in detectTaskType()
const userInput = input
  .filter(item => item.type === "message" && item.role === "user")
  .map(item => item.content.map(c => c.type === "input_text" ? c.text : "").join(""))
  .join(" ")
  .toLowerCase();

// Detection patterns:
- "generate|create|write" → code-generation
- "review|analyze|check" → code-review
- "fix|bug|error" → bug-fix
- "refactor|improve|optimize" → refactoring
- "document|comment|explain" → documentation
- "test|unit|spec" → testing
```

**2.2 Context-Aware Task Classification**

- **File Extensions**: .js/.ts (JavaScript/TypeScript), .py (Python), .md (Documentation)
- **Command Patterns**: npm/yarn commands, git operations, build tools
- **Project Structure**: Package.json, Makefile, requirements.txt presence
- **Historical Context**: Previous commands and outputs in conversation

#### 3. **Execution Pipeline Stages**

**3.1 Request Preparation Stage**

```typescript
// In run() method around line 536
const turnInput: Array<ResponseInputItem> = [];
let transcriptPrefixLen = 0;

// Context building for stateless operation
if (this.disableResponseStorage) {
  this.transcript.push(...filterToApiMessages(input));
  turnInput = [...this.transcript, ...abortOutputs].map(stripInternalFields);
} else {
  turnInput = [...abortOutputs, ...input].map(stripInternalFields);
}
```

**3.2 Model Selection and Routing**

```typescript
// Current OpenAI-only flow (to be enhanced with SLM routing)
const shouldUseSLM = this.shouldUseSLM(turnInput);

if (shouldUseSLM && this.slmProvider) {
  // Route to task-specific SLM
  stream = await this.makeSLMRequest(turnInput, tools);
} else {
  // Use general-purpose OpenAI model
  stream = await responseCall({
    /* OpenAI parameters */
  });
}
```

**3.3 Response Processing Stage**

```typescript
// Event-driven response handling
for await (const event of stream) {
  if (event.type === "response.output_item.done") {
    const item = event.item;

    if (item.type === "function_call" || item.type === "local_shell_call") {
      // Track pending tool calls for cancellation
      this.pendingAborts.add(callId);
    } else {
      // Surface completed items to UI
      stageItem(item as ResponseItem);
    }
  }
}
```

#### 4. **Tool Call Processing Flow**

**4.1 Function Call Normalization**

```typescript
// handleFunctionCall() method
// Handles both chat and responses API formats
const isChatStyle = (item as any).function != null;
const name = isChatStyle ? (item as any).function?.name : (item as any).name;
const rawArguments = isChatStyle
  ? (item as any).function?.arguments
  : (item as any).arguments;
const callId = (item as any).call_id ?? (item as any).id;
```

**4.2 Argument Parsing and Validation**

```typescript
const args = parseToolCallArguments(rawArguments ?? "{}");
if (args == null) {
  return [
    {
      type: "function_call_output",
      call_id: item.call_id,
      output: `invalid arguments: ${rawArguments}`,
    },
  ];
}
```

**4.3 Tool Execution Routing**

```typescript
if (name === "container.exec" || name === "shell") {
  const {
    outputText,
    metadata,
    additionalItems: additionalItemsFromExec,
  } = await handleExecCommand(
    args,
    this.config,
    this.approvalPolicy,
    this.additionalWritableRoots,
    this.getCommandConfirmation,
    this.execAbortController?.signal,
  );
  outputItem.output = JSON.stringify({ output: outputText, metadata });
}
```

#### 5. **Security and Approval System**

**5.1 Command Classification**

```typescript
// In handleExecCommand()
const safety = canAutoApprove(command, workdir, policy, [process.cwd()]);

switch (safety.type) {
  case "ask-user": // Requires explicit user approval
  case "auto-approve": // Safe to execute automatically
  case "reject": // Blocked by policy
}
```

**5.2 Sandbox Selection**

```typescript
// Platform-specific sandboxing
async function getSandbox(runInSandbox: boolean): Promise<SandboxType> {
  if (runInSandbox) {
    if (process.platform === "darwin") {
      return SandboxType.MACOS_SEATBELT; // Uses sandbox-exec
    } else if (process.platform === "linux") {
      return SandboxType.LINUX_LANDLOCK; // Uses Landlock LSM
    }
  }
  return SandboxType.NONE;
}
```

**5.3 Execution with Resource Limits**

```typescript
// In raw-exec.ts
const stdoutCollector = createTruncatingCollector(
  child.stdout!,
  maxBytes,
  maxLines,
);
const stderrCollector = createTruncatingCollector(
  child.stderr!,
  maxBytes,
  maxLines,
);
```

#### 6. **Error Handling and Recovery**

**6.1 Network Error Recovery**

```typescript
// Retry logic with exponential backoff
for (let attempt = 1; attempt <= MAX_RETRIES; attempt++) {
  try {
    stream = await responseCall(params);
    break;
  } catch (error) {
    if (isTimeout || isServerError || isConnectionError) {
      if (attempt < MAX_RETRIES) continue;
    }
    throw error;
  }
}
```

**6.2 Rate Limit Handling**

```typescript
if (isRateLimit && attempt < MAX_RETRIES) {
  let delayMs = RATE_LIMIT_RETRY_WAIT_MS * 2 ** (attempt - 1);

  // Parse suggested retry time from error message
  const m = /(?:retry|try) again in ([\d.]+)s/i.exec(errCtx?.message ?? "");
  if (m && m[1]) {
    delayMs = parseFloat(m[1]) * 1000;
  }

  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

**6.3 Cancellation Handling**

```typescript
public cancel(): void {
  this.canceled = true;
  this.currentStream?.controller?.abort?.();
  this.execAbortController?.abort();

  // Create synthetic function_call_output for pending aborts
  if (this.pendingAborts.size === 0) {
    this.onLastResponseId("");
  }
}
```

#### 7. **SLM Integration Points**

**7.1 Task-Specific Model Selection**

- **Code Generation**: Use specialized code generation SLMs for create/write tasks
- **Code Review**: Route analysis/review requests to review-optimized models
- **Bug Fixing**: Use debugging-focused SLMs for error resolution
- **Documentation**: Route explanation/comment requests to documentation SLMs

**7.2 Context Optimization**

- **Focused Instructions**: Task-specific prompts rather than general instructions
- **Reduced Context**: Filter conversation history for relevant task context
- **Tool Selection**: Limit available tools based on task type

**7.3 Output Processing**

- **Format Adaptation**: Convert SLM outputs to standard response format
- **Quality Validation**: Verify SLM outputs meet task requirements
- **Fallback Handling**: Route to general model if SLM fails

### Task-Specific SLM Workflow Examples

#### Code Generation Task Flow

1. **Detection**: User request contains "create", "generate", "write"
2. **Context**: Extract file type, existing code patterns, dependencies
3. **SLM Selection**: Route to code-generation-optimized SLM
4. **Prompt**: Task-specific instructions + minimal context
5. **Output**: Generated code with apply_patch formatting
6. **Validation**: Syntax check and integration verification

#### Bug Fix Task Flow

1. **Detection**: User mentions "fix", "bug", "error", or provides error logs
2. **Context**: Error messages, stack traces, related code sections
3. **SLM Selection**: Route to debugging-focused SLM
4. **Analysis**: Root cause identification and fix suggestions
5. **Output**: Targeted fix with explanation
6. **Testing**: Validation commands to verify fix effectiveness

#### Code Review Task Flow

1. **Detection**: User requests "review", "analyze", "check"
2. **Context**: Code files, commit diffs, review criteria
3. **SLM Selection**: Route to review-specialized SLM
4. **Analysis**: Code quality, security, performance assessment
5. **Output**: Structured feedback with suggestions
6. **Actionability**: Specific improvement recommendations

This detailed understanding of the agent-loop task execution provides the foundation for implementing SLM integration that preserves the existing workflow while adding task-specific optimization capabilities.

## Phase 1: Core Infrastructure (Foundation)

### 1.1 Create SLM Provider Interface

**File: `codex-cli/src/utils/slm/types.ts`**

```typescript
export interface SLMProvider {
  id: string;
  name: string;
  baseURL: string;
  apiKey?: string;
  model: string;
  taskType:
    | "code-generation"
    | "code-review"
    | "bug-fix"
    | "refactoring"
    | "documentation"
    | "testing";
  maxTokens: number;
  temperature: number;
  supportsStreaming: boolean;
  customHeaders?: Record<string, string>;
  wireApi: "chat" | "responses";
  queryParams?: Record<string, string>;
}

export interface SLMResponse {
  content: string;
  toolCalls?: Array<{
    name: string;
    arguments: Record<string, any>;
  }>;
  metadata?: Record<string, any>;
}

export interface SLMRequest {
  model: string;
  messages: Array<{
    role: "system" | "user" | "assistant";
    content: string;
  }>;
  max_tokens?: number;
  temperature?: number;
  tools?: Array<any>;
  tool_choice?: string;
  stream?: boolean;
}
```

### 1.2 Create SLM Client Factory

**File: `codex-cli/src/utils/slm/client-factory.ts`**

```typescript
import OpenAI from "openai";
import { SLMProvider } from "./types.js";

export class SLMClientFactory {
  static createClient(provider: SLMProvider): OpenAI {
    return new OpenAI({
      apiKey: provider.apiKey || "dummy",
      baseURL: provider.baseURL,
      defaultHeaders: {
        ...provider.customHeaders,
        originator: "codex-cli",
        version: "1.0.0",
      },
      timeout: 30000,
    });
  }
}
```

### 1.3 Create SLM Registry

**File: `codex-cli/src/utils/slm/registry.ts`**

```typescript
import { SLMProvider } from "./types.js";

export class SLMRegistry {
  private static providers = new Map<string, SLMProvider>();

  static register(provider: SLMProvider): void {
    this.providers.set(provider.id, provider);
  }

  static get(id: string): SLMProvider | undefined {
    return this.providers.get(id);
  }

  static getAll(): SLMProvider[] {
    return Array.from(this.providers.values());
  }

  static getByTaskType(taskType: string): SLMProvider[] {
    return this.getAll().filter((p) => p.taskType === taskType);
  }
}
```

## Phase 2: Configuration System Updates

### 2.1 Update AppConfig Interface

**File: `codex-cli/src/utils/config.ts`**

```typescript
// Add to existing AppConfig interface
export type AppConfig = {
  // ... existing fields
  slmProvider?: string;
  useSLMForTask?: boolean;
  taskSpecificInstructions?: string;
  slmProviders?: Record<string, SLMProvider>;
  slmTaskMapping?: Record<string, string>; // task -> slm provider id
};
```

### 2.2 Update StoredConfig Interface

**File: `codex-cli/src/utils/config.ts`**

```typescript
// Add to existing StoredConfig interface
export type StoredConfig = {
  // ... existing fields
  slmProvider?: string;
  useSLMForTask?: boolean;
  taskSpecificInstructions?: string;
  slmProviders?: Record<string, SLMProvider>;
  slmTaskMapping?: Record<string, string>;
};
```

### 2.3 Update Configuration Loading

**File: `codex-cli/src/utils/config.ts`**

```typescript
// Add to loadConfig function
export function loadConfig(): AppConfig {
  const config = loadStoredConfig();

  return {
    // ... existing fields
    slmProvider: process.env.CODEX_SLM_PROVIDER || config.slmProvider,
    useSLMForTask:
      process.env.CODEX_USE_SLM === "true" || config.useSLMForTask || false,
    taskSpecificInstructions:
      process.env.CODEX_TASK_INSTRUCTIONS || config.taskSpecificInstructions,
    slmProviders: config.slmProviders || {},
    slmTaskMapping: config.slmTaskMapping || {},
  };
}
```

## Phase 3: AgentLoop Modifications

### 3.1 Update AgentLoopParams

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
type AgentLoopParams = {
  // ... existing fields
  slmProvider?: SLMProvider;
  taskSpecificInstructions?: string;
  useSLMForTask?: boolean;
  slmTaskMapping?: Record<string, string>;
};
```

### 3.2 Add SLM Properties to AgentLoop Class

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
export class AgentLoop {
  // ... existing properties
  private slmProvider?: SLMProvider;
  private taskSpecificInstructions?: string;
  private useSLMForTask: boolean;
  private slmTaskMapping: Record<string, string>;
  private slmClient?: OpenAI;
}
```

### 3.3 Update Constructor

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
constructor({
  // ... existing parameters
  slmProvider,
  taskSpecificInstructions,
  useSLMForTask = false,
  slmTaskMapping = {},
}: AgentLoopParams & { config?: AppConfig }) {
  // ... existing initialization

  this.slmProvider = slmProvider;
  this.taskSpecificInstructions = taskSpecificInstructions;
  this.useSLMForTask = useSLMForTask;
  this.slmTaskMapping = slmTaskMapping;

  // Initialize SLM client if provided
  if (this.slmProvider) {
    this.slmClient = SLMClientFactory.createClient(this.slmProvider);
  }
}
```

### 3.4 Add SLM Request Method

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
private async makeSLMRequest(
  input: Array<ResponseInputItem>,
  tools: Array<Tool>
): Promise<any> {
  if (!this.slmProvider || !this.slmClient) {
    throw new Error("SLM provider not configured");
  }

  const prompt = this.buildSLMPrompt(input);
  const messages = [
    {
      role: "system" as const,
      content: this.taskSpecificInstructions || this.instructions || ""
    },
    {
      role: "user" as const,
      content: prompt
    }
  ];

  const request = {
    model: this.slmProvider.model,
    messages,
    max_tokens: this.slmProvider.maxTokens,
    temperature: this.slmProvider.temperature,
    tools: tools.length > 0 ? tools : undefined,
    tool_choice: tools.length > 0 ? "auto" : undefined,
    stream: this.slmProvider.supportsStreaming,
  };

  if (this.slmProvider.wireApi === 'responses') {
    return this.slmClient.responses.create(request);
  } else {
    return this.slmClient.chat.completions.create(request);
  }
}

private buildSLMPrompt(input: Array<ResponseInputItem>): string {
  return input
    .filter(item => item.type === "message")
    .map(item => {
      if (item.type === "message") {
        return `${item.role}: ${item.content.map(c =>
          c.type === "input_text" ? c.text : ""
        ).join("")}`;
      }
      return "";
    })
    .join("\n");
}
```

### 3.5 Add Task Detection Logic

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
private detectTaskType(input: Array<ResponseInputItem>): string {
  const userInput = input
    .filter(item => item.type === "message" && item.role === "user")
    .map(item => item.content.map(c => c.type === "input_text" ? c.text : "").join(""))
    .join(" ")
    .toLowerCase();

  if (userInput.includes("generate") || userInput.includes("create") || userInput.includes("write")) {
    return "code-generation";
  } else if (userInput.includes("review") || userInput.includes("analyze") || userInput.includes("check")) {
    return "code-review";
  } else if (userInput.includes("fix") || userInput.includes("bug") || userInput.includes("error")) {
    return "bug-fix";
  } else if (userInput.includes("refactor") || userInput.includes("improve") || userInput.includes("optimize")) {
    return "refactoring";
  } else if (userInput.includes("document") || userInput.includes("comment") || userInput.includes("explain")) {
    return "documentation";
  } else if (userInput.includes("test") || userInput.includes("unit") || userInput.includes("spec")) {
    return "testing";
  }

  return "general";
}

private shouldUseSLM(input: Array<ResponseInputItem>): boolean {
  if (!this.useSLMForTask) return false;

  const taskType = this.detectTaskType(input);
  const slmProviderId = this.slmTaskMapping[taskType];

  if (slmProviderId) {
    const provider = SLMRegistry.get(slmProviderId);
    if (provider) {
      this.slmProvider = provider;
      this.slmClient = SLMClientFactory.createClient(provider);
      return true;
    }
  }

  return false;
}
```

### 3.6 Modify Main Run Loop

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
// In the run() method, replace the existing API call section
// Around line 800, replace the existing OpenAI call with:

const shouldUseSLM = this.shouldUseSLM(turnInput);

if (shouldUseSLM && this.slmProvider) {
  // Use SLM for specific task
  stream = await this.makeSLMRequest(turnInput, tools);
} else {
  // Use regular OpenAI flow (existing code)
  const responseCall =
    !this.config.provider ||
    this.config.provider?.toLowerCase() === "openai" ||
    this.config.provider?.toLowerCase() === "azure"
      ? (params: ResponseCreateParams) => this.oai.responses.create(params)
      : (params: ResponseCreateParams) =>
          responsesCreateViaChatCompletions(
            this.oai,
            params as ResponseCreateParams & { stream: true },
          );

  stream = await responseCall({
    model: this.model,
    instructions: mergedInstructions,
    input: turnInput,
    stream: true,
    parallel_tool_calls: false,
    reasoning,
    ...(this.config.flexMode ? { service_tier: "flex" } : {}),
    ...(this.disableResponseStorage
      ? { store: false }
      : {
          store: true,
          previous_response_id: lastResponseId || undefined,
        }),
    tools: tools,
    tool_choice: "auto",
  });
}
```

## Phase 4: CLI Integration

### 4.1 Update CLI Arguments

**File: `codex-cli/src/cli.tsx`**

```typescript
// Add to CLI flags
const cli = meow(
  `
  Usage
    $ codex [input]

  Options
    --model, -m          Model to use
    --provider, -p       Provider to use
    --slm-provider       SLM provider to use for specific tasks
    --use-slm            Enable SLM for task-specific operations
    --task-instructions  Task-specific instructions for SLM
    --slm-config         Path to SLM configuration file
    // ... existing options
  `,
  {
    importMeta: import.meta,
    flags: {
      // ... existing flags
      slmProvider: {
        type: "string",
        alias: "s",
      },
      useSlm: {
        type: "boolean",
        default: false,
      },
      taskInstructions: {
        type: "string",
      },
      slmConfig: {
        type: "string",
      },
    },
  },
);
```

### 4.2 Update TerminalChat Component

**File: `codex-cli/src/components/chat/terminal-chat.tsx`**

```typescript
// Update AgentLoop instantiation
agentRef.current = new AgentLoop({
  model,
  provider,
  config,
  instructions: config.instructions,
  approvalPolicy,
  disableResponseStorage: config.disableResponseStorage,
  additionalWritableRoots,
  onLastResponseId: setLastResponseId,
  onItem: (item) => {
    // ... existing onItem logic
  },
  onLoading: setLoading,
  getCommandConfirmation: async (command, applyPatch) => {
    // ... existing confirmation logic
  },

  // Add SLM parameters
  slmProvider: config.slmProvider
    ? SLMRegistry.get(config.slmProvider)
    : undefined,
  taskSpecificInstructions: config.taskSpecificInstructions,
  useSLMForTask: config.useSLMForTask || false,
  slmTaskMapping: config.slmTaskMapping || {},
});
```

## Phase 5: Configuration Files

### 5.1 Create Default SLM Configuration

**File: `codex-cli/src/utils/slm/default-config.ts`**

```typescript
import { SLMProvider } from "./types.js";

export const DEFAULT_SLM_PROVIDERS: Record<string, SLMProvider> = {
  "codegen-slm": {
    id: "codegen-slm",
    name: "Code Generation SLM",
    baseURL: "https://your-codegen-slm.com/v1",
    model: "codegen-model",
    taskType: "code-generation",
    maxTokens: 2048,
    temperature: 0.1,
    supportsStreaming: false,
    wireApi: "chat",
  },
  "review-slm": {
    id: "review-slm",
    name: "Code Review SLM",
    baseURL: "https://your-review-slm.com/v1",
    model: "review-model",
    taskType: "code-review",
    maxTokens: 1024,
    temperature: 0.3,
    supportsStreaming: true,
    wireApi: "chat",
  },
  "bugfix-slm": {
    id: "bugfix-slm",
    name: "Bug Fix SLM",
    baseURL: "https://your-bugfix-slm.com/v1",
    model: "bugfix-model",
    taskType: "bug-fix",
    maxTokens: 1536,
    temperature: 0.2,
    supportsStreaming: false,
    wireApi: "responses",
  },
};

export const DEFAULT_TASK_MAPPING: Record<string, string> = {
  "code-generation": "codegen-slm",
  "code-review": "review-slm",
  "bug-fix": "bugfix-slm",
  "refactoring": "codegen-slm",
  "documentation": "codegen-slm",
  "testing": "codegen-slm",
};
```

### 5.2 Update Configuration Loading

**File: `codex-cli/src/utils/config.ts`**

```typescript
import {
  DEFAULT_SLM_PROVIDERS,
  DEFAULT_TASK_MAPPING,
} from "./slm/default-config.js";
import { SLMRegistry } from "./slm/registry.js";

// In loadConfig function, add:
export function loadConfig(): AppConfig {
  const config = loadStoredConfig();

  // Register default SLM providers
  Object.values(DEFAULT_SLM_PROVIDERS).forEach((provider) => {
    SLMRegistry.register(provider);
  });

  // Register custom SLM providers from config
  if (config.slmProviders) {
    Object.values(config.slmProviders).forEach((provider) => {
      SLMRegistry.register(provider);
    });
  }

  return {
    // ... existing fields
    slmProvider: process.env.CODEX_SLM_PROVIDER || config.slmProvider,
    useSLMForTask:
      process.env.CODEX_USE_SLM === "true" || config.useSLMForTask || false,
    taskSpecificInstructions:
      process.env.CODEX_TASK_INSTRUCTIONS || config.taskSpecificInstructions,
    slmProviders: { ...DEFAULT_SLM_PROVIDERS, ...config.slmProviders },
    slmTaskMapping: { ...DEFAULT_TASK_MAPPING, ...config.slmTaskMapping },
  };
}
```

## Phase 6: Testing Infrastructure

### 6.1 Create SLM Test Utilities

**File: `codex-cli/tests/slm-test-utils.ts`**

```typescript
import { SLMProvider } from "../src/utils/slm/types.js";

export const createMockSLMProvider = (
  overrides: Partial<SLMProvider> = {},
): SLMProvider => ({
  id: "test-slm",
  name: "Test SLM",
  baseURL: "https://test-slm.com/v1",
  model: "test-model",
  taskType: "code-generation",
  maxTokens: 1024,
  temperature: 0.1,
  supportsStreaming: false,
  wireApi: "chat",
  ...overrides,
});

export const createMockSLMResponse = (content: string) => ({
  choices: [
    {
      message: {
        content,
        role: "assistant",
      },
    },
  ],
});
```

### 6.2 Create SLM Integration Tests

**File: `codex-cli/tests/agent-slm-integration.test.ts`**

```typescript
import { AgentLoop } from "../src/utils/agent/agent-loop.js";
import { createMockSLMProvider } from "./slm-test-utils.js";

describe("AgentLoop SLM Integration", () => {
  it("uses SLM for code generation tasks", async () => {
    const mockSLMProvider = createMockSLMProvider({
      taskType: "code-generation",
    });

    const agent = new AgentLoop({
      model: "test-model",
      instructions: "Test instructions",
      approvalPolicy: { mode: "auto" } as any,
      additionalWritableRoots: [],
      onItem: () => {},
      onLoading: () => {},
      getCommandConfirmation: async () => ({ review: "yes" }) as any,
      onLastResponseId: () => {},
      slmProvider: mockSLMProvider,
      useSLMForTask: true,
      slmTaskMapping: { "code-generation": "test-slm" },
    });

    // Test implementation
  });
});
```

## Phase 7: Documentation and Examples

### 7.1 Update README

**File: `README.md`**

````markdown
## SLM (Small Language Model) Support

Codex CLI now supports task-specific Small Language Models for improved performance and cost efficiency.

### Configuration

Add SLM providers to your `~/.codex/config.json`:

```json
{
  "useSLMForTask": true,
  "slmProviders": {
    "codegen-slm": {
      "id": "codegen-slm",
      "name": "Code Generation SLM",
      "baseURL": "https://your-slm-endpoint.com/v1",
      "model": "codegen-model",
      "taskType": "code-generation",
      "maxTokens": 2048,
      "temperature": 0.1,
      "supportsStreaming": false,
      "wireApi": "chat"
    }
  },
  "slmTaskMapping": {
    "code-generation": "codegen-slm",
    "code-review": "review-slm",
    "bug-fix": "bugfix-slm"
  }
}
```
````

### Environment Variables

- `CODEX_USE_SLM=true` - Enable SLM functionality
- `CODEX_SLM_PROVIDER=provider-id` - Set default SLM provider
- `CODEX_TASK_INSTRUCTIONS="Custom task instructions"`

### Usage

```bash
# Enable SLM for all tasks
codex --use-slm "Generate a React component"

# Use specific SLM provider
codex --slm-provider codegen-slm "Write a Python function"
```

````

### 7.2 Create SLM Configuration Guide
**File: `docs/slm-configuration.md`**
```markdown
# SLM Configuration Guide

## Overview
This guide explains how to configure and use Small Language Models (SLMs) with Codex CLI.

## Provider Configuration
...

## Task Mapping
...

## Best Practices
...
````

## Phase 8: Error Handling and Fallbacks

### 8.1 Add SLM Error Handling

**File: `codex-cli/src/utils/agent/agent-loop.ts`**

```typescript
private async handleSLMError(error: any, fallbackToMainModel: boolean = true): Promise<void> {
  log(`SLM request failed: ${error.message}`);

  if (fallbackToMainModel) {
    log('Falling back to main model');
    this.useSLMForTask = false;
    // Retry with main model
    return this.run(this.lastInput, this.lastResponseId);
  }

  throw error;
}
```

### 8.2 Add SLM Health Checks

**File: `codex-cli/src/utils/slm/health-check.ts`**

```typescript
export class SLMHealthChecker {
  static async checkProvider(provider: SLMProvider): Promise<boolean> {
    try {
      const client = SLMClientFactory.createClient(provider);
      await client.models.list();
      return true;
    } catch (error) {
      log(`SLM provider ${provider.id} health check failed: ${error.message}`);
      return false;
    }
  }
}
```

## Implementation Order

1. **Phase 1**: Core Infrastructure (types, client factory, registry)
2. **Phase 2**: Configuration System Updates
3. **Phase 3**: AgentLoop Modifications (core logic)
4. **Phase 4**: CLI Integration
5. **Phase 5**: Configuration Files
6. **Phase 6**: Testing Infrastructure
7. **Phase 7**: Documentation
8. **Phase 8**: Error Handling and Fallbacks

## Testing Strategy

1. **Unit Tests**: Test each component in isolation
2. **Integration Tests**: Test SLM integration with AgentLoop
3. **End-to-End Tests**: Test complete SLM workflow
4. **Fallback Tests**: Test fallback to main model when SLM fails
5. **Performance Tests**: Compare SLM vs main model performance

## Migration Strategy

1. **Backward Compatibility**: All changes maintain existing functionality
2. **Feature Flags**: SLM functionality is opt-in via configuration
3. **Gradual Rollout**: Users can enable SLM features incrementally
4. **Fallback Mechanisms**: Automatic fallback to main model on SLM failure

## Monitoring and Observability

1. **Logging**: Comprehensive logging for SLM operations
2. **Metrics**: Track SLM usage, performance, and error rates
3. **Health Checks**: Monitor SLM provider availability
4. **User Feedback**: Collect feedback on SLM performance vs main model

This plan provides a modular, extensible approach to integrating SLM functionality while maintaining the existing architecture and ensuring backward compatibility.
