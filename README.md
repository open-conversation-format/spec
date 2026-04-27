# Open Conversation Format (OCF)

> **Version 0.1.0 — Draft.** Pre-1.0. The schema may change incompatibly between minor versions until v1.0.0. Pin to a specific `ocf_version` if you depend on the format.

A portable interchange format for AI conversations. Wraps the OpenAI-compatible Messages schema in a session envelope, adds a resource layer for files and artifacts, and supports both atomic snapshots and append-only event logs.

## Why

ChatGPT, Claude, Gemini, Copilot, Grok — every platform exports conversations in a different shape. None of them export everything: timestamps, tool calls, attached files, generated artifacts, branching from edits. Most third-party exporters drop content or strip metadata.

OCF defines a single format that:

- **Embeds the OpenAI Messages standard** as the message core, so `messages[].message` is wire-compatible with `chat.completions.create()` after `strip_to_wire()`.
- **Adds a session envelope** with operational metadata (IDs, timestamps, model attribution, token usage, branching).
- **Adds a resource layer** for files the user uploaded, files the assistant generated, tool inputs/outputs, and platform-specific artifacts (Claude Artifacts, GPT Canvas).
- **Supports two storage modes**: a single `.ocf.json` snapshot, or an append-only `.ocf.jsonl` event log.
- **Bundles into `.ocf.zip`** when files are external — a complete conversation can be downloaded, archived, and re-uploaded with everything intact.

## Quickstart

A minimal valid OCF document:

```json
{
  "ocf_version": "0.1.0",
  "conversation": {
    "id": "conv_001",
    "created_at": "2026-04-26T12:00:00Z"
  },
  "messages": [
    {
      "id": "msg_001",
      "message": {
        "role": "user",
        "content": "Hello"
      }
    }
  ]
}
```

The inner `message` field is strict OpenAI Messages format. The envelope (`id`, `created_at`, `model`, `usage`, etc.) lives outside it.

See [`examples/`](examples/) for richer cases including resources, tool calls, and event logs.

## File modes

| Mode      | Extension      | Use case                                                   |
| --------- | -------------- | ---------------------------------------------------------- |
| Snapshot  | `.ocf.json`    | Single conversation as one JSON object. Inline resources possible via base64. |
| Event log | `.ocf.jsonl`   | Append-only event stream, one JSON object per line. Streaming/operational. |
| Bundle    | `.ocf.zip`     | A `.ocf.json` plus a `resources/` directory with attached files.            |

## Architecture

```
{
  "ocf_version": "0.1.0",
  "conversation": { ... },         ← session-level metadata
  "resources": [ ... ],            ← files, artifacts, tool I/O (optional)
  "messages": [
    {
      "id": "...",                 ← envelope: OCF-specific operational fields
      "parent_id": null,
      "created_at": "...",
      "model": "...",
      "usage": { ... },
      "annotations": { ... },
      "message": {                 ← strict OpenAI Messages body
        "role": "...",
        "content": "...",
        "tool_calls": [ ... ]
      }
    }
  ]
}
```

The boundary is intentional: `m["message"]` is what goes on the wire to a provider after `strip_to_wire()` translates extension content blocks (`resource_ref`, `thinking`) into provider-specific representations.

## Relationship to other formats and protocols

OCF is a **portable session/replay/archive layer**. It captures *what was said* in a conversation and *what artifacts were attached*, in a format that survives platform changes, enables re-projection when mappings improve, and round-trips through provider APIs after `strip_to_wire`. OCF does **not** define how conversations are produced — that's the job of API protocols and runtime standards.

**OCF embeds or aligns with**:

