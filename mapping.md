# OCF Platform Mapping Guide

**Version 0.1.0 - Draft**

This document specifies how to convert between OCF and AI platform formats, plus the `strip_to_wire` operation that prepares OCF data for direct API submission.

---

## Core principle

OCF separates two concerns:

1. **Wire-compatible message body** - `messages[].message` is strict OpenAI Messages format. After `strip_to_wire()` (which translates extension content blocks), it can be passed directly to `chat.completions.create()`.
2. **Operational envelope** - IDs, timestamps, model attribution, token usage, branching, resource references, annotations. These live *outside* the inner `message` block.

This separation means: extending OCF doesn't break OpenAI compat. Platform-specific data lives in `meta` fields or in clearly-marked extension content blocks (`resource_ref`, `thinking`).

---

## strip_to_wire

`strip_to_wire(ocf, target="openai")` returns a `messages[]` array suitable for direct API submission to the target provider.

The conversion:

1. **Drops envelope fields**: `id`, `parent_id`, `created_at`, `model`, `duration_ms`, `usage`, `annotations`, `meta`.
2. **Unwraps the `message` body**: effectively `[m["message"] for m in ocf.messages]`.
3. **Translates extension content blocks**:
   - `{"type": "resource_ref", "resource_id": "X"}` → provider-specific reference (see table below).
   - `{"type": "thinking", "thinking": "..."}` → drop for OpenAI; preserve as `{"type": "thinking", ...}` for Anthropic; drop for Gemini.
4. **Optionally regenerates IDs** for `tool_calls` if the target requires fresh ones.

### Branching: which thread does strip_to_wire emit?

When a conversation contains branches (multiple messages with the same `parent_id`, or non-null `parent_id` values), `strip_to_wire` cannot blindly unwrap all messages - that would intermix branches and produce an invalid thread. The selection rule:

1. **Caller-provided** - if `strip_to_wire(ocf, leaf_id="msg_X")` is called with an explicit leaf, start from that message.
2. **Document-declared** - else if `conversation.active_message_id` is set, start from that message.
3. **Fallback** - else, start from the last message in `messages[]` array order.

From the start message, walk back through ancestors and build the path. At each step, the next ancestor depends on the current message's `parent_id`:

- **`parent_id` is non-null** - the next ancestor is **the message with that ID** (explicit branch parent).
- **`parent_id` is null** - the next ancestor is **the previous message in `messages[]` array order**. This preserves the documented `parent_id: null` semantics ("follows previous message in array").
- **The current message is the first in `messages[]`** - stop. No predecessor exists.

Reverse the collected path for root-to-leaf order. Only messages on that walk are emitted.

For purely linear conversations (`parent_id` null on every message), the rule reduces to emitting `messages[]` in array order - every message's "parent" is its array predecessor, and the walk traverses the full array.

For mixed branching (some messages have explicit `parent_id`, others null), the walk correctly preserves linear context **before** the branch point. Example: a system message at the start with `parent_id: null` will be picked up via array-order chaining when the branch tip's explicit `parent_id` chain eventually lands on a message whose `parent_id: null` resolves to that system message.

Implementations MUST detect cycles in `parent_id` chains (a malformed document) and reject the document or abort the conversion. A cycle is any path that visits the same message ID twice. Implementations MUST also detect when an explicit `parent_id` references a message that does not exist or appears later in array order - both indicate a malformed document.

### Per-target code block translation

The `code` content block is an OCF extension that preserves structured code with `language` metadata. When converting to a target without a structured code type, render as a Markdown fenced code block with the language as the info-string.

| Target                       | `code` block becomes                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------ |
| OpenAI Chat Completions      | `text` content part with content `"```<language>\n<code>\n```"` (Markdown fence) |
| Anthropic Messages           | `text` content part with content `"```<language>\n<code>\n```"` |
| Google Gemini                | `parts: [{text: "```<language>\n<code>\n```"}]`              |
| Cursor / IDE destinations    | Native structured `{lang, code}` (preserved verbatim - round-trip lossless)                |

When `language` is null, emit fence with no info-string: `"```\n<code>\n```"`. When `filename` is set, IDE destinations MAY use it directly; markdown destinations MAY append it as a comment.

### Tool-call argument format conventions

