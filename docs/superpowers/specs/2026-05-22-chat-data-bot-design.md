# Chat Data Bot Design Spec

## Purpose

Build a local learning project that turns the repository's Claude Code architecture notes into a hands-on chat data bot.

The first version prioritizes the agent orchestration layer rather than a polished product UI. The bot should help the builder understand how a Claude Code-style system organizes model calls, tool registry, MCP tool execution, retry loops, user clarification, and visible execution traces.

## Product Shape

The product is a local browser-based learning tool:

- A lightweight chat page for asking mixed financial data questions.
- A model configuration page for OpenAI-compatible providers.
- A RAGFlow MCP configuration section for connecting to a local HTTP/SSE MCP endpoint.
- A collapsible trace panel that shows how the agent understood the question, chose actions, called tools, evaluated evidence, retried, and decided whether to ask the user for clarification.

The first version is single-user, local-only, and file-backed. It does not attempt deployment, authentication, permissions for multiple users, or a production-grade database.

## Primary Learning Goals

The project should teach these design ideas:

1. A chatbot is not only a prompt plus a final answer. It is a loop of planning, action, observation, evaluation, and stopping.
2. Tool calls should be registered through a tool registry rather than hard-coded into the prompt path.
3. MCP servers can be treated as tool providers whose schemas are discovered at runtime.
4. User clarification is one possible action in the loop, not a fixed pre-answer step.
5. The system should expose trace data so the builder can inspect why the agent chose a retry, a tool call, or a user question.

## Scope

### In Scope

- React/Vite frontend.
- Node.js TypeScript backend.
- Local JSON persistence for configuration, sessions, messages, and traces.
- OpenAI-compatible model adapter with provider settings.
- SiliconFlow profile with API key input, default base URL, model list retrieval, and checkbox model enablement.
- Z.AI Coding Plan profile with explicit coding-only scope, OpenAI-compatible coding endpoint, and GLM model role assignment.
- Z.AI media probe adapters for learning image generation and speech-to-text API shapes without adding a separate Z.AI General provider profile.
- Light model role assignment: default, planner, answerer, evaluator.
- HTTP/SSE RAGFlow MCP connection.
- MCP tool discovery and registration into the internal tool registry.
- Agent loop with planner, action execution, observation, evaluator, retry, and final answer.
- Low-interruption clarification policy: ask the user only after cheap self-repair and retrieval probes fail, or when required slots are impossible to infer.
- Collapsible trace UI for reasoning steps and MCP tool calls.

### Out of Scope

- Multi-user accounts.
- Cloud deployment.
- Full permissions system equivalent to Claude Code.
- Arbitrary local MCP server marketplace.
- Complex financial data engines outside RAGFlow MCP.
- Long-term memory beyond local session files.
- Multi-agent worker orchestration.
- Automatic model cost optimization.
- Production observability, billing, or audit compliance.

## Architecture

Use an agent-loop-first architecture.

```text
Browser UI
  -> Backend API
  -> ChatSessionService
  -> AgentLoop
       -> ModelRouter
       -> ToolRegistry
       -> McpHttpClient
       -> TraceRecorder
  -> Local JSON Store
```

The frontend is intentionally thin. The backend owns all agent decisions, model calls, tool calls, trace recording, and persistence.

## Major Components

### Frontend

The frontend contains three main views:

- Chat view: message list, input box, answer display, and collapsible trace panel for each assistant response.
- Model configuration view: provider settings, model discovery, enabled model checkboxes, and role assignment.
- MCP configuration view: HTTP/SSE endpoint, connection test, tool list, and allowed tool selection.

The UI should favor observability over visual sophistication. The trace panel is part of the learning surface, not a debugging afterthought.

### Backend API

The backend exposes local API endpoints for:

- Reading and saving provider configuration.
- Testing model provider connection.
- Refreshing provider model list.
- Reading and saving model role assignments.
- Running explicit Z.AI image and audio probe calls when enabled for learning.
- Reading and saving RAGFlow MCP endpoint configuration.
- Testing MCP connection and listing tools.
- Creating chat sessions.
- Sending user messages into the agent loop.
- Returning assistant messages and trace records.

### Local JSON Store

Use local JSON files to keep the first implementation inspectable:

```text
data/config/providers.json
data/config/models.json
data/config/mcp.json
data/sessions/<session-id>.json
data/traces/<session-id>/<turn-id>.json
```

The store should use small repository-style functions such as `readConfig`, `writeConfig`, `appendMessage`, and `writeTrace`. It should not expose raw file paths to the agent loop.

## Model Configuration

The model layer uses an OpenAI-compatible adapter.

### Provider Settings

Minimum fields:

