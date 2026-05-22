# Chat Data Bot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local TypeScript chat data bot that demonstrates provider configuration, RAGFlow MCP tool discovery, bounded agent loops, collapsible traces, and Z.AI Coding Plan compatibility probes.

**Architecture:** Use a Vite React frontend and a Node/Express TypeScript backend in one repo. The backend owns JSON persistence, provider profiles, MCP client behavior, agent orchestration, and trace recording; the frontend stays thin and calls local APIs.

**Tech Stack:** Node.js 22+, TypeScript, Vite, React, Express, Vitest, Zod, local JSON files, OpenAI-compatible HTTP calls, HTTP/SSE MCP endpoint probing.

---

## File Structure

Create these top-level files:

- `package.json`: workspace scripts and shared dev dependencies.
- `tsconfig.base.json`: shared strict TypeScript settings.
- `.gitignore`: ignore dependencies, build output, env files, and local runtime data.
- `.env.example`: documented local environment keys.

Create backend files:

- `backend/package.json`: backend package scripts.
- `backend/tsconfig.json`: backend TypeScript config.
- `backend/vitest.config.ts`: backend test config.
- `backend/src/server.ts`: Express app entrypoint.
- `backend/src/config/paths.ts`: canonical data paths.
- `backend/src/store/json-store.ts`: atomic JSON read/write helpers.
- `backend/src/providers/types.ts`: provider and model types.
- `backend/src/providers/profiles.ts`: SiliconFlow and Z.AI Coding Plan profiles.
- `backend/src/providers/model-discovery.ts`: model list refresh and normalization.
- `backend/src/providers/zai-media.ts`: disabled-by-default image and audio probe request builders.
- `backend/src/mcp/types.ts`: MCP tool and connection types.
- `backend/src/mcp/http-sse-client.ts`: minimal HTTP/SSE MCP tool discovery and call wrapper.
- `backend/src/tools/tool-registry.ts`: internal registry for MCP and local actions.
- `backend/src/agent/types.ts`: agent loop, action, trace, and message types.
- `backend/src/agent/clarification-policy.ts`: low-interruption ASK_USER policy.
- `backend/src/agent/agent-loop.ts`: bounded planner-observer-evaluator loop.
- `backend/src/routes/config-routes.ts`: provider/model/MCP config APIs.
- `backend/src/routes/chat-routes.ts`: session and chat APIs.
- `backend/test/*.test.ts`: focused backend tests.

Create frontend files:

- `frontend/package.json`: frontend scripts.
- `frontend/tsconfig.json`: frontend TypeScript config.
- `frontend/vite.config.ts`: Vite config with API proxy.
- `frontend/index.html`: root HTML.
- `frontend/src/main.tsx`: React entrypoint.
- `frontend/src/App.tsx`: tabbed local app shell.
- `frontend/src/api/client.ts`: typed fetch helpers.
- `frontend/src/components/ChatView.tsx`: chat UI.
- `frontend/src/components/TracePanel.tsx`: collapsible trace UI.
- `frontend/src/components/ModelConfigView.tsx`: model/provider config UI.
- `frontend/src/components/McpConfigView.tsx`: RAGFlow MCP config UI.
- `frontend/src/styles.css`: practical local UI styling.

---

### Task 1: Workspace Scaffold

**Files:**
- Create: `package.json`
- Create: `tsconfig.base.json`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `backend/package.json`
- Create: `backend/tsconfig.json`
- Create: `backend/vitest.config.ts`
- Create: `frontend/package.json`
- Create: `frontend/tsconfig.json`
- Create: `frontend/vite.config.ts`
- Create: `frontend/index.html`

- [ ] **Step 1: Create root workspace files**

```json
{
  "name": "chat-data-bot-learning",
  "private": true,
  "type": "module",
  "workspaces": ["backend", "frontend"],
  "scripts": {
    "dev": "concurrently \"npm run dev -w backend\" \"npm run dev -w frontend\"",
    "test": "npm run test -w backend",
    "build": "npm run build -w backend && npm run build -w frontend",
    "typecheck": "npm run typecheck -w backend && npm run typecheck -w frontend"
  },
  "devDependencies": {
    "@types/node": "^22.15.0",
    "concurrently": "^9.1.2",
    "typescript": "^5.8.3"
  }
}
```

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

```gitignore
node_modules/
dist/
.env
data/
*.log
.DS_Store
```

```bash
ZHIPU_API_KEY=
SILICONFLOW_API_KEY=
RAGFLOW_MCP_ENDPOINT=http://localhost:9382/sse
```

- [ ] **Step 2: Create backend package files**

```json
{
  "name": "chat-data-bot-backend",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "express": "^5.1.0",
    "zod": "^3.25.76"
  },
  "devDependencies": {
    "@types/cors": "^2.8.17",
    "@types/express": "^5.0.1",
    "tsx": "^4.19.4",
    "vitest": "^3.1.4"
  }
}
```

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "types": ["node"]
  },
  "include": ["src", "test"]
}
```

```ts
import { defineConfig } from "vitest/config"