- **OpenAI Messages format** — embedded directly inside `messages[].message`. No translation required for the core after `strip_to_wire`. The message-object shape tracks [OpenAI's OpenAPI spec](https://github.com/openai/openai-openapi); an OCF conversation is, at its core, a history of these message objects wrapped in the OCF envelope. (Informative reference, not a build dependency — OCF maintains its own JSON Schema for stability across OpenAI's release cadence.)
- **[Model Context Standard (MCS)](https://github.com/modelcontextstandard)** — recommended companion for tool definitions. OCF tool calls follow a universal `name + description + JSON-Schema-parameters` pattern, fully expressible as MCS `Tool` / `ToolParameter` objects. MCS provides this as a **transport-agnostic driver interface** — implementations choose the wire protocol (HTTP, gRPC, AS2, etc.) and the internal schema format (OpenAPI, JSON Schema, proprietary). The persistence layer (OCF) and the runtime layer (MCS) compose without introducing a separate protocol stack — same capabilities as alternative tool protocols, less plumbing. See [tool-conventions.md](tool-conventions.md).
- **OMF ([open-llm-initiative/open-message-format](https://github.com/open-llm-initiative/open-message-format))** — vendor-agnostic message contract. OCF is compatible with the same intent and adds the session envelope and resource layer that OMF leaves out of scope.
- **OpenAI fine-tuning JSONL** — applying `strip_to_wire()` to an OCF document yields `{"messages": [...]}` per conversation, ready for fine-tuning.

**OCF complements (does not compete with)**:

- **MCP (Model Context Protocol)** — runtime tool/resource/prompt protocol between clients and servers. MCP describes *how a session communicates*; OCF describes *what a completed session looks like persisted*.
- **A2A (Agent-to-Agent)** — protocol for agent task coordination and artifact exchange between agents. Concerned with *workflow*; OCF is concerned with *records*.
- **OpenTelemetry GenAI** — observability and tracing for LLM calls. Concerned with *operational telemetry*; OCF is concerned with *user-visible content*.
- **ATIF (Agent Trajectory Interchange Format)** — agent-trajectory logging for evaluation and training. Concerned with *step-level decisions*; OCF is concerned with *the conversation a human or system would replay*.

**Source platform exports** — see [mapping.md](mapping.md) for full conversion specs from ChatGPT, Claude, Gemini, agentic IDE sessions, and browser extensions.

## Spec documents

- [`schema/ocf-v0.1.0.schema.json`](schema/ocf-v0.1.0.schema.json) — JSON Schema (draft-07)
- [`mapping.md`](mapping.md) — platform mappings, `strip_to_wire` definition, hashing, redaction, raw-native preservation
- [`tool-conventions.md`](tool-conventions.md) — canonical tool names and argument schemas, aligned with [MCS](https://github.com/modelcontextstandard)
- [`examples/`](examples/) — reference documents

## License

MIT (placeholder — final license TBD before v1.0).

## Status

| Area                                | Status        |
| ----------------------------------- | ------------- |
| Conversation envelope               | Draft         |
| Message envelope (with `status`)    | Draft         |
| Resources layer (with `revises`)    | Draft         |
| Code content block (`code`)         | Draft         |
| ID provenance (`id_origin`)         | Draft         |
| Role-specific message validation    | Draft         |
| OpenAI Chat Completions mapping     | Draft         |
| Anthropic Messages mapping          | Draft         |
| Gemini mapping                      | Draft         |
| ChatGPT export mapping              | Draft         |
| Claude export mapping               | Draft         |
| Hashing / canonical JSON            | Draft         |
| Redaction layer                     | Draft         |
| `raw_native` re-projection          | Draft         |
| Partial-tail-safe event log         | Draft         |
| Event-log `seq` / `event_id`        | Draft         |
| Tool conventions (MCS-aligned)      | Draft         |
| Agentic IDE session mapping         | Draft         |
| Project organization                | Draft         |
| Toolsets (dynamic tool sets)        | Draft         |
| Validators                          | Not started   |
| Reference converters                | Not started   |
| OpenAI Responses API mapping        | v0.2 roadmap  |
| Project-level OCF documents         | v0.2 roadmap  |
| Multi-writer event log              | v0.2+ roadmap |