- provider name
- base URL
- API key
- default model
- optional request timeout

SiliconFlow should be provided as a built-in profile:

- default base URL: `https://api.siliconflow.cn/v1`
- model list endpoint: `GET /models`
- authorization: bearer token

SiliconFlow model list behavior was verified against the official documentation on 2026-05-22. The documentation describes `GET https://api.siliconflow.cn/v1/models` and supports optional model filters such as `type` and `sub_type`. Reference: https://docs.siliconflow.cn/en/api-reference/models/get-model-list

Z.AI Coding Plan should also be provided as a built-in profile:

- profile id: `zai-coding-plan`
- display name: `Z.AI Coding Plan`
- protocol: OpenAI-compatible chat completions
- default base URL: `https://api.z.ai/api/coding/paas/v4`
- API key environment hint: `ZHIPU_API_KEY`
- default models: `glm-5.1`, `glm-5-turbo`, `glm-4.7`, `glm-4.5-air`
- default extra body for Z.AI GLM models:

```json
{
  "thinking": {
    "type": "enabled",
    "clear_thinking": false
  }
}
```

The Z.AI Coding Plan profile is intentionally separate from any general Z.AI API profile. This spec does not add a Z.AI General provider. The Coding Plan profile should be labeled `coding-only` and `experimental` in the UI because Z.AI's public documentation states that Coding Plan uses the dedicated coding endpoint and is intended for supported coding tools and scenarios. Reference: https://docs.z.ai/api-reference/introduction

### Provider Usage Policy

The app should keep provider usage explicit:

- `zai-coding-plan` may be used for coding-agent experiments, repository/documentation RAG for development work, API documentation lookup, tool-calling experiments, and OpenCode/OpenClaw-style coding workflow studies.
- `zai-coding-plan` should not be used as the default route for financial RAG, customer support, roleplay, marketing chat, or broad non-coding knowledge-base workloads.
- The app must not spoof Claude Code, OpenCode, or OpenClaw client headers, version identifiers, user agents, or session attribution.
- The app may implement protocol-compatible calls and OpenCode-inspired provider configuration, but it should identify itself truthfully as this local learning project.
- Non-coding chatbot traffic should be routed to non-Coding-Plan providers such as SiliconFlow or another custom OpenAI-compatible provider.

### Z.AI Media Probes

The project should include explicit learning probes for Z.AI image and audio APIs, but these probes are not part of the core chat data bot loop in v1.

Image generation probe:

- official REST path documented by Z.AI: `POST /images/generations`
- documented base endpoint: `https://api.z.ai/api/paas/v4/images/generations`
- models: `glm-image`, `cogview-4-250304`
- key parameters: `model`, `prompt`, `size`, `quality`
- reference: https://docs.z.ai/api-reference/image/generate-image

Speech-to-text probe:

- official REST path documented by Z.AI: `POST /audio/transcriptions`
- documented base endpoint: `https://api.z.ai/api/paas/v4/audio/transcriptions`
- model: `glm-asr-2512`
- input: multipart audio file or `file_base64`
- constraints: `.wav` or `.mp3`, file size up to 25 MB, duration up to 30 seconds
- reference: https://docs.z.ai/api-reference/audio/audio-transcriptions

Because the public image and audio documentation uses the general `/api/paas/v4` endpoint while Coding Plan uses `/api/coding/paas/v4`, v1 should treat media probes as endpoint compatibility experiments:

1. The UI should state that media probes may not be covered by Coding Plan.
2. The implementation should allow the probe to be disabled by default.
3. The trace should record the exact endpoint used, model, status code, and error response.
4. The app should not silently fall back from the Coding Plan endpoint to a general Z.AI provider profile, because this spec intentionally does not define a Z.AI General provider.
5. If the media endpoint requires ordinary API billing or rejects the Coding Plan endpoint, the app should show that result as a learning outcome rather than trying to bypass it.

### Model Discovery

The configuration page should let the user:

1. Enter an API key.
2. Test the provider connection.
3. Refresh the model list.
4. Select enabled models with checkboxes.
5. Assign models to roles.

The model list endpoint may not provide complete capability metadata. The system should store capability fields as `known`, `unknown`, or user-overridden rather than pretending the provider response is complete.

### Model Roles

Support light role assignment:

- default model
- planner model
- answerer model
- evaluator model

The default behavior is that all roles use the same selected default model. Users may override roles later.

## RAGFlow MCP Integration

The first version connects directly to a local RAGFlow MCP server over HTTP/SSE.

Configuration fields:

- server name, default `ragflow-local`
- transport, fixed to HTTP/SSE for v1
- endpoint URL, such as `http://localhost:<port>/sse`
- timeout
- allowed tools mode: all discovered tools or selected tools