export default defineConfig({
  test: {
    environment: "node",
    include: ["test/**/*.test.ts"]
  }
})
```

- [ ] **Step 3: Create frontend package files**

```json
{
  "name": "chat-data-bot-frontend",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host 127.0.0.1 --port 5173",
    "build": "tsc -p tsconfig.json && vite build",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@vitejs/plugin-react": "^4.4.1",
    "vite": "^6.3.5",
    "react": "^19.1.0",
    "react-dom": "^19.1.0"
  },
  "devDependencies": {
    "@types/react": "^19.1.3",
    "@types/react-dom": "^19.1.3"
  }
}
```

```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "jsx": "react-jsx",
    "types": ["vite/client"]
  },
  "include": ["src"]
}
```

```ts
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://127.0.0.1:8787"
    }
  }
})
```

```html
<div id="root"></div>
<script type="module" src="/src/main.tsx"></script>
```

- [ ] **Step 4: Install dependencies**

Run: `npm install`

Expected: root `package-lock.json` is created and both workspace packages install without errors.

- [ ] **Step 5: Commit**

```bash
git add package.json package-lock.json tsconfig.base.json .gitignore .env.example backend frontend
git commit -m "chore: scaffold chat data bot app"
```

---

### Task 2: JSON Store And Core Types

**Files:**
- Create: `backend/src/config/paths.ts`
- Create: `backend/src/store/json-store.ts`
- Create: `backend/src/providers/types.ts`
- Create: `backend/src/agent/types.ts`
- Test: `backend/test/json-store.test.ts`

- [ ] **Step 1: Write failing JSON store tests**

```ts
import { mkdtemp, rm } from "node:fs/promises"
import { tmpdir } from "node:os"
import { join } from "node:path"
import { describe, expect, it, afterEach } from "vitest"
import { readJson, writeJson } from "../src/store/json-store"

let tempDir = ""

afterEach(async () => {
  if (tempDir) await rm(tempDir, { recursive: true, force: true })
})