OpenAI Chat Completions stores `tool_calls[].function.arguments` as a **JSON-encoded string**. Anthropic's `tool_use.input` and Codex/Gemini `function_call.args` use **structured objects**. OCF follows the OpenAI wire convention: `tool_calls[].function.arguments` is **always a JSON string**.

Conversion rules:

| Source field                                  | OCF storage (`function.arguments`) | strip_to_wire output                    |
| --------------------------------------------- | ---------------------------------- | --------------------------------------- |
| Anthropic `tool_use.input` (object)           | `JSON.stringify(input)`            | OpenAI: same string. Anthropic: `JSON.parse(arguments)` back to object. |
| OpenAI `tool_call.function.arguments` (string)| string verbatim                    | OpenAI: same string. Anthropic: `JSON.parse(arguments)`. |
| Codex `tool_call.arguments` (object)          | `JSON.stringify(arguments)`        | Codex: `JSON.parse(arguments)` back to object. OpenAI: same string. |
| Gemini `function_call.args` (object)          | `JSON.stringify(args)`             | Gemini: `JSON.parse(arguments)` back to object. OpenAI: same string. |

Implementations MUST validate that `arguments` is parseable JSON before passing to a target that expects an object. Malformed JSON in `arguments` indicates a bad converter - the target SHOULD reject the message rather than silently emit a corrupted call.

### Per-target resource_ref translation

| Target                       | `resource_ref` becomes                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------- |
| OpenAI Chat Completions      | Image: `{"type": "image_url", "image_url": {"url": "data:..."}}` - File: `{"type": "file", "file": {"file_id": "..."}}` (after Files API upload) or `{"type": "file", "file": {"filename": "...", "file_data": "data:..."}}` (inline). Note: `input_file` is **Responses API** terminology, not Chat Completions. |
| Anthropic Messages           | Image: `{"type": "image", "source": {...}}` - Document: `{"type": "document", "source": {...}}` |
| Google Gemini                | Small: `{"inline_data": {"mime_type": "...", "data": "..."}}` - Large: File API upload + `{"file_data": {"file_uri": "..."}}` |

For `external_link` resources, the URL is referenced inline if the provider accepts URL-based references; otherwise the resource is dropped or fetched and inlined.

---

## Platform → OCF mappings

### OpenAI Chat Completions API → OCF

```
OpenAI                              OCF
─────────────────────────────────────────────────────
{role, content}                  → messages[].message (1:1)
tool_calls                       → messages[].message.tool_calls (1:1)
tool_call_id                     → messages[].message.tool_call_id (1:1)
name                             → messages[].message.name (1:1)

usage.prompt_tokens              → messages[].usage.input
usage.completion_tokens          → messages[].usage.output
model                            → messages[].model
created (unix seconds)           → messages[].created_at (convert to ISO 8601)
```

**Files**: OpenAI Files API uploads → `resources[]` with `kind: "user_file"` (uploaded by user) or `kind: "tool_input"`/`tool_output"` (used by code interpreter, etc.). Original `file_id` goes in `meta.openai_file_id`.

**OpenAI Responses API (v0.2 roadmap)**: OpenAI's newer Responses API uses a different content shape (`input_text`, `input_image`, `input_file` for inputs; `output_text` with annotations for outputs) and slightly different tool-call conventions. v0.1.0 does **not** specify a Responses API mapping. Converters targeting Responses API SHOULD round-trip via Chat Completions semantics for now. A formal mapping is planned for OCF v0.2; semantically the data is the same (file inputs, text/structured outputs, tool calls) and existing OCF resources/messages cover it - only the wire shapes differ.

---

### Anthropic Messages API → OCF

```
Anthropic                           OCF
─────────────────────────────────────────────────────
{role: "user"|"assistant", content}
                                 → messages[].message (translate content blocks)
system (top-level parameter)     → first message with role: "system"

content: [{type: "text", text}]
                                 → message.content: [{type: "text", text}]   (1:1)

content: [{type: "thinking", thinking}]
                                 → message.content: [{type: "thinking", thinking}]
                                   (OCF extension; preserved)

content: [{type: "tool_use", id, name, input}]
                                 → message.tool_calls: [{
                                     id, type: "function",
                                     function: {name, arguments: JSON.stringify(input)}
                                   }]
                                   (move from content array to tool_calls sibling)

content: [{type: "tool_result", tool_use_id, content}]
                                 → new message with role: "tool",
                                   tool_call_id: tool_use_id,
                                   content: stringify(result)

content: [{type: "image", source: {type, media_type, data}}]
                                 → message.content: [{type: "resource_ref", resource_id: "res_X"}]
                                   + new resources[] entry of kind: "inline_attachment"

usage.input_tokens               → messages[].usage.input
usage.output_tokens              → messages[].usage.output
```