Startup behavior:

```text
read MCP config
connect to HTTP/SSE endpoint
list available tools
normalize tool schemas
register each allowed MCP tool in ToolRegistry
surface connection status to UI
```

The agent should not call arbitrary MCP servers in v1. It should only use the configured RAGFlow MCP server.

## Tool Registry

The internal tool registry stores every executable action the agent may choose.

Initial tool types:

- MCP tools discovered from RAGFlow.
- `ASK_USER`, a local action that pauses the loop and asks the user for clarification.
- `FINAL_ANSWER`, a local action that completes the turn.

Internal planning labels may include:

- `PLAN_SUBQUERIES`
- `CLARIFY_TERMS`
- `EXPAND_TIME_RANGE`
- `EVALUATE_EVIDENCE`
- `RETRY_WITH_QUERY`

These labels do not all need to be external tools. Some can be structured model outputs consumed by the agent loop.

## Agent Loop

The core design is a bounded planner-observer-evaluator loop.

```text
receive user query
load session context
create initial trace
loop until stop:
  planner decides next action
  execute action
  observe action result
  evaluator judges answer readiness and evidence quality
  record trace step
  decide continue, retry, ask user, or final
return final answer or clarification request
```

### Loop Actions

The planner may choose:

- Use financial dictionary expansion.
- Rewrite the query.
- Split the query into subqueries.
- Expand relative or ambiguous time expressions.
- Run one or more retrieval probes through RAGFlow MCP tools.
- Retry with a different query.
- Ask the user for missing information.
- Produce the final answer.

### Stop Conditions

The loop stops when one of these is true:

- The evaluator marks the answer as complete enough.
- The agent emits an `ASK_USER` action and waits for user input.
- Maximum iterations are reached.
- Maximum MCP tool calls are reached.
- Tool errors or empty retrievals show the system cannot improve without user input.
- The model provider fails in a way the current turn cannot recover from.

Default budgets:

- maximum iterations: 6
- maximum MCP tool calls per user turn: 8
- maximum retrieval retry rounds: 2

These values should be configurable later, but hard-coded defaults are acceptable in the first implementation.

## Clarification Policy

The default clarification policy should avoid interrupting the user.

Clarification is not triggered merely because the question is ambiguous. The agent should first attempt cheap repair:

1. Use session context.
2. Apply financial term aliases and dictionary expansion.
3. Expand time expressions when possible.
4. Rewrite the query.
5. Run parallel or near-parallel retrieval probes.
6. Evaluate whether retrieved chunks are empty, weak, or contradictory.
7. Retry with improved queries.
8. Ask the user only if still blocked.

Configuration should expose a clarification tendency:

- Conservative: ask only when a required slot is missing and cannot be inferred.
- Balanced: default. Try cheap repair and retrieval probes before asking.
- Aggressive: ask earlier for high-impact ambiguity such as multiple company aliases, conflicting financial metrics, or time ranges that would change the conclusion.

## Trace Design

Each assistant turn should produce a trace record that can be rendered as collapsible steps.

Trace step fields:

- step id
- iteration number
- type: plan, model_call, tool_call, observation, evaluation, ask_user, final
- title
- summary
- input preview
- output preview
- raw JSON, collapsible
- status: success, empty, error, timeout, skipped
- timestamps and duration

Example trace:

```text
Iteration 1
- plan: split the question into revenue trend and margin trend
- tool_call: RAGFlow MCP search for revenue trend
- observation: 3 chunks found
- evaluation: evidence missing latest year

Iteration 2
- plan: retry with explicit fiscal year
- tool_call: RAGFlow MCP search
- observation: 2 chunks found
- evaluation: answer ready

Final
- answer with citations
```

If the system asks the user:

```text
Iteration 3
- ask_user: "你说的增长是同比、环比，还是 CAGR？"
- status: waiting_for_user
```

## Answer Behavior

Final answers should:

- Answer the user's question directly.
- Include important caveats when evidence is weak.
- Cite retrieved chunks when RAGFlow returns source metadata.
- Separate factual evidence from model inference.
- State when the bot could not find enough supporting material.

For mixed financial questions, the bot should prefer explicit metric names, company/entity names, and time ranges. If it infers a term or time range, the trace should show that inference.

## Error Handling

### Model Provider Errors

- Invalid API key: show provider connection error and do not start a chat turn.
- Model unavailable: suggest selecting another enabled model.
- Rate limit: record the error in trace and stop the turn with a concise user-facing message.
- Invalid structured output: retry once with a stricter repair prompt, then fail gracefully.

### MCP Errors

