# Claude Code Design Logic Learning Plan

## Purpose

This study plan helps the reader systematically understand the design logic described in this repository's `README.md`.

The goal is broader than reading the document once. The reader wants to build three abilities:

- Understand the product and architecture logic behind a CLI agent such as Claude Code.
- Translate the design into an implementable minimal CLI agent framework.
- Evaluate the security boundaries, risks, and misuse cases introduced by each layer.

The repository currently contains a single analysis document, not the full source tree. The learning process therefore treats the README as an architecture index and separates verified claims, author inference, and items that require external validation.

## Recommended Approach

Use a trunk-first study path with implementation and security checks at every stage.

The recommended path is:

1. Understand the information boundary and source-map leak mechanism.
2. Build the main CLI agent architecture model.
3. Study the tool and permission layer as the core system design.
4. Study multi-agent, memory, proactive, and remote-planning extensions.
5. Convert the understanding into a minimal CLI agent blueprint and security checklist.

This path avoids starting with attention-grabbing hidden features such as BUDDY or KAIROS. Those features are useful later as examples of extension mechanisms, but they are not the architectural foundation.

## Learning Architecture

### Phase 1: Information Boundary and Credibility

Read:

- `How Did Claude Code's Source Code Leak from npm?`
- `What Is Inside the Leaked Claude Code Codebase?`
- `FAQ`
- `References & Further Reading`
- `Disclaimer`

Focus:

- How source maps expose `sourcesContent`.
- What the README directly claims.
- Which claims are facts, inferred architecture, or unverifiable without source access.
- How much confidence to assign to each section.

Output:

- A three-column note: `facts / inferences / needs verification`.
- A short description of what this repository can and cannot prove.

Checkpoint:

- The reader can explain why a sourcemap leak can expose original TypeScript source.
- The reader can avoid treating every README statement as a verified source fact.

### Phase 2: CLI Agent Main Architecture

Read:

- Runtime and package details in `What Is Inside the Leaked Claude Code Codebase?`
- `Hidden Beta Headers and Unreleased API Features`
- `Feature Gating — Internal vs External Builds`
- Relevant parts of `Other Notable Findings`, especially API headers and client attestation.

Focus:

- How a terminal-first AI product is likely structured around an entry point, terminal UI, session state, context construction, model requests, and response rendering.
- How Bun, TypeScript, React, and Ink fit into a CLI application.
- How feature flags split internal and external behavior.
- How API headers, workload metadata, and beta negotiation shape runtime behavior.

Output:

- A main-chain diagram: `user input -> terminal UI -> state/session -> prompt/context -> model request -> response rendering`.
- A glossary of runtime terms: Bun, Ink, feature gate, beta header, workload, client attestation.

Checkpoint:

- The reader can describe the main request loop of a CLI agent without discussing advanced agent modes.
- The reader can identify which parts belong to local runtime behavior and which parts depend on remote API behavior.

### Phase 3: Tool and Permission Core

Read:

- `Claude Code's 40+ Tool Registry`
- `The Permission and Security System`
- `The Cyber Risk Instruction`
- Security-related items from `Other Notable Findings`

Focus:

- Tool registry design: how tools are named, described, filtered, and exposed to the model.
- Permission modes and the tradeoff between interactivity, automation, and risk.
- Risk classification for tool actions.
- Protected files, path traversal prevention, and permission explanation.
- Why this layer is the core of a CLI agent rather than a secondary feature.

Output:

- A tool interface model describing each tool's name, schema, risk profile, permission requirement, and execution result.
- A permission decision flow: `tool request -> policy lookup -> risk classification -> user or auto approval -> execution -> result`.
- A risk table for shell, file edit, web access, MCP, and agent-spawning tools.

Checkpoint:

- The reader can explain how tool execution differs from normal model text generation.
- The reader can propose a minimal permission gate for a self-built CLI agent.
- The reader can identify at least three concrete risks in automatic tool execution.

### Phase 4: Agent, Memory, and Proactive Extensions

Read:

- `Multi-Agent Orchestration — Coordinator Mode`
- `The Dream System`
- `KAIROS — Claude Code's Unreleased Always-On Mode`
- `ULTRAPLAN — 30-Minute Remote Planning Sessions`
- `BUDDY` only as an example of feature-gated product extension, not as a core architecture section.

Focus:

- Difference between child agents, worker agents, background memory consolidation, proactive assistants, and remote planning sessions.
- Coordination patterns: research, synthesis, implementation, verification.
- How durable scratchpads or memory files change the agent's behavior across time.
- Why proactive or always-on behavior raises different safety and UX requirements than request-response chat.

Output:

- A boundary comparison table for synchronous agent, subagent, multi-agent coordinator, background memory task, proactive assistant, and remote planning session.
- A memory lifecycle diagram covering gather, consolidate, prune, and index.
- A list of failure modes for multi-agent systems: duplicated work, conflicting edits, stale memory, runaway automation, and unclear ownership.

Checkpoint:

- The reader can explain why advanced modes should be studied after the tool and permission layer.
- The reader can decide which extension patterns belong in an MVP and which should remain future work.

### Phase 5: Minimal CLI Agent Blueprint

Synthesize:

- The main architecture model from Phase 2.
- The tool and permission model from Phase 3.
- The extension boundaries from Phase 4.
- The security constraints from all phases.

Design the minimum viable system:

- CLI shell or terminal UI.
- Model request loop.
- Session and transcript store.
- Tool registry.
- Permission gate.
- Tool execution adapters.
- Result feedback loop.
- Optional subagent runner after the main loop works.

Output:

- `architecture-map.md`
- `tool-permission-model.md`
- `agent-memory-patterns.md`
- `mvp-cli-agent-blueprint.md`
- `security-checklist.md`

Checkpoint:

- The reader can explain the full path from user input to model response, tool selection, permission approval, tool execution, and result return.
- The reader can write a small implementation plan for a CLI agent without copying Claude Code-specific internals.
- The reader can state the main safety boundaries required before enabling automatic file, shell, or network actions.

## Reusable Analysis Questions

For every phase, answer the same four questions:

1. What user or product problem does this module solve?
2. What are its inputs, outputs, and dependencies?
3. What is the smallest version worth implementing in an independent project?
4. What security or misuse risks does this module introduce?

## Completion Standard

The learning plan is complete when the reader can, without looking at the README, explain the full CLI agent loop:

`user input -> context construction -> model request -> tool selection -> permission approval -> tool execution -> result feedback -> final response`

The reader should also be able to produce a minimal implementation blueprint and a security checklist for a self-built CLI agent.

## Scope Boundaries

This study plan does not attempt to recreate proprietary source code. It focuses on architecture reasoning, general design patterns, implementation abstraction, and security analysis.

It also does not treat unreleased feature descriptions as confirmed product behavior unless they can be independently verified. Such descriptions are useful as design prompts, but they should be labeled as claims from the README.