describe("json-store", () => {
  it("writes and reads JSON with a fallback", async () => {
    tempDir = await mkdtemp(join(tmpdir(), "chat-data-bot-"))
    const file = join(tempDir, "config", "providers.json")

    const fallback = { providers: [] as string[] }
    expect(await readJson(file, fallback)).toEqual(fallback)

    await writeJson(file, { providers: ["zai-coding-plan"] })

    expect(await readJson(file, fallback)).toEqual({ providers: ["zai-coding-plan"] })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run test -w backend -- json-store`

Expected: FAIL because `backend/src/store/json-store.ts` does not exist.

- [ ] **Step 3: Implement paths and JSON store**

```ts
import { join } from "node:path"

export const DATA_DIR = process.env.CHAT_DATA_BOT_DATA_DIR ?? join(process.cwd(), "..", "data")

export const dataPath = (...parts: string[]) => join(DATA_DIR, ...parts)
```

```ts
import { mkdir, readFile, rename, writeFile } from "node:fs/promises"
import { dirname } from "node:path"

export async function readJson<T>(filePath: string, fallback: T): Promise<T> {
  try {
    const raw = await readFile(filePath, "utf8")
    return JSON.parse(raw) as T
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return fallback
    throw error
  }
}

export async function writeJson<T>(filePath: string, value: T): Promise<void> {
  await mkdir(dirname(filePath), { recursive: true })
  const tempPath = `${filePath}.tmp`
  await writeFile(tempPath, `${JSON.stringify(value, null, 2)}\n`, "utf8")
  await rename(tempPath, filePath)
}
```

- [ ] **Step 4: Add shared types**

```ts
export type ProviderProfileId = "siliconflow" | "zai-coding-plan" | "custom-openai-compatible"

export type ProviderProfile = {
  id: ProviderProfileId
  displayName: string
  protocol: "openai-compatible"
  defaultBaseUrl: string
  modelListPath?: string
  apiKeyEnvHint?: string
  defaultModels: string[]
  labels?: string[]
  extraBody?: Record<string, unknown>
}

export type EnabledModel = {
  providerId: ProviderProfileId
  modelId: string
  enabled: boolean
  capabilities: {
    tools: "known" | "unknown"
    vision: "known" | "unknown"
    audio: "known" | "unknown"
  }
}
```

```ts
export type ChatMessage = {
  id: string
  role: "user" | "assistant"
  content: string
  createdAt: string
}

export type TraceStep = {
  id: string
  iteration: number
  type: "plan" | "model_call" | "tool_call" | "observation" | "evaluation" | "ask_user" | "final"
  title: string
  summary: string
  inputPreview?: unknown
  outputPreview?: unknown
  raw?: unknown
  status: "success" | "empty" | "error" | "timeout" | "skipped" | "waiting_for_user"
  startedAt: string
  durationMs?: number
}

export type ChatSession = {
  id: string
  messages: ChatMessage[]
  createdAt: string
  updatedAt: string
}
```

- [ ] **Step 5: Run tests**

Run: `npm run test -w backend`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add backend/src/config backend/src/store backend/src/providers/types.ts backend/src/agent/types.ts backend/test/json-store.test.ts
git commit -m "feat: add json store and core types"
```

---

### Task 3: Provider Profiles And Model Discovery

**Files:**
- Create: `backend/src/providers/profiles.ts`
- Create: `backend/src/providers/model-discovery.ts`
- Test: `backend/test/provider-profiles.test.ts`
- Test: `backend/test/model-discovery.test.ts`

- [ ] **Step 1: Write failing provider profile tests**

```ts
import { describe, expect, it } from "vitest"
import { providerProfiles } from "../src/providers/profiles"

describe("providerProfiles", () => {
  it("defines Z.AI Coding Plan without a Z.AI General profile", () => {
    expect(providerProfiles["zai-coding-plan"].defaultBaseUrl).toBe("https://api.z.ai/api/coding/paas/v4")
    expect(providerProfiles["zai-coding-plan"].labels).toContain("coding-only")
    expect(Object.keys(providerProfiles)).not.toContain("zai-general")
  })

  it("injects Z.AI thinking parameters", () => {
    expect(providerProfiles["zai-coding-plan"].extraBody).toEqual({
      thinking: { type: "enabled", clear_thinking: false }
    })
  })
})
```

- [ ] **Step 2: Write failing model discovery tests**

```ts
import { describe, expect, it, vi } from "vitest"
import { discoverModels } from "../src/providers/model-discovery"

describe("discoverModels", () => {
  it("normalizes OpenAI-compatible model list responses", async () => {
    const fetcher = vi.fn(async () =>
      new Response(JSON.stringify({ data: [{ id: "glm-5.1" }, { id: "glm-4.7" }] }), { status: 200 })
    )

    const result = await discoverModels({
      baseUrl: "https://api.example.test/v1",
      apiKey: "test-key",
      fetcher
    })

    expect(fetcher).toHaveBeenCalledWith("https://api.example.test/v1/models", {
      headers: { Authorization: "Bearer test-key" }
    })
    expect(result.map((item) => item.modelId)).toEqual(["glm-5.1", "glm-4.7"])
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npm run test -w backend -- provider model-discovery`

Expected: FAIL because provider modules do not exist.

- [ ] **Step 4: Implement provider profiles**

```ts
import type { ProviderProfile, ProviderProfileId } from "./types"

export const providerProfiles: Record<ProviderProfileId, ProviderProfile> = {
  siliconflow: {
    id: "siliconflow",
    displayName: "SiliconFlow",
    protocol: "openai-compatible",
    defaultBaseUrl: "https://api.siliconflow.cn/v1",
    modelListPath: "/models",
    apiKeyEnvHint: "SILICONFLOW_API_KEY",
    defaultModels: []
  },
  "zai-coding-plan": {
    id: "zai-coding-plan",
    displayName: "Z.AI Coding Plan",
    protocol: "openai-compatible",
    defaultBaseUrl: "https://api.z.ai/api/coding/paas/v4",
    modelListPath: "/models",
    apiKeyEnvHint: "ZHIPU_API_KEY",
    defaultModels: ["glm-5.1", "glm-5-turbo", "glm-4.7", "glm-4.5-air"],
    labels: ["coding-only", "experimental"],
    extraBody: {
      thinking: { type: "enabled", clear_thinking: false }
    }
  },
  "custom-openai-compatible": {
    id: "custom-openai-compatible",
    displayName: "Custom OpenAI-Compatible",
    protocol: "openai-compatible",
    defaultBaseUrl: "",
    modelListPath: "/models",
    defaultModels: []
  }
}
```

- [ ] **Step 5: Implement model discovery**

```ts
import { z } from "zod"

const ModelListResponse = z.object({
  data: z.array(z.object({ id: z.string() }))
})

export type DiscoveredModel = {
  modelId: string
  raw: unknown
}

export async function discoverModels(input: {
  baseUrl: string
  apiKey: string
  fetcher?: typeof fetch
}): Promise<DiscoveredModel[]> {
  const fetcher = input.fetcher ?? fetch
  const url = `${input.baseUrl.replace(/\/$/, "")}/models`
  const response = await fetcher(url, {
    headers: { Authorization: `Bearer ${input.apiKey}` }
  })
  if (!response.ok) throw new Error(`Model discovery failed: ${response.status}`)
  const parsed = ModelListResponse.parse(await response.json())
  return parsed.data.map((item) => ({ modelId: item.id, raw: item }))
}
```

- [ ] **Step 6: Run tests**

Run: `npm run test -w backend`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add backend/src/providers backend/test/provider-profiles.test.ts backend/test/model-discovery.test.ts
git commit -m "feat: add provider profiles and model discovery"
```

---

### Task 4: Z.AI Media Probe Request Builders

**Files:**
- Create: `backend/src/providers/zai-media.ts`
- Test: `backend/test/zai-media.test.ts`

- [ ] **Step 1: Write failing tests**

```ts
import { describe, expect, it } from "vitest"
import { buildZaiAudioTranscriptionRequest, buildZaiImageGenerationRequest } from "../src/providers/zai-media"

describe("zai-media probes", () => {
  it("builds a disabled-by-default image probe request", () => {
    const request = buildZaiImageGenerationRequest({
      enabled: true,
      apiKey: "key",
      prompt: "architecture diagram",
      model: "glm-image",
      size: "1280x1280"
    })

    expect(request.url).toBe("https://api.z.ai/api/paas/v4/images/generations")
    expect(request.init.headers).toEqual({
      Authorization: "Bearer key",
      "Content-Type": "application/json"
    })
  })

  it("rejects image probes unless explicitly enabled", () => {
    expect(() =>
      buildZaiImageGenerationRequest({
        enabled: false,
        apiKey: "key",
        prompt: "architecture diagram",
        model: "glm-image",
        size: "1280x1280"
      })
    ).toThrow("Z.AI media probes are disabled")
  })

  it("builds an audio transcription request", () => {
    const file = new Blob(["audio"])
    const request = buildZaiAudioTranscriptionRequest({
      enabled: true,
      apiKey: "key",
      model: "glm-asr-2512",
      file
    })

    expect(request.url).toBe("https://api.z.ai/api/paas/v4/audio/transcriptions")
    expect(request.init.headers).toEqual({ Authorization: "Bearer key" })
    expect(request.init.body).toBeInstanceOf(FormData)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run test -w backend -- zai-media`

Expected: FAIL because `zai-media.ts` does not exist.

- [ ] **Step 3: Implement probe builders**

```ts
export type ZaiImageModel = "glm-image" | "cogview-4-250304"

export function buildZaiImageGenerationRequest(input: {
  enabled: boolean
  apiKey: string
  prompt: string
  model: ZaiImageModel
  size: string
  quality?: "standard" | "hd"
}) {
  if (!input.enabled) throw new Error("Z.AI media probes are disabled")
  return {
    url: "https://api.z.ai/api/paas/v4/images/generations",
    init: {
      method: "POST",
      headers: {
        Authorization: `Bearer ${input.apiKey}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        model: input.model,
        prompt: input.prompt,
        size: input.size,
        ...(input.quality ? { quality: input.quality } : {})
      })
    } satisfies RequestInit
  }
}

export function buildZaiAudioTranscriptionRequest(input: {
  enabled: boolean
  apiKey: string
  model: "glm-asr-2512"
  file: Blob
  prompt?: string
  hotwords?: string[]
}) {
  if (!input.enabled) throw new Error("Z.AI media probes are disabled")
  const body = new FormData()
  body.set("model", input.model)
  body.set("stream", "false")
  body.set("file", input.file)
  if (input.prompt) body.set("prompt", input.prompt)
  for (const word of input.hotwords ?? []) body.append("hotwords", word)
  return {
    url: "https://api.z.ai/api/paas/v4/audio/transcriptions",
    init: {
      method: "POST",
      headers: { Authorization: `Bearer ${input.apiKey}` },
      body
    } satisfies RequestInit
  }
}
```

- [ ] **Step 4: Run tests**

Run: `npm run test -w backend`

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add backend/src/providers/zai-media.ts backend/test/zai-media.test.ts
git commit -m "feat: add zai media probe builders"
```

---

### Task 5: MCP Client And Tool Registry

**Files:**
- Create: `backend/src/mcp/types.ts`
- Create: `backend/src/mcp/http-sse-client.ts`
- Create: `backend/src/tools/tool-registry.ts`
- Test: `backend/test/tool-registry.test.ts`
- Test: `backend/test/mcp-client.test.ts`

- [ ] **Step 1: Write failing tool registry test**

```ts
import { describe, expect, it } from "vitest"
import { ToolRegistry } from "../src/tools/tool-registry"

describe("ToolRegistry", () => {
  it("registers MCP tools and local actions", () => {
    const registry = new ToolRegistry()
    registry.registerLocalAction({ name: "ASK_USER", description: "Ask user for clarification" })
    registry.registerMcpTool({
      serverName: "ragflow-local",
      name: "search",
      description: "Search RAGFlow",
      inputSchema: { type: "object", properties: { query: { type: "string" } } }
    })

    expect(registry.list().map((tool) => tool.name)).toEqual(["ASK_USER", "ragflow-local.search"])
  })
})
```

- [ ] **Step 2: Write failing MCP client test**

```ts
import { describe, expect, it, vi } from "vitest"
import { McpHttpClient } from "../src/mcp/http-sse-client"

describe("McpHttpClient", () => {
  it("normalizes discovered MCP tools", async () => {
    const fetcher = vi.fn(async () =>
      new Response(
        JSON.stringify({
          tools: [
            {
              name: "search",
              description: "Search knowledge base",
              inputSchema: { type: "object", properties: { query: { type: "string" } } }
            }
          ]
        }),
        { status: 200 }
      )
    )
    const client = new McpHttpClient({ endpoint: "http://localhost:9382/sse", fetcher })

    const tools = await client.listTools()

    expect(tools[0]?.name).toBe("search")
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npm run test -w backend -- tool-registry mcp-client`

Expected: FAIL because modules do not exist.

- [ ] **Step 4: Implement MCP types and client**

```ts
export type McpTool = {
  name: string
  description?: string
  inputSchema: Record<string, unknown>
}

export type McpConnectionConfig = {
  serverName: string
  endpoint: string
  timeoutMs: number
  allowedTools: "all" | string[]
}
```

```ts
import { z } from "zod"
import type { McpTool } from "./types"

const ToolListResponse = z.object({
  tools: z.array(
    z.object({
      name: z.string(),
      description: z.string().optional(),
      inputSchema: z.record(z.unknown()).default({})
    })
  )
})

export class McpHttpClient {
  constructor(private readonly input: { endpoint: string; fetcher?: typeof fetch }) {}

  async listTools(): Promise<McpTool[]> {
    const response = await (this.input.fetcher ?? fetch)(this.input.endpoint, {
      headers: { Accept: "application/json" }
    })
    if (!response.ok) throw new Error(`MCP tool discovery failed: ${response.status}`)
    return ToolListResponse.parse(await response.json()).tools
  }
}
```

- [ ] **Step 5: Implement tool registry**

```ts
import type { McpTool } from "../mcp/types"

export type RegisteredTool = {
  name: string
  description: string
  inputSchema: Record<string, unknown>
  source: "local" | "mcp"
}

export class ToolRegistry {
  private readonly tools = new Map<string, RegisteredTool>()

  registerLocalAction(input: { name: string; description: string }) {
    this.tools.set(input.name, {
      name: input.name,
      description: input.description,
      inputSchema: { type: "object", properties: {} },
      source: "local"
    })
  }

  registerMcpTool(input: McpTool & { serverName: string }) {
    const name = `${input.serverName}.${input.name}`
    this.tools.set(name, {
      name,
      description: input.description ?? "",
      inputSchema: input.inputSchema,
      source: "mcp"
    })
  }

  list() {
    return [...this.tools.values()]
  }
}
```

- [ ] **Step 6: Run tests**

Run: `npm run test -w backend`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add backend/src/mcp backend/src/tools backend/test/tool-registry.test.ts backend/test/mcp-client.test.ts
git commit -m "feat: add mcp client and tool registry"
```

---

### Task 6: Clarification Policy And Agent Loop

**Files:**
- Create: `backend/src/agent/clarification-policy.ts`
- Create: `backend/src/agent/agent-loop.ts`
- Test: `backend/test/clarification-policy.test.ts`
- Test: `backend/test/agent-loop.test.ts`

- [ ] **Step 1: Write failing clarification policy test**

```ts
import { describe, expect, it } from "vitest"
import { shouldAskUser } from "../src/agent/clarification-policy"

describe("shouldAskUser", () => {
  it("waits until retrieval repair attempts are exhausted in balanced mode", () => {
    expect(
      shouldAskUser({
        mode: "balanced",
        missingRequiredSlot: false,
        retrievalProbeCount: 2,
        retryRound: 1,
        usefulChunkCount: 0,
        contradictoryEvidence: false
      })
    ).toBe(false)

    expect(
      shouldAskUser({
        mode: "balanced",
        missingRequiredSlot: false,
        retrievalProbeCount: 4,
        retryRound: 2,
        usefulChunkCount: 0,
        contradictoryEvidence: false
      })
    ).toBe(true)
  })
})
```

- [ ] **Step 2: Write failing agent loop test**

```ts
import { describe, expect, it } from "vitest"
import { runAgentLoop } from "../src/agent/agent-loop"

describe("runAgentLoop", () => {
  it("stops with ASK_USER after low-value retrieval attempts", async () => {
    const result = await runAgentLoop({
      userQuery: "这个公司怎么样",
      maxIterations: 3,
      plan: async (iteration) =>
        iteration < 3
          ? { type: "CALL_TOOL", toolName: "ragflow-local.search", input: { query: "company" } }
          : { type: "ASK_USER", question: "你想查询哪家公司？" },
      callTool: async () => ({ status: "empty", chunks: [] }),
      evaluate: async () => ({ answerReady: false, usefulChunkCount: 0 })
    })

    expect(result.status).toBe("waiting_for_user")
    expect(result.trace.at(-1)?.type).toBe("ask_user")
  })
})
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npm run test -w backend -- clarification-policy agent-loop`

Expected: FAIL because modules do not exist.

- [ ] **Step 4: Implement clarification policy**

```ts
export type ClarificationMode = "conservative" | "balanced" | "aggressive"

export function shouldAskUser(input: {
  mode: ClarificationMode
  missingRequiredSlot: boolean
  retrievalProbeCount: number
  retryRound: number
  usefulChunkCount: number
  contradictoryEvidence: boolean
}) {
  if (input.missingRequiredSlot) return true
  if (input.mode === "aggressive" && input.contradictoryEvidence) return true
  if (input.mode === "conservative") {
    return input.retryRound >= 2 && input.retrievalProbeCount >= 4 && input.usefulChunkCount === 0
  }
  return input.retryRound >= 2 && input.retrievalProbeCount >= 4 && input.usefulChunkCount === 0
}
```

- [ ] **Step 5: Implement bounded agent loop**

```ts
import type { TraceStep } from "./types"

type PlanAction =
  | { type: "CALL_TOOL"; toolName: string; input: unknown }
  | { type: "ASK_USER"; question: string }
  | { type: "FINAL_ANSWER"; answer: string }

type ToolResult = { status: "success" | "empty" | "error"; chunks?: unknown[]; raw?: unknown }
type Evaluation = { answerReady: boolean; usefulChunkCount: number; answer?: string }

export async function runAgentLoop(input: {
  userQuery: string
  maxIterations: number
  plan: (iteration: number) => Promise<PlanAction>
  callTool: (toolName: string, args: unknown) => Promise<ToolResult>
  evaluate: (result: ToolResult | undefined) => Promise<Evaluation>
}): Promise<{ status: "answered" | "waiting_for_user" | "budget_exhausted"; answer?: string; trace: TraceStep[] }> {
  const trace: TraceStep[] = []
  let lastToolResult: ToolResult | undefined

  for (let iteration = 1; iteration <= input.maxIterations; iteration++) {
    const startedAt = new Date().toISOString()
    const action = await input.plan(iteration)

    if (action.type === "ASK_USER") {
      trace.push({
        id: `step-${iteration}-ask`,
        iteration,
        type: "ask_user",
        title: "Ask user",
        summary: action.question,
        status: "waiting_for_user",
        startedAt
      })
      return { status: "waiting_for_user", trace }
    }

    if (action.type === "FINAL_ANSWER") {
      trace.push({
        id: `step-${iteration}-final`,
        iteration,
        type: "final",
        title: "Final answer",
        summary: action.answer,
        status: "success",
        startedAt
      })
      return { status: "answered", answer: action.answer, trace }
    }

    trace.push({
      id: `step-${iteration}-tool`,
      iteration,
      type: "tool_call",
      title: action.toolName,
      summary: "Calling tool",
      inputPreview: action.input,
      status: "success",
      startedAt
    })

    lastToolResult = await input.callTool(action.toolName, action.input)
    const evaluation = await input.evaluate(lastToolResult)

    trace.push({
      id: `step-${iteration}-eval`,
      iteration,
      type: "evaluation",
      title: "Evaluate evidence",
      summary: evaluation.answerReady ? "Answer ready" : "Need more evidence",
      outputPreview: evaluation,
      status: evaluation.usefulChunkCount > 0 ? "success" : "empty",
      startedAt: new Date().toISOString()
    })

    if (evaluation.answerReady && evaluation.answer) {
      return { status: "answered", answer: evaluation.answer, trace }
    }
  }

  return { status: "budget_exhausted", trace }
}
```

- [ ] **Step 6: Run tests**

Run: `npm run test -w backend`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add backend/src/agent backend/test/clarification-policy.test.ts backend/test/agent-loop.test.ts
git commit -m "feat: add bounded agent loop"
```

---

### Task 7: Backend API

**Files:**
- Create: `backend/src/routes/config-routes.ts`
- Create: `backend/src/routes/chat-routes.ts`
- Create: `backend/src/server.ts`
- Test: `backend/test/server.test.ts`

- [ ] **Step 1: Write failing API smoke test**

```ts
import { describe, expect, it } from "vitest"
import { createServer } from "../src/server"

describe("server", () => {
  it("returns health status", async () => {
    const app = createServer()
    const server = app.listen(0)
    const address = server.address()
    if (!address || typeof address === "string") throw new Error("No port")
    const response = await fetch(`http://127.0.0.1:${address.port}/api/health`)
    await new Promise<void>((resolve) => server.close(() => resolve()))

    expect(await response.json()).toEqual({ ok: true })
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm run test -w backend -- server`

Expected: FAIL because `server.ts` does not exist.

- [ ] **Step 3: Implement config routes**

```ts
import { Router } from "express"
import { dataPath } from "../config/paths"
import { providerProfiles } from "../providers/profiles"
import { discoverModels } from "../providers/model-discovery"
import { readJson, writeJson } from "../store/json-store"

export const configRoutes = Router()

configRoutes.get("/providers", (_req, res) => {
  res.json({ profiles: providerProfiles })
})

configRoutes.get("/config/providers", async (_req, res) => {
  res.json(await readJson(dataPath("config", "providers.json"), { providers: {} }))
})

configRoutes.post("/config/providers", async (req, res) => {
  await writeJson(dataPath("config", "providers.json"), req.body)
  res.json({ ok: true })
})

configRoutes.post("/providers/discover-models", async (req, res) => {
  const models = await discoverModels({
    baseUrl: String(req.body.baseUrl),
    apiKey: String(req.body.apiKey)
  })
  res.json({ models })
})
```

- [ ] **Step 4: Implement chat routes**

```ts
import { randomUUID } from "node:crypto"
import { Router } from "express"
import { runAgentLoop } from "../agent/agent-loop"

export const chatRoutes = Router()

chatRoutes.post("/sessions", (_req, res) => {
  res.json({ sessionId: randomUUID() })
})

chatRoutes.post("/chat", async (req, res) => {
  const query = String(req.body.message ?? "")
  const result = await runAgentLoop({
    userQuery: query,
    maxIterations: 2,
    plan: async (iteration) =>
      iteration === 1
        ? { type: "CALL_TOOL", toolName: "ragflow-local.search", input: { query } }
        : { type: "FINAL_ANSWER", answer: "POC response. Real model and MCP wiring comes in the next task." },
    callTool: async () => ({ status: "empty", chunks: [] }),
    evaluate: async () => ({ answerReady: false, usefulChunkCount: 0 })
  })
  res.json(result)
})
```

- [ ] **Step 5: Implement server**

```ts
import cors from "cors"
import express from "express"
import { chatRoutes } from "./routes/chat-routes"
import { configRoutes } from "./routes/config-routes"

export function createServer() {
  const app = express()
  app.use(cors())
  app.use(express.json({ limit: "2mb" }))
  app.get("/api/health", (_req, res) => res.json({ ok: true }))
  app.use("/api", configRoutes)
  app.use("/api", chatRoutes)
  return app
}

if (import.meta.url === `file://${process.argv[1]}`) {
  createServer().listen(8787, "127.0.0.1", () => {
    console.log("backend listening on http://127.0.0.1:8787")
  })
}
```

- [ ] **Step 6: Run tests and typecheck**

Run: `npm run test -w backend && npm run typecheck -w backend`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add backend/src/routes backend/src/server.ts backend/test/server.test.ts
git commit -m "feat: add local backend api"
```

---

### Task 8: Frontend Chat, Config, And Trace UI

**Files:**
- Create: `frontend/src/main.tsx`
- Create: `frontend/src/App.tsx`
- Create: `frontend/src/api/client.ts`
- Create: `frontend/src/components/ChatView.tsx`
- Create: `frontend/src/components/TracePanel.tsx`
- Create: `frontend/src/components/ModelConfigView.tsx`
- Create: `frontend/src/components/McpConfigView.tsx`
- Create: `frontend/src/styles.css`

- [ ] **Step 1: Create API client**

```ts
export async function apiGet<T>(path: string): Promise<T> {
  const response = await fetch(path)
  if (!response.ok) throw new Error(`GET ${path} failed: ${response.status}`)
  return response.json() as Promise<T>
}

export async function apiPost<T>(path: string, body: unknown): Promise<T> {
  const response = await fetch(path, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body)
  })
  if (!response.ok) throw new Error(`POST ${path} failed: ${response.status}`)
  return response.json() as Promise<T>
}
```

- [ ] **Step 2: Create trace panel**

```tsx
type TraceStep = {
  id: string
  iteration: number
  type: string
  title: string
  summary: string
  status: string
  raw?: unknown
}

export function TracePanel(props: { trace: TraceStep[] }) {
  return (
    <details className="trace-panel">
      <summary>Trace ({props.trace.length})</summary>
      <div className="trace-list">
        {props.trace.map((step) => (
          <details key={step.id} className="trace-step">
            <summary>
              <span>#{step.iteration}</span>
              <strong>{step.title}</strong>
              <em>{step.status}</em>
            </summary>
            <p>{step.summary}</p>
            {step.raw ? <pre>{JSON.stringify(step.raw, null, 2)}</pre> : null}
          </details>
        ))}
      </div>
    </details>
  )
}
```

- [ ] **Step 3: Create chat view**

```tsx
import { useState } from "react"
import { apiPost } from "../api/client"
import { TracePanel } from "./TracePanel"

type ChatResult = { status: string; answer?: string; trace: any[] }

export function ChatView() {
  const [message, setMessage] = useState("")
  const [result, setResult] = useState<ChatResult | null>(null)
  const [loading, setLoading] = useState(false)

  async function send() {
    setLoading(true)
    try {
      setResult(await apiPost<ChatResult>("/api/chat", { message }))
    } finally {
      setLoading(false)
    }
  }

  return (
    <section className="panel">
      <h2>Chat</h2>
      <textarea value={message} onChange={(event) => setMessage(event.target.value)} />
      <button onClick={send} disabled={loading || !message.trim()}>
        {loading ? "Running..." : "Send"}
      </button>
      {result ? (
        <article className="answer">
          <h3>{result.status}</h3>
          <p>{result.answer ?? "No final answer yet."}</p>
          <TracePanel trace={result.trace} />
        </article>
      ) : null}
    </section>
  )
}
```

- [ ] **Step 4: Create model and MCP config views**

```tsx
import { useEffect, useState } from "react"
import { apiGet } from "../api/client"

export function ModelConfigView() {
  const [profiles, setProfiles] = useState<Record<string, unknown>>({})

  useEffect(() => {
    apiGet<{ profiles: Record<string, unknown> }>("/api/providers").then((data) => setProfiles(data.profiles))
  }, [])

  return (
    <section className="panel">
      <h2>Model Config</h2>
      <p>Provider profiles are loaded from the backend. Z.AI Coding Plan is marked coding-only.</p>
      <pre>{JSON.stringify(profiles, null, 2)}</pre>
    </section>
  )
}
```

```tsx
export function McpConfigView() {
  return (
    <section className="panel">
      <h2>RAGFlow MCP</h2>
      <label>
        Endpoint
        <input defaultValue="http://localhost:9382/sse" />
      </label>
      <p>Tool discovery wiring lands in the backend integration task.</p>
    </section>
  )
}
```

- [ ] **Step 5: Create app shell and styles**

```tsx
import { useState } from "react"
import { ChatView } from "./components/ChatView"
import { McpConfigView } from "./components/McpConfigView"
import { ModelConfigView } from "./components/ModelConfigView"
import "./styles.css"

type Tab = "chat" | "models" | "mcp"

export function App() {
  const [tab, setTab] = useState<Tab>("chat")
  return (
    <main>
      <header>
        <h1>Chat Data Bot Lab</h1>
        <nav>
          <button onClick={() => setTab("chat")}>Chat</button>
          <button onClick={() => setTab("models")}>Models</button>
          <button onClick={() => setTab("mcp")}>MCP</button>
        </nav>
      </header>
      {tab === "chat" ? <ChatView /> : null}
      {tab === "models" ? <ModelConfigView /> : null}
      {tab === "mcp" ? <McpConfigView /> : null}
    </main>
  )
}
```

```tsx
import { createRoot } from "react-dom/client"
import { App } from "./App"

createRoot(document.getElementById("root")!).render(<App />)
```

```css
body {
  margin: 0;
  font-family: Inter, ui-sans-serif, system-ui, sans-serif;
  color: #172026;
  background: #f6f7f8;
}

main {
  max-width: 1080px;
  margin: 0 auto;
  padding: 24px;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

nav,
.panel {
  display: flex;
  gap: 8px;
}

.panel {
  flex-direction: column;
  margin-top: 20px;
  padding: 16px;
  border: 1px solid #d8dde3;
  border-radius: 8px;
  background: white;
}

textarea {
  min-height: 120px;
  resize: vertical;
}

button,
input,
textarea {
  font: inherit;
}

button {
  width: fit-content;
  padding: 8px 12px;
  border: 1px solid #b7c0ca;
  border-radius: 6px;
  background: #ffffff;
}

.trace-panel,
.trace-step {
  border-top: 1px solid #e3e6ea;
  padding-top: 8px;
}

pre {
  overflow: auto;
  padding: 12px;
  background: #f0f2f4;
}
```

- [ ] **Step 6: Typecheck frontend**

Run: `npm run typecheck -w frontend`

Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add frontend/src
git commit -m "feat: add local learning ui"
```

---

### Task 9: Verification And Local Run

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add local run instructions to README**

Append:

```md
## Chat Data Bot Lab

This repository includes a local learning app for experimenting with a Claude Code-style agent loop.

Run:

```bash
npm install
npm run dev
```

Open:

```text
http://127.0.0.1:5173
```

The backend listens on `http://127.0.0.1:8787`.
```

- [ ] **Step 2: Run full verification**

Run: `npm run test && npm run typecheck && npm run build`

Expected: all commands pass.

- [ ] **Step 3: Start dev server**

Run: `npm run dev`

Expected:

```text
backend listening on http://127.0.0.1:8787
VITE ready in ...
Local: http://127.0.0.1:5173/
```

- [ ] **Step 4: Browser smoke test**

Open `http://127.0.0.1:5173`.

Verify:

- Chat tab loads.
- Sending a message returns a POC response and expandable trace.
- Models tab shows SiliconFlow, Z.AI Coding Plan, and Custom OpenAI-Compatible profiles.
- MCP tab shows the local RAGFlow endpoint field.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add chat data bot run instructions"
```

---

## Self-Review

Spec coverage:

- Local React/Vite plus Node backend: Tasks 1, 7, 8, 9.
- JSON persistence: Task 2 creates the store; later API tasks use it.
- SiliconFlow profile and model discovery: Task 3.
- Z.AI Coding Plan profile and thinking parameters: Task 3.
- Z.AI image/audio learning probes: Task 4.
- HTTP/SSE RAGFlow MCP and tool registry: Task 5.
- Agent loop, low-interruption clarification, retry/ASK_USER trace: Task 6.
- Collapsible trace UI: Task 8.
- Manual verification and local run: Task 9.

Scope note:

This plan produces a working vertical slice first. Real model-driven planner prompts, real RAGFlow MCP method details, and richer UI controls can be added after this foundation is verified, because they depend on the local RAGFlow server's actual tool schema and provider behavior.