- Endpoint unavailable: show MCP status as disconnected.
- Tool list failure: do not register MCP tools.
- Tool timeout: record timeout trace step and allow the evaluator to retry or ask the user.
- Empty retrieval: treat as an observation, not an exception.
- Malformed tool result: store raw output in trace and summarize the parse failure.

### Agent Loop Errors

- Budget exhausted: return the best partial answer if evidence exists, otherwise ask the user for a narrower query.
- Repeated low-quality retrieval: ask the user for one targeted clarification.
- Conflicting evidence: answer with the conflict visible, or ask the user to choose scope if the conflict comes from ambiguous query intent.

## Testing Strategy

The first implementation should include focused tests around the backend because the learning goal is agent orchestration.

Recommended test layers:

- Unit tests for provider config validation.
- Unit tests for model discovery response normalization.
- Unit tests for Z.AI Coding Plan extra body injection.
- Unit tests for Z.AI media probe request construction and disabled-by-default behavior.
- Unit tests for MCP tool schema normalization.
- Unit tests for agent loop stop conditions.
- Unit tests for clarification policy.
- Integration test with a fake MCP server returning empty, partial, and successful retrievals.
- Integration test with a fake OpenAI-compatible model returning structured planner/evaluator outputs.
- Frontend smoke tests for chat page, model config page, and trace expansion.

Manual verification should include:

- Start the local app.
- Configure SiliconFlow or custom OpenAI-compatible provider.
- Configure Z.AI Coding Plan for a coding-agent test session.
- Refresh models and enable at least one chat model.
- Run disabled-by-default Z.AI image and audio probes only when intentionally enabled, and confirm the trace records endpoint, request shape, response, and any Coding Plan compatibility error.
- Configure RAGFlow MCP endpoint.
- Verify tool list appears.
- Ask a question that succeeds.
- Ask a question that requires retry.
- Ask a question that eventually triggers `ASK_USER`.

## Implementation Order

1. Create the TypeScript project structure with frontend and backend.
2. Add local JSON persistence.
3. Build provider configuration and model discovery.
4. Add the Z.AI Coding Plan provider profile and extra body handling.
5. Add disabled-by-default Z.AI image and audio media probes.
6. Build MCP HTTP/SSE connection and tool discovery.
7. Build the internal tool registry.
8. Build a fake-model and fake-MCP path for deterministic tests.
9. Implement the bounded agent loop.
10. Add trace recording.
11. Build chat UI and trace viewer.
12. Build model and MCP configuration UI.
13. Wire real provider and RAGFlow MCP calls.

## Success Criteria

The v1 design is successful when a local user can:

1. Open the browser UI.
2. Configure an OpenAI-compatible provider.
3. Refresh model list and enable models.
4. Assign planner, answerer, and evaluator roles, with all roles defaulting to the same model.
5. Configure Z.AI Coding Plan for a coding-agent compatibility session without spoofing another client.
6. Intentionally enable a Z.AI image or audio probe and inspect the exact endpoint, response, and compatibility result.
7. Configure a local RAGFlow HTTP/SSE MCP endpoint.
8. See discovered MCP tools.
9. Ask a mixed financial data question.
10. See the agent perform planning, MCP tool calls, evaluation, retry, and final answering.
11. Expand the trace to inspect each loop iteration.
12. See the agent ask for clarification only after self-repair and retrieval attempts fail.

## Open Risks

- RAGFlow MCP's exact tool names and schemas may vary by local setup, so the design must rely on runtime discovery.
- OpenAI-compatible providers differ in structured output, tool calling, and model metadata support.
- SiliconFlow's model list endpoint provides useful model IDs, but complete capability detection may still require local profiles or user overrides.
- Z.AI Coding Plan usage rules are policy-sensitive. The app should not treat technical endpoint compatibility as permission to run non-coding workloads.
- Z.AI image generation and audio transcription documentation currently describes general `/api/paas/v4` media endpoints, while Coding Plan uses `/api/coding/paas/v4`; media probes may fail or require ordinary API billing.
- LLM-based planner and evaluator steps may produce invalid JSON unless prompts and parsers are strict.
- Parallel retrieval probes can increase cost and latency; v1 should keep budgets small and visible.

## Non-Goals for V1

- Recreating Claude Code internals.
- Building a general-purpose agent platform.
- Supporting every MCP transport.
- Supporting multiple knowledge backends.
- Hiding the agent process behind a polished product UX.

## Review Check

This spec intentionally focuses on one implementable project: a local TypeScript chat data bot with visible agent loop behavior and direct RAGFlow MCP integration. It does not include unrelated product features, cloud deployment, or advanced multi-agent work.
