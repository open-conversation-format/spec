# OCF Tool Conventions

**Version 0.1.0 - Draft**

This document defines canonical tool names and argument schemas for OCF tool calls. The conventions enable cross-converter interoperability for agentic IDE sessions (Claude Code, Cursor, Aider, OpenAI Code Interpreter, Codex) where the conversation consists primarily of structured tool invocations.

OCF embeds tool calls in `messages[].message.tool_calls[]` as OpenAI-compatible siblings of `content`. The `function.arguments` field is a JSON-encoded string. Tool results appear as separate messages with `role: "tool"` and a `tool_call_id` referencing the call.

This document defines:

- A set of canonical tool names with parameter schemas (MCS-compatible)
- The recommended companion standard ([MCS](https://github.com/modelcontextstandard)) and why
- The mapping from MCS `Tool` definitions to OCF `tool_call` / `tool_result` wire shape
- How implementations declare and document non-canonical tools

---

## Tool schema conventions (MCS-compatible)

OCF defines a small set of canonical conventions for what a tool definition looks like. These conventions reflect the universal pattern observed across major tool-calling implementations: tools are described by a **name**, an optional **title**, a **description**, and **parameters** with per-parameter JSON Schema. Whether one calls this an OpenAI function tool, an Anthropic tool definition, or an MCS `Tool`, the underlying shape is the same.

OCF is **MCS-compatible**: the canonical tool names below can be expressed directly as [Model Context Standard (MCS)](https://github.com/modelcontextstandard) `Tool` and `ToolParameter` objects without translation. The MCS Python SDK provides a reference implementation in [`mcs_tool_driver_interface.py`](https://github.com/modelcontextstandard/python-sdk/blob/main/packages/core/src/mcs/driver/core/mcs_tool_driver_interface.py).

**Why MCS as the recommended companion**: MCS is a **transport-agnostic tool driver interface** - implementations choose any wire protocol (HTTP, gRPC, AS2, CAN-Bus) and any internal schema format (OpenAPI, JSON Schema, proprietary). It delivers the same tool-calling capabilities as alternative protocols without introducing a separate protocol stack. The OCF/MCS pair separates *what was said* (OCF persistence) from *how tools are called at runtime* (MCS interface), without forcing a specific transport or schema format on either side. An OCF document persisted today can be served by any MCS driver without translation; new tool capabilities can be added at the driver level without inventing a new wire protocol.

The conventions:

- **`name`** - unique machine-readable identifier (e.g. `"read_file"`, `"bash"`).
- **`title`** - short human-readable label, optional. May be auto-filled from description.
- **`description`** - full LLM-facing detail; may include usage instructions, constraints, examples.
- **`parameters`** - list of parameter objects, each carrying:
  - **`name`** - parameter identifier (used as JSON key inside OCF `function.arguments`)
  - **`description`** - what the parameter represents
  - **`required`** - boolean
  - **`schema`** - JSON Schema for the parameter's type (e.g. `{"type": "string"}`, `{"type": "integer", "minimum": 0}`)

The three-tier identification (`name` / `title` / `description`) lets a caller expose only `name + title` to the LLM for token-efficient tool listings, then load full `description` on demand when a tool is selected.

A driver implementing the MCS `MCSToolDriver` interface can serve OCF tool calls directly without translation. Implementations using their own internal tool-schema representation can map to OCF as long as they preserve the four conventions above.

---

## Wire mapping

An OCF tool call uses the OpenAI Chat Completions tool_call shape:

```json
{
  "id": "call_a1b2c3",
  "type": "function",
  "function": {
    "name": "<canonical_tool_name>",
    "arguments": "<JSON-encoded string of {param_name: value, ...}>"
  }
}
```

The argument keys are MCS `ToolParameter.name` values. Each value MUST validate against the corresponding `ToolParameter.schema`.

A tool result is a separate message:

```json
{
  "id": "msg_006",
  "message": {
    "role": "tool",
    "tool_call_id": "call_a1b2c3",
    "content": "<result text or JSON-encoded structure>"
  }
}
```

The `content` shape per canonical tool is documented in each tool's section below. For tools that produce structured results (e.g., `bash` with stdout/stderr/exit_code), the result is JSON-encoded inside the content string.

For large results (multi-MB logs, generated files, binary responses), implementations SHOULD use a content array with a `resource_ref` block and externalize the payload to `resources[]` with `kind: "tool_output"`:

```json
{
  "role": "tool",
  "tool_call_id": "call_a1b2c3",
  "content": [
    {"type": "text", "text": "{\"exit_code\": 0, \"stdout_resource\": \"res_stdout_001\"}"},
    {"type": "resource_ref", "resource_id": "res_stdout_001"}
  ]
}
```

This is Anthropic-wire-compatible. For OpenAI strict-wire conversion, `strip_to_wire` must inline the resource content into the string `content` (subject to size limits) or replace it with a placeholder.

---

## Canonical tool names

### `read_file`

Reads a file from the local filesystem and returns its content.

**MCS Tool:**

```python
Tool(
    name="read_file",
    title="Read a file",
    description="Read a file from the local filesystem and return its content as text.",
    parameters=[
        ToolParameter(name="path", description="Absolute or workspace-relative file path",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="encoding", description="Text encoding",
                      required=False, schema={"type": "string", "default": "utf-8"}),
    ],
)
```

**Result content**: file content as a string. For binary files, implementations SHOULD use a `resource_ref` to a `kind: "tool_output"` resource instead of inlining bytes.

---

### `write_file`

Creates a new file or overwrites an existing one with the provided content.

**MCS Tool:**

```python
Tool(
    name="write_file",
    title="Write a file",
    description="Create or overwrite a file with the provided content.",
    parameters=[
        ToolParameter(name="path", description="Target file path",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="content", description="File content as text",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="encoding", description="Text encoding",
                      required=False, schema={"type": "string", "default": "utf-8"}),
    ],
)
```

**Result content**: success marker, e.g. `"Wrote 1234 bytes to /path/foo.py"`.

---

### `edit_file`

Performs an exact search-and-replace edit on an existing file.

**MCS Tool:**

```python
Tool(
    name="edit_file",
    title="Edit a file",
    description="Replace `old_string` with `new_string` in the file at `path`. The match must be exact and unique unless `replace_all` is true.",
    parameters=[
        ToolParameter(name="path", description="File to edit",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="old_string", description="Exact text to find",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="new_string", description="Replacement text",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="replace_all", description="Replace every occurrence",
                      required=False, schema={"type": "boolean", "default": False}),
    ],
)
```

**Result content**: success marker with line/character count, e.g. `"Applied. 14 lines changed."`. On error: error description.

**Error signaling**: Anthropic's `tool_result` block has a native `is_error` flag; OpenAI does not. OCF preserves error state via `messages[].status` on the `role: "tool"` message envelope:

- `status: "ok"` - tool ran successfully (default; equivalent to absent / null).
- `status: "error"` - tool returned an error. Maps to Anthropic `is_error: true` in `strip_to_wire`. For OpenAI strip-to-wire, status is dropped (the error description flows as content text).
- `status: "cancelled"` - tool execution was interrupted (timeout, user cancel, budget limit). OCF-specific; no native Anthropic/OpenAI representation. `strip_to_wire` drops the field for both providers - the consumer of agentic OCF documents reads it directly from the envelope.

Storing status on the envelope (not in `meta` and not inside `message`) keeps the inner `message` block wire-clean and makes the outcome discoverable for any consumer without provider-specific knowledge.

**Alternative form (`apply_diff`)**: For tools that prefer diff-based editing over search/replace, `apply_diff` with parameters `{path, diff}` (unified diff format) is also canonical.

---

### `bash`

Executes a shell command and returns its result.

**MCS Tool:**

```python
Tool(
    name="bash",
    title="Execute a shell command",
    description="Run a command in a bash shell. Returns stdout, stderr, and exit code.",
    parameters=[
        ToolParameter(name="command", description="Shell command line",
                      required=True, schema={"type": "string"}),
        ToolParameter(name="cwd", description="Working directory",
                      required=False, schema={"type": "string"}),
        ToolParameter(name="timeout_ms", description="Timeout in milliseconds",
                      required=False, schema={"type": "integer", "minimum": 0}),
    ],
)
```

**Result content**: JSON-encoded structured result:

```json
{"stdout": "...", "stderr": "...", "exit_code": 0}
```

For large outputs (>~10 KB stdout or stderr), the result message SHOULD use a content array with `resource_ref` blocks pointing to `tool_output` resources containing the streams. The JSON in `content[0].text` then carries `stdout_resource` / `stderr_resource` IDs instead of the inline strings.

---

### `code_execution`

Runs code in a sandboxed interpreter (Python REPL, JavaScript runtime, Code Interpreter).

**MCS Tool:**

```python
Tool(
    name="code_execution",
    title="Execute code",
    description="Run code in a sandboxed interpreter. Returns stdout, stderr, exit code, and any generated artifacts.",
    parameters=[
        ToolParameter(name="language", description="Programming language",
                      required=True,
                      schema={"type": "string", "enum": ["python", "javascript", "shell"]}),
        ToolParameter(name="code", description="Source code to execute",
                      required=True, schema={"type": "string"}),
    ],
)
```

**Result content**: JSON-encoded structured result with optional artifact references:

```json
{
  "stdout": "...",
  "stderr": "...",
  "exit_code": 0,
  "artifacts": [
    {"resource_id": "res_chart_001"}
  ]
}
```

Artifacts (charts, generated files, plots) MUST be entered into `resources[]` with `kind: "tool_output"` and referenced by ID. The result message MAY use a content array with `resource_ref` blocks for direct linking.

---

### `web_fetch`

Fetches a URL and returns its content.

**MCS Tool:**

```python
Tool(
    name="web_fetch",
    title="Fetch a URL",
    description="Fetch the content at the given URL. Returns text for text/html responses, or a resource reference for binary content.",
    parameters=[
        ToolParameter(name="url", description="Target URL",
                      required=True, schema={"type": "string", "format": "uri"}),
        ToolParameter(name="method", description="HTTP method",
                      required=False,
                      schema={"type": "string", "enum": ["GET", "POST"], "default": "GET"}),
    ],
)
```

**Result content**: page content as text. For binary responses (images, PDFs, archives), the result MUST use `resource_ref` to a `tool_output` resource holding the bytes.

---

## ID provenance - `id_origin`

Tools whose source format does not assign IDs to tool calls (Codex, Gemini, some browser-extension exports) require the converter to synthesize tool_call IDs to satisfy OCF's wire-strict requirement. To distinguish synthesized from source-derived IDs, set `tool_calls[].id_origin`:

- **`"source"`** - the ID came from the source format (e.g., Anthropic `tool_use.id`, OpenAI `tool_call.id`, Claude Code `tool_use_id`). Target systems on roundtrip SHOULD preserve this ID verbatim.
- **`"synthesized"`** - the converter generated this ID because the source format had no native ID (e.g., Codex `tool_call` blocks have no `id` field). Target systems on roundtrip MAY safely regenerate or discard the ID, since the source format does not expect a specific value.
- **null / absent** - provenance unknown (legacy or undocumented converter).

The same pattern applies to `messages[].id_origin` for synthesized message IDs (e.g., when a source format event has no native ID and the converter assigns `msg_001`, `msg_002`, etc.).

A roundtrip target reading OCF SHOULD use `id_origin` to decide whether to preserve the ID literally or assign a fresh target-internal ID. Cross-references within OCF (between `tool_call.id` and `tool_call_id` on the corresponding tool message) MUST always be respected - even if both are marked synthesized.

## Argument format conventions

OpenAI Chat Completions stores `tool_calls[].function.arguments` as a **JSON-encoded string**. OCF follows this convention strictly. Sources that store arguments as a structured object (Anthropic `tool_use.input`, Codex `tool_call.arguments`, Gemini `function_call.args`) MUST be `JSON.stringify`'d on projection into OCF; reverse projection uses `JSON.parse`. See [`mapping.md` § Tool-call argument format conventions](mapping.md#tool-call-argument-format-conventions) for the full table.

## Non-canonical tools

Implementations MAY define tools beyond the canonical set above. To preserve cross-converter interpretability:

1. **Tool name** SHOULD be unique and self-descriptive (e.g., `git_commit`, `kubectl_apply`, not `tool_47`).
2. **Tool schema** MAY be embedded in `conversation.meta.tool_schemas` as a map from tool name to a serialized MCS `Tool` definition:

```json
"meta": {
  "tool_schemas": {
    "git_commit": {
      "name": "git_commit",
      "title": "Create a Git commit",
      "description": "Create a new Git commit with the given message and staged files.",
      "parameters": [
        {"name": "message", "description": "Commit message",
         "required": true, "schema": {"type": "string"}},
        {"name": "amend", "description": "Amend the previous commit",
         "required": false, "schema": {"type": "boolean", "default": false}}
      ]
    }
  }
}
```

3. **Result format** SHOULD be a string unless the tool naturally produces structured data, in which case JSON-encoded content is preferred.

Consumers without canonical knowledge of a tool name SHOULD fall back to:

1. Looking up `conversation.meta.tool_schemas[<tool_name>]`.
2. If absent, treating arguments as opaque JSON and result as opaque text.

---

## Round-trip considerations

When converting an agentic IDE session to OCF:

1. **Each rendered "block"** in the source UI (Claude Code's `MessagePart.kind`, Cursor's composer states, Aider's session blocks) maps to a `tool_call` + corresponding `tool_result` message pair.
2. **Source rendering metadata** richer than OCF's structure (line numbers, syntax-highlighting hints, collapse state, UI annotations) belongs in `messages[].meta.<source>_render` - namespaced by source identifier (e.g., `meta.claude_code_render`, `meta.cursor_render`). This keeps the OCF core wire-clean while preserving fidelity.
3. **Preserve the raw source** as a `kind: "raw_native"` resource (see [`mapping.md`](mapping.md#raw-native-preservation)) when conversion fidelity is uncertain. Future converter versions can re-project from the raw bytes without going back to the source platform.

---

## Validation

A consumer validating tool conventions:

1. For tool calls with names matching the canonical list above, MAY validate `arguments` against the documented schema and warn on mismatches.
2. For tool calls with non-canonical names, SHOULD look up `conversation.meta.tool_schemas[<tool_name>]` and validate against the embedded MCS `Tool` definition.
3. SHOULD NOT reject a document on argument-schema validation failure - log and continue. Source platforms occasionally emit non-conformant arguments that are still processed downstream.