**Note**: Anthropic puts tool calls *inside content*, OpenAI puts them *as siblings of content*. OCF follows OpenAI. The conversion is mechanical but not 1:1 in shape.

**Claude Artifacts** (via UI or future API): become `resources[]` with `kind: "assistant_artifact"`. The artifact content is stored in the resource. The message text gets a `resource_ref` block where the artifact appears in the conversation flow.

---

### Google Gemini API → OCF

```
Gemini                              OCF
─────────────────────────────────────────────────────
role: "model"                    → message.role: "assistant"
role: "user"                     → message.role: "user"
system_instruction (top-level)   → first message with role: "system"

parts: [{text}]                  → message.content: [{type: "text", text}]
parts: [{inline_data: {mime_type, data}}]
                                 → message.content: [{type: "resource_ref", ...}]
                                   + resources[] entry, source.type: "inline"
parts: [{file_data: {file_uri, mime_type}}]
                                 → message.content: [{type: "resource_ref", ...}]
                                   + resources[] entry, source.type: "url"
parts: [{function_call: {name, args}}]
                                 → message.tool_calls: [{
                                     id: generated, type: "function",
                                     function: {name, arguments: JSON.stringify(args)}
                                   }]
parts: [{function_response: {name, response}}]
                                 → new message with role: "tool",
                                   tool_call_id: correlated
```

**Note**: Gemini doesn't natively assign IDs to function calls. The converter must generate IDs and correlate `function_call` / `function_response` pairs by name and order. Store the original Gemini-correlation hint in `meta.gemini_correlation`.

---

### ChatGPT Web Export (ZIP) → OCF

ChatGPT exports a tree (`mapping` with `parent`/`children`). Linearize by depth-first traversal, taking the latest child at each branch point. Branches can be preserved via `parent_id` on the divergent messages.

```
ChatGPT Export                      OCF
─────────────────────────────────────────────────────
conversations[].id                → conversation.source.original_id
conversations[].title             → conversation.title
conversations[].create_time       → conversation.created_at  (Unix → ISO 8601)
conversations[].mapping[node].message
                                  → messages[].message (translate)
mapping[node].id                  → messages[].id
mapping[node].message.author.role → message.role  (normalize to system/user/assistant/tool)
mapping[node].message.content.parts
                                  → message.content (parse parts)
mapping[node].message.create_time → messages[].created_at
mapping[node].parent              → messages[].parent_id  (when divergent from prior order)
mapping[node].message.metadata.model_slug
                                  → messages[].model

Attached files (multimodal)       → resources[] with kind: "user_file"
DALL-E generated images           → resources[] with kind: "assistant_artifact"
Code Interpreter outputs          → resources[] with kind: "tool_output"
```

**Edge case**: ChatGPT's edited messages create branches (a node with multiple children, where the latest child is the active edit). A linear converter follows the latest-child path; a branch-aware converter emits all children with `parent_id` pointing back to the divergence point.

---

### Claude Web Export (JSON) → OCF

Claude's data export delivers per-conversation JSON files.

```
Claude Export                       OCF
─────────────────────────────────────────────────────
uuid                              → conversation.source.original_id
name                              → conversation.title
created_at                        → conversation.created_at
chat_messages[].uuid              → messages[].id
chat_messages[].sender            → message.role
                                    ("human" → "user", "assistant" → "assistant")
chat_messages[].text              → message.content (string)
chat_messages[].created_at        → messages[].created_at
chat_messages[].attachments[]     → resources[] with kind: "user_file"
chat_messages[].files[]           → resources[] with kind: "user_file"
```

**Artifacts** (when present in the export): become `resources[]` with `kind: "assistant_artifact"`. Claude's `artifact_type` and `title` go to the resource's `media_type`/`title`, with the original type stored in `meta.original_artifact_type`.

---

### Browser Extension Markdown Export → OCF

Lossy. Heuristic only.

```
Markdown                            OCF
─────────────────────────────────────────────────────
"## User" / "## Assistant" headers
                                  → message.role (parse headers)
Text between headers               → message.content (string)
Inline code blocks (```)           → keep in message.content as Markdown
Images (![alt](path))              → resources[] entry + resource_ref content block
```

Synthetic IDs are generated. `created_at` is null. `model` is null. The conversion is one-way; round-trips are not supported.

---

### Agentic coding sessions (Claude Code, Cursor, Aider, ...) → OCF

Agentic IDE sessions consist primarily of structured tool invocations: file edits, shell commands, code execution. OCF represents these natively via `tool_calls[]` and `role: "tool"` messages - no custom content block types are required. The information in source-format renderings (Claude Code's `MessagePart.kind`, Cursor's composer state) lives in `tool_call.function.arguments` (JSON-encoded) and `tool_result.content`.

For the canonical tool names and argument schemas this mapping uses, see [`tool-conventions.md`](tool-conventions.md). The canonical set is aligned with the [Model Context Standard (MCS)](https://github.com/modelcontextstandard).

```
Source kind                         OCF
─────────────────────────────────────────────────────
file_edit (with diff)            → tool_call named `edit_file` (or `apply_diff`)
                                   + tool_result message
terminal (command + output)      → tool_call named `bash`
                                   + tool_result with JSON-encoded
                                     {stdout, stderr, exit_code}
code execution                   → tool_call named `code_execution`
                                   + tool_result; artifacts → resources[] (tool_output)
file read                        → tool_call named `read_file`
                                   + tool_result with file content
file write                       → tool_call named `write_file`
                                   + tool_result with success marker
reasoning / thinking             → content block {type: "thinking", thinking: "..."}
attachment / reference           → resource_ref content block + resources[] entry
system instruction               → message with role: "system"
non-canonical tool               → tool_call with arbitrary name
                                   + tool_schema in conversation.meta.tool_schemas
```

**Rendering metadata** richer than OCF's structure (line numbers, syntax-highlighting hints, collapse state, UI annotations) belongs in `messages[].meta.<source>_render` - namespaced by source identifier (e.g., `meta.claude_code_render`, `meta.cursor_render`). The OCF core stays wire-clean while preserving source fidelity.

**Raw source preservation**: Always preserve the original session as a `kind: "raw_native"` resource with `meta.raw_format` set (e.g., `"claude-code-session-jsonl"`, `"cursor-composerdata-v3"`). Future converter versions can re-project from the raw bytes when mappings improve. See [Raw native preservation](#raw-native-preservation).

---

## File layouts

### Single-file snapshot (`.ocf.json`)

A complete conversation as a single JSON object. Resources may use `inline` source type to embed file content as base64 - useful when the entire session needs to fit in a single file.

### Bundle with external resources (`.ocf.zip`)

```
conv_<id>.ocf.zip
├── conversation.ocf.json
└── resources/
    ├── inventory.csv
    └── chart_output.png
```

Resources reference files via `source: {"type": "file", "path": "resources/inventory.csv"}`. Implementations MAY use a content-addressed layout (`resources/sha256/<digest>`) and adjust `source.path` accordingly.

#### Path validation (security)

Implementations extracting or reading a `.ocf.zip` bundle MUST validate every `source.path` (where `source.type` is `file`) before opening the file. Path-traversal protection is **mandatory** - a malicious bundle with `../../etc/passwd` or absolute paths could overwrite arbitrary files on the receiver's system.

Validation rules - every `source.path` MUST satisfy:

1. **POSIX separators only** - paths use `/`. Backslashes (`\`), mixed separators, or other separators MUST be rejected.
2. **Relative only** - paths MUST NOT begin with `/` (no absolute Unix paths).
3. **No drive letters / UNC** - paths MUST NOT contain Windows drive prefixes (`C:`, `D:`, etc.) or UNC prefixes (`\\server\share`).
4. **No parent traversal** - paths MUST NOT contain any `..` segment. Reject literally; do not interpret-and-resolve before checking.
5. **No null bytes / control chars** - paths MUST NOT contain ` ` or other C0 control characters.
6. **NFC-normalized** - paths SHOULD be in Unicode NFC form (consistent with the canonical-JSON contract above).
7. **Stays within bundle root** - after resolving `<bundle_root>/<path>`, the resolved real path MUST be a descendant of `<bundle_root>`. (Defense-in-depth check after rule 4.)

Implementations MUST reject the entire bundle if any `source.path` fails these checks. Do NOT silently skip the offending resource - partial extraction of a malicious bundle is itself a vulnerability.

The recommended layout puts files under `resources/` or `resources/raw/`. Implementations MAY enforce these prefixes as a hard requirement.

### Append-only event log (`.ocf.jsonl`)

One JSON object per line, each carrying a `type` discriminator. Every event MUST include `seq` (writer-monotonic sequence number) and SHOULD include `event_id` (stable identity for dedup). Optionally `occurred_at` records when the event happened in the source system (distinct from write order).

```jsonl
{"event_id":"evt_a1","seq":1,"type":"session","ocf_version":"0.1.0","conversation":{...}}
{"event_id":"evt_a2","seq":2,"type":"resource","id":"res_001","kind":"user_file",...}
{"event_id":"evt_a3","seq":3,"type":"message","id":"msg_001","message":{...}}
{"event_id":"evt_a4","seq":4,"type":"resource","id":"res_002","kind":"assistant_artifact","created_by":"msg_002",...}
{"event_id":"evt_a5","seq":5,"type":"message","id":"msg_002","message":{...}}
{"event_id":"evt_a6","seq":6,"type":"annotation","message_id":"msg_001","annotations":{"relevance":"high"}}
```

#### Event field requirements

- **`seq`** - **MUST** be present. Writer-monotonic integer, starting at 1, incrementing by 1 per event. **Single-writer assumption** for v0.1.0: a given `.ocf.jsonl` file MUST be written by one logical writer at a time. Multi-writer concurrent appending is out of scope (v0.2+ may add a writer_id field for multi-writer scenarios).
- **`event_id`** - **SHOULD** be present. Stable, globally-unique event identifier (UUID, KSUID, or similar). Enables dedup when a reader resumes from a checkpoint and may re-read recent events. Without `event_id`, dedup falls back to `(type, primary_id)` heuristics where `primary_id` is the most-specific ID in the event (`id` / `message_id` / etc.).
- **`occurred_at`** - **MAY** be present. ISO 8601 timestamp (UTC `Z`-suffixed) of when the event happened in the source system. Distinct from write order: a converter post-processing a batch may emit events with arbitrary `occurred_at` values while `seq` follows write order. When absent, readers SHOULD fall back to per-entity timestamps (`messages[].created_at`, etc.).
- **`type`** - **MUST** be present. Discriminator for the event kind (see below).

Event types:

- **`session`** - emitted exactly once, must be the first event. Carries `ocf_version` and the `conversation` block.
- **`resource`** - emitted when a resource is added. SHOULD be emitted before any message that references it, so streaming readers can resolve `resource_ref` blocks immediately.
- **`message`** - emitted when a message is appended. Same fields as a `message_envelope` in the snapshot format.
- **`annotation`** - optional. Updates `annotations` on a previously-emitted message identified by `message_id`.

A reader reconstructs the snapshot by replaying events in order:

- `session` populates the document root.
- `resource` events append to `resources[]`.
- `message` events append to `messages[]`.
- `annotation` events update the annotations on the referenced message.

Events are append-only. Mutating prior messages or resources is not supported in v0.1.0. (Future versions may add `delete` or `update` event types - but explicit re-emission is preferred.)

#### Crash safety: partial-tail-safe readers

Writers can be killed mid-line (process crash, kill -9, OOM, container restart). Readers MUST handle this gracefully:

1. **Empty file** → no events.
2. **File ends with `\n`** → all lines are complete; parse all.
3. **File does NOT end with `\n`** → the last line is partial; **DROP it**. The next read after the writer recovers will see the complete line.
4. **A non-final line that fails JSON parse** SHOULD be skipped with a logged warning. A reader MUST NOT abort the entire file on a single corrupt line.
5. **A non-final event with unknown `type`** SHOULD be ignored (forward compatibility). A reader MAY warn but MUST NOT abort.

Writers MUST follow these rules to make readers' job possible:

- Each line MUST be terminated with `\n`. A line is durable only after `\n` is flushed.
- Writers SHOULD prefer atomic single-syscall writes per line (most filesystems guarantee atomicity for writes ≤ PIPE_BUF or ≤ 4 KiB; larger lines may tear).
- Writers MUST NOT mutate previously-written lines. Append-only.
- Writers SHOULD `fsync()` after critical events (e.g., user message committed) if durability matters; otherwise OS-level buffering applies.

---

## Projects and bulk export layout

ChatGPT Projects, Claude Projects, and similar workspace concepts organize conversations into named groups. OCF captures this via the optional `conversation.project` block:

```json
"conversation": {
  "id": "conv_001",
  "title": "Inventory parser",
  "created_at": "2026-04-25T14:30:00Z",
  "project": {
    "id": "proj_xyz",
    "name": "Cephix Development",
    "platform_id": "claude_proj_abc123",
    "description": "Inventory tooling and CSV parsers"
  }
}
```

The `project` block is per-conversation and is the **canonical** assignment. Importers reconstructing project structure on a target platform MUST read `conversation.project.id` from each conversation; directory layout is a human-readable convention only and MUST NOT be the source of truth.

### Bulk export layout (informative, not normative)

When exporting many conversations from a single source platform, the recommended directory layout:

```
account_export.ocf.zip
├── projects/
│   ├── proj_xyz/
│   │   ├── conv_001.ocf.json
│   │   └── conv_002.ocf.json
│   └── proj_abc/
│       └── conv_003.ocf.json
└── unfiled/
    ├── conv_004.ocf.json
    └── conv_005.ocf.json
```

This layout helps humans navigate the archive but the canonical project assignment lives in `conversation.project`. A flat layout with all conversations at the bundle root is equally valid.

### Project-level OCF documents (v0.2 roadmap)

A project itself has metadata that is currently outside OCF v0.1.0 scope:

- Project name, description, custom instructions
- Project-shared knowledge files / system prompts
- References to member conversations
- Project-level tool sets

OCF v0.2 may introduce a `project.ocf.json` document type alongside conversation documents to cover this. For v0.1.0, store such metadata in `conversation.project.description` and `conversation.meta` if needed for round-trip fidelity. Project-shared files attached to multiple conversations are duplicated into each conversation's `resources[]` for now (lossy on storage but lossless on semantics).

---

## Hashing and canonical JSON

Resources have a `sha256` field. Bundle integrity, content-addressed storage, and cross-implementation comparison all depend on **bit-stable serialization**: two implementations must produce identical bytes for identical logical data, or hashes diverge silently.

OCF defines a strict canonical form. Implementations that compute or compare hashes MUST use it.

### Canonical JSON contract

1. **Encoding**: UTF-8, no BOM, no trailing newline.
2. **Key order**: object keys sorted byte-wise lexicographic (codepoint order over UTF-8 bytes).
3. **Whitespace**: no spaces between tokens. No indentation. No line breaks inside objects/arrays.
4. **Strings**: escape only the JSON-required characters (`"`, `\`, control chars ` `-``). Non-ASCII Unicode characters MUST be emitted literally (not `\uXXXX`-escaped). Strings MUST be in **Unicode NFC** (Normalization Form C) before serialization - otherwise `"é"` (precomposed U+00E9) and `"é"` (decomposed U+0065 + U+0301) hash differently despite being the same logical string.
5. **Numbers**:
   - **Integers** serialize without decimal point: `42`, not `42.0`.
   - **Floats** use the shortest round-trip-safe representation. Python: `repr()` on `float`. Other languages: equivalent shortest-form algorithms (e.g., Ryu, Grisu3).
   - **NaN and Infinity MUST be rejected**. The serializer MUST raise an error rather than emit `null` (this is the orjson default and silently breaks hashes - explicit rejection is required).
   - Integers that overflow IEEE 754 doubles (`> 2^53`) SHOULD be rejected unless explicitly handled by both writer and reader.
6. **Timestamps**: ISO 8601 in UTC with `Z` suffix. No offset notation (`+00:00` is forbidden - must be normalized to `Z`). Sub-second precision is preserved verbatim from the source.
7. **Booleans and null**: `true`, `false`, `null` exactly.

### Computing a resource hash

For `resources[].sha256`, hash the raw file bytes - not a JSON serialization of the resource entry. Direct file-content hashing.

For inline resources (`source.type: "inline"`), hash the **decoded** bytes (after base64-decoding `source.data`), not the base64 string.

### Computing a redaction policy hash

For `conversation.redaction.policy_hash`:

```
policy_hash = sha256_hex(canonical_json({
  "mode": <mode>,
  "patterns": <patterns sorted byte-wise lexicographic>
}))
```

Implementations sharing a redaction policy MUST agree on the pattern set and ordering, or `policy_hash` will diverge.

### Compatibility note

The canonical form above overlaps substantially with [RFC 8785 (JSON Canonicalization Scheme)](https://datatracker.ietf.org/doc/html/rfc8785). Implementations that already use RFC 8785 will produce compatible output for most documents, but RFC 8785 follows ECMAScript number serialization rules (which differ subtly from Python `repr` for some edge cases) and does not mandate NFC. OCF's contract is the authoritative reference; RFC 8785 is informative.

---

## Redaction

OCF documents may contain secrets: API keys pasted in chat, `.env` file contents uploaded as resources, credentials shown in tool output. Sharing a bundle without sanitization is a real-world security incident waiting to happen.

OCF defines a **redaction layer** that records what redaction (if any) has been applied to a document, so receivers can verify safety before processing.

### conversation.redaction

```json
"redaction": {
  "mode": "off | flag_only | mask | drop",
  "policy_hash": "<sha256 hex over canonical(mode, sorted-patterns)>",
  "applied_at": "<ISO 8601 timestamp>"
}
```

**Modes**:

- **`off`** - no redaction has been applied. The document MAY contain secrets. Default for raw exports.
- **`flag_only`** - sensitive content has been *identified* (via `sensitive: true` flags on content blocks, messages, or resources) but content is **unchanged**. Useful for review workflows.
- **`mask`** - sensitive content has been *replaced* with masking markers (e.g. `[REDACTED]`, `AKIA****`, `<email>`). Content blocks marked `sensitive: true` indicate where masking was applied.
- **`drop`** - sensitive content blocks, messages, or resources have been *removed entirely*. The document is shorter than the original. Removed entries leave no trace except possibly an envelope-level `meta.redacted: true`.

**`policy_hash`** lets a receiver verify the bundle was processed with a known policy. The hash is computed as:

```
policy_hash = sha256_hex(canonical_json({
  "mode": <mode>,
  "patterns": [<patterns sorted byte-wise lexicographic>]
}))
```

A receiver with the same policy can recompute the hash and confirm match. Receivers MAY reject bundles with `mode: "off"` or unknown `policy_hash` from untrusted senders.

### sensitive flag

The `sensitive: boolean` field appears at three levels:

- **`content_block.sensitive`** - fine-grained: this specific block (text, image, resource_ref, thinking) contains sensitive data.
- **`message_envelope.sensitive`** - coarse: the entire message contains sensitive data. Useful when `message.content` is a string and per-block flagging is impossible.
- **`resource.sensitive`** - the resource (file, artifact) contains sensitive data.

These flags are advisory. They mark content for redaction tooling but do not enforce it. Whether the content is actually masked or dropped depends on `conversation.redaction.mode`.

### Workflow

A typical redaction pass:

1. Reader walks the OCF document with a pattern matcher.
2. For each match, set `sensitive: true` on the relevant block / message / resource.
3. Apply transformation according to chosen mode (`flag_only` / `mask` / `drop`).
4. Set `conversation.redaction = {mode, policy_hash, applied_at}`.
5. (For bundles) re-compute `resources[].sha256` for any resource whose content was modified by masking.

---

## Raw native preservation

When converting from a platform-specific format, the conversion is often lossy or improvable. A converter from Cursor's `composerData` blob today might miss edge cases that a v0.4 converter handles correctly. To enable re-projection without re-fetching from the source platform, OCF supports preserving the original bytes via the `raw_native` resource kind.

### When to use raw_native

- Converting from a fast-evolving format (Cursor composerData, Claude Code session jsonl, ChatGPT export internal structure) where mapping fidelity is improving over time.
- Source format is undocumented or partially documented; preserving the bytes is cheap insurance.
- Compliance requires retaining original data alongside derived projections.

### Bundle convention

Raw native resources go under `resources/raw/<sha256>` in bundles, separate from the `resources/` directory used for user/assistant artifacts:

```
conv_<id>.ocf.zip
├── conversation.ocf.json
├── resources/
│   ├── inventory.csv
│   └── chart_output.png
└── resources/raw/
    ├── e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    └── 7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730
```

The schema entry:

```json
{
  "id": "res_raw_001",
  "kind": "raw_native",
  "media_type": "application/x-ndjson",
  "byte_size": 487521,
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "source": {"type": "file", "path": "resources/raw/e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"},
  "meta": {
    "raw_format": "claude-code-session-jsonl",
    "raw_format_version": "1.x",
    "captured_at": "2026-04-26T14:00:00Z"
  }
}
```

`meta.raw_format` is critical: it lets future converter tooling find all `raw_native` resources of a given format. Suggested format identifiers (informative, not exhaustive):

- `chatgpt-export-conversations-v1` - ChatGPT data export `conversations.json`
- `claude-export-v1` - Claude data export per-conversation JSON
- `claude-code-session-jsonl` - Claude Code session log
- `cursor-composerdata-v3` - Cursor composer state blob (subscript by major version)
- `gemini-takeout-v1` - Google Takeout Gemini activity export

### Re-projection workflow

A future converter improvement:

1. Tool scans bundle for `resources[]` where `kind == "raw_native"` and `meta.raw_format` matches a supported format/version.
2. Tool re-runs the (improved) mapping over the raw bytes.
3. Tool emits a new OCF document with `produced_by.version` reflecting the new converter version.
4. The original document and new document can be diffed; the new mapping's improvements are auditable.

---

## Validation

An OCF document is valid when:

1. `ocf_version` is present and recognized by the validator.
2. `conversation.id` and `conversation.created_at` are present.
3. Every message has `id` and `message`.
4. Every `message.role` is one of `system`, `developer`, `user`, `assistant`, `tool`.
5. Every non-null `parent_id` references an existing message ID.
6. Every `tool_call_id` (in role: "tool" messages) has a corresponding `tool_calls[].id` in some prior assistant message in the same branch.
7. Every `resource_ref.resource_id` (in any content block) points to an existing `resources[].id`.
8. Every `resources[].source` matches one of the three sub-schemas (`file` / `inline` / `url`).
9. Every `resources[].kind` is one of the seven defined values.
10. When `resources[].kind == "raw_native"`, `meta.raw_format` SHOULD be set. Validators MAY warn (not fail) if absent - re-projection tooling cannot locate untagged raw resources.
11. When `conversation.redaction.mode` is set to anything other than `"off"`, `policy_hash` SHOULD be set. Validators MAY warn if absent.
12. SHA-256 digests, when computed for comparison or `policy_hash`, MUST follow the canonical JSON contract in [Hashing and canonical JSON](#hashing-and-canonical-json). Hashes computed with non-canonical serialization are not interoperable.
13. When `conversation.active_message_id` is set, it MUST reference an existing `messages[].id`.
14. When `messages[].toolset_id` is set, it MUST reference an existing `conversation.toolsets[].id`.
15. When `resources[].revises` is set, it MUST reference an existing `resources[].id`. Revision chains MUST be acyclic (A revises B, B revises A is invalid).
16. Every non-null `parent_id` MUST reference a message with a smaller array index (parents come before children in array order). Forward references are invalid.
17. When `messages[].id_origin` or `tool_calls[].id_origin` is set, it MUST be one of `"source"` or `"synthesized"`. Validators MAY warn when synthesized IDs collide with source IDs in the same conversation.

A bundle (`.ocf.zip`) is additionally valid when:

18. `conversation.ocf.json` exists at the bundle root.
19. Every `source.path` (where `type` is `file`) references a file inside the bundle.
20. Every `source.path` passes the path-validation rules (POSIX separators, relative, no `..`, no drive letters, no null bytes - see [Path validation](#path-validation-security)).
21. Where `sha256` is set on a resource, it matches the actual file's SHA-256 digest (over the file bytes, not the JSON entry).
22. `raw_native` resources SHOULD live under `resources/raw/` (convention, not enforced).

An event log (`.ocf.jsonl`) is additionally valid when:

23. The first line is `type: "session"`.
24. Every event has `seq` (positive integer, monotonic from 1).
25. Every `resource_ref.resource_id` references a resource emitted in a *prior* line.
26. Every `annotation.message_id` references a message emitted in a *prior* line.
27. The reader is partial-tail-safe per [Crash safety](#crash-safety-partial-tail-safe-readers): a truncated final line MUST be dropped, not parsed.
