# OCF Roundtrip Analysis — ChatSyncer Sources

**Source project**: `C:\Development\Projekte\Python\ChatSyncer`
**Tested adapters**: `claude_code`, `cursor`, `codex`
**OCF version**: 0.1.0-draft
**Test method**: Static analysis. No code executed. Each ChatSyncer fixture (from `tests/conftest.py`) was projected by hand into OCF following the adapters' normalization rules in `src/chatsyncer/adapters/sources/{claude_code,cursor,codex}.py`. Then each OCF document was checked for round-trip completeness — i.e., can the original source format be reconstructed from the OCF alone without consulting the source again?

**Assumption**: Internal IDs in the target system differ from those in the source system. The OCF document is the only carrier of identity-and-content; the consumer creates new internal IDs as needed. Cross-reference is only via OCF-internal IDs (`messages[].id`, `tool_calls[].id`, `resources[].id`).

---

## Source 1 — Claude Code

### Source format (per `_claude_lines()` in `tests/conftest.py`)

JSONL with five line types: `system`, `user`, `assistant`, `tool_result`, `result`.

```jsonl
{"type":"system","cwd":"/home/user/projects/demo","model":"claude-opus-4-7","timestamp":"2026-04-24T10:00:00Z"}
{"type":"user","timestamp":"2026-04-24T10:00:05Z","message":{"id":"u1","role":"user","content":"Hallo Claude, schreib mir eine Funktion."}}
{"type":"assistant","timestamp":"2026-04-24T10:00:10Z","message":{"id":"a1","role":"assistant","model":"claude-opus-4-7","content":[{"type":"thinking","thinking":"Thinking privately..."},{"type":"text","text":"Hier ist die Funktion:"},{"type":"tool_use","id":"t1","name":"Write","input":{"path":"demo.py","content":"def add(a,b): return a+b"}}]}}
{"type":"tool_result","timestamp":"2026-04-24T10:00:11Z","tool_use_id":"t1","content":"File created at demo.py"}
{"type":"result","timestamp":"2026-04-24T10:00:12Z","duration_ms":1200,"usage":{"input_tokens":50,"output_tokens":120}}
```

### OCF projection: `claude-code.ocf.json`

5 → 4 messages: `system`, `user`, `assistant`, `tool`. The Claude `result` event is dropped from `messages[]` (per the adapter), but its data is preserved on the assistant envelope (`duration_ms`, `usage`) AND in `conversation.meta.claude_code.result_event_data` for full round-trip.

### Forward projection — losses?

| Source field                                        | OCF location                                        | Lossless? |
| --------------------------------------------------- | --------------------------------------------------- | --------- |
| `system.cwd`                                        | `conversation.project.description` + `messages[0].meta.claude_code_render.cwd` | ✅ |
| `system.model`                                      | `conversation.default_model` + `messages[0].model`  | ✅ |
| `system.timestamp`                                  | `messages[0].created_at` + `conversation.created_at` | ✅ |
| `user.message.id`                                   | `messages[1].id`                                    | ✅ (preserved verbatim) |
| `user.message.content` (string)                     | `messages[1].message.content`                       | ✅ |
| `user.timestamp`                                    | `messages[1].created_at`                            | ✅ |
| `assistant.message.id`                              | `messages[2].id`                                    | ✅ |
| `assistant.message.model`                           | `messages[2].model`                                 | ✅ |
| `assistant.content[].thinking`                      | `messages[2].message.content[].thinking` (type=thinking) | ✅ |
| `assistant.content[].text`                          | `messages[2].message.content[].text` (type=text) | ✅ |
| `assistant.content[].tool_use{id,name,input}`       | `messages[2].message.tool_calls[]{id,function.name,function.arguments}` | ⚠ Format-shift: input (object) → arguments (JSON-string). Reversible via JSON.parse. |
| `assistant` block order                             | `messages[2].meta.claude_code_render.original_block_order` | ✅ (preserved as hint) |
| `tool_result.tool_use_id`                           | `messages[3].message.tool_call_id`                  | ✅ |
| `tool_result.content`                               | `messages[3].message.content`                       | ✅ |
| `tool_result.timestamp`                             | `messages[3].created_at`                            | ✅ |
| `result.duration_ms`                                | `messages[2].duration_ms`                           | ✅ |
| `result.usage.input_tokens` / `output_tokens`       | `messages[2].usage.input` / `output`                | ✅ |
| `result.timestamp`                                  | `conversation.meta.claude_code.result_event_data` (full record) | ✅ |
| Filename `session-abc-123.jsonl`                    | `conversation.source.original_id` + `resources[0].filename` | ✅ |

### Reverse projection (OCF → Claude Code)

A consumer reconstructs the JSONL line-by-line:

1. **system event**: from `messages[0].meta.claude_code_render.cwd`, `messages[0].model`, `messages[0].created_at`. The synthesized OCF system message ID is dropped (Claude system events have no `id` field).
2. **user event**: from `messages[1]`. `message.id` = OCF id; `message.content` = OCF content; `timestamp` = `created_at`.
3. **assistant event**: from `messages[2]`. Reconstruct `content[]` array by:
   - Walking `original_block_order` from meta;
   - For each entry, picking from OCF content (text/thinking) OR from tool_calls (the tool_use entry) in order;
   - For tool_use: emit `{"type":"tool_use","id":<tool_calls[0].id>,"name":<function.name>,"input":<JSON.parse(function.arguments)>}`.
4. **tool_result event**: from `messages[3]`. `tool_use_id` = `tool_call_id`; `content` = content; `timestamp` = `created_at`.
5. **result event**: from `conversation.meta.claude_code.result_event_data` — synthesized as a final line right after the assistant message it telemeters.

**Verdict**: ✅ **Round-trip lossless** with respect to the data preserved by the ChatSyncer adapter. The `_strip_keys(ev, {"type"})` call in the adapter means *additional* unrecognized fields on the source events would be lost — but those are in the `raw_native` resource (sha256), recoverable via re-projection.

**Internal ID divergence handling**: All cross-references inside OCF use OCF-internal IDs. `tool_call.id = "t1"` is the OCF ID; the corresponding `tool_call_id = "t1"` on the tool message matches. A target Claude Code instance can either preserve `t1` literally (the format permits arbitrary tool-use IDs) or rewrite it to `<new_id>` consistently in both places.

---

## Source 2 — Cursor

### Source format (per `cursor_state_db` in `tests/conftest.py`)

SQLite KV store with `composerData:<cid>` and `bubbleId:<cid>:<bid>` rows.

```json
// composerData:c-1
{
  "_v": 3,
  "workspaceFolder": "/home/user/projects/demo",
  "title": "Async-Diskussion",
  "latestConversationSummary": {"model": "cursor-large", "title": "Async-Diskussion"},
  "fullConversationHeadersOnly": [{"bubbleId": "b-1"}, {"bubbleId": "b-2"}]
}

// bubbleId:c-1:b-1
{"type": 1, "text": "Erklär mir async/await", "createdAt": "2026-04-24T09:00:00Z"}

// bubbleId:c-1:b-2
{"type": 2,
 "text": "async/await sind Syntax-Zucker…",
 "thinking": "Nutzer fragt nach async.",
 "codeBlocks": [{"lang": "python", "code": "async def f(): pass"}],
 "createdAt": "2026-04-24T09:00:05Z"}
```

### OCF projection: `cursor.ocf.json`

2 messages: `user` (b-1), `assistant` (b-2). The `_v=3`, workspace folder, headers order, and codeBlocks are preserved in `conversation.meta.cursor` and `messages[1].meta.cursor_render`.

### Forward projection — losses?

| Source field                                        | OCF location                                        | Lossless? |
| --------------------------------------------------- | --------------------------------------------------- | --------- |
| `composer._v`                                       | `conversation.meta.cursor.composer_v`               | ✅ |
| `composer.workspaceFolder`                          | `conversation.project.description` + `meta.cursor.workspace_folder` | ✅ |
| `composer.title`                                    | `conversation.title`                                | ✅ |
| `composer.latestConversationSummary.model`          | `conversation.default_model` + `meta.cursor.latest_conversation_summary.model` | ✅ |
| `composer.latestConversationSummary.title`          | `meta.cursor.latest_conversation_summary.title`     | ✅ |
| `composer.fullConversationHeadersOnly[]`            | `meta.cursor.header_order` (also recoverable from `messages[]` array order) | ✅ |
| `bubble.type` (1=user, 2=assistant)                 | `messages[].message.role` + `meta.cursor_render.bubble_type` | ✅ |
| `bubble.text`                                       | `messages[].message.content` (text block) + origin in `meta.cursor_render.content_block_origins` | ✅ |
| `bubble.thinking`                                   | `messages[].message.content` (thinking block) | ✅ |
| `bubble.codeBlocks[]{lang, code}`                   | `messages[].message.content[]` as native `code` block (`{type: "code", code, language}`) | ✅ (first-class, no meta workaround) |
| `bubble.createdAt`                                  | `messages[].created_at`                             | ✅ |
| `bubble.id`                                         | `messages[].id`                                     | ✅ |
| `composer_id` (`c-1`)                               | `meta.cursor.composer_id` + `source.original_id` (`state.vscdb::c-1`) | ✅ |

**`agentKv:*` and `checkpointId:*`** keys are deliberately ignored by the adapter (M1 out-of-scope). They do NOT appear in OCF — but they are present in the `raw_native` resource (the sha256-addressed copy of the full extracted payload), so a future converter could pick them up.

### Reverse projection (OCF → Cursor)

A consumer reconstructs the KV store rows:

1. **`composerData:c-1`**: build from
   - `_v` ← `meta.cursor.composer_v`
   - `workspaceFolder` ← `meta.cursor.workspace_folder` (or `conversation.project.description`)
   - `title` ← `conversation.title`
   - `latestConversationSummary` ← `meta.cursor.latest_conversation_summary`
   - `fullConversationHeadersOnly` ← `meta.cursor.header_order` mapped to `[{"bubbleId": id}, ...]`
   - The `c-1` composer ID itself can be preserved from `meta.cursor.composer_id` or regenerated at the target.
2. **`bubbleId:c-1:b-1`** (user): build from
   - `type` ← 1 (from `role: user`)
   - `text` ← `message.content` (string)
   - `createdAt` ← `created_at`
3. **`bubbleId:c-1:b-2`** (assistant): build from
   - `type` ← 2 (from `role: assistant`)
   - `text` ← OCF content blocks of `type: "text"` (concatenated or first, per Cursor's text field)
   - `thinking` ← OCF content block of `type: "thinking"`
   - `codeBlocks` ← OCF content blocks of `type: "code"` (each one becomes `{lang: <language>, code: <code>}`)
   - `createdAt` ← `created_at`

**Verdict**: ✅ **Round-trip lossless**. The `content_block_origins` map in meta resolves the question "which OCF text block came from which Cursor field" so that codeBlocks reconstruction is deterministic.

**Internal ID divergence handling**: A target Cursor install assigns its own composer/bubble IDs. The OCF `messages[].id` values (`b-1`, `b-2`) are stable identifiers within OCF; on import to a different Cursor install, the importer creates new bubble IDs and updates `fullConversationHeadersOnly` accordingly. The `meta.cursor.composer_id` is informative-only on the target side.

**Edge case (composer schema bump)**: If Cursor releases `_v=4` with new fields, OCF's `meta.cursor.composer_v=3` plus the `raw_native` resource (with `meta.raw_format_version: "3"`) lets future converters re-project the original v3 data into a richer OCF document. Old OCF files don't gain new fields, but the source-of-truth is preserved.

---

## Source 3 — Codex

### Source format (per `_codex_lines()` in `tests/conftest.py`)

JSONL with `system`, `user`, `assistant` events, Unix-second timestamps, content as block arrays.

```jsonl
{"type":"system","cwd":"/srv/work/codex-project","model":"gpt-5-codex","timestamp":1713961200}
{"type":"user","timestamp":1713961210,"message":{"role":"user","content":[{"type":"text","text":"Generate a unit test."}]}}
{"type":"assistant","timestamp":1713961225,"message":{"role":"assistant","model":"gpt-5-codex","content":[{"type":"text","text":"Here is a unit test."},{"type":"tool_call","name":"run_shell","arguments":{"cmd":"pytest -q"}}]}}
```

### OCF projection: `codex.ocf.json`

3 messages: `system`, `user`, `assistant`. Message IDs synthesized (the Codex fixture provides no `message.id` fields). Timestamps converted from Unix seconds to ISO 8601 UTC.

### Forward projection — losses?

| Source field                                        | OCF location                                        | Lossless? |
| --------------------------------------------------- | --------------------------------------------------- | --------- |
| `system.cwd`                                        | `conversation.project.description` + `meta.codex.absolute_path_at_capture` + `messages[0].meta.codex_render.cwd` | ✅ |
| `system.model`                                      | `conversation.default_model` + `messages[0].model`  | ✅ |
| `system.timestamp` (Unix seconds)                   | `messages[0].created_at` (ISO 8601) + `meta.codex_render.raw_timestamp` (preserved Unix value) | ✅ |
| `user.message.role`                                 | `messages[1].message.role`                          | ✅ |
| `user.message.content[]` (block array)              | `messages[1].message.content[]`                     | ✅ |
| `user.timestamp` (Unix)                             | `messages[1].created_at` + `meta.codex_render.raw_timestamp` | ✅ |
| `assistant.message.model`                           | `messages[2].model`                                 | ✅ |
| `assistant.content[].text`                          | `messages[2].message.content[].text`                | ✅ |
| `assistant.content[].tool_call{name, arguments}`    | `messages[2].message.tool_calls[]{function.name, function.arguments}` | ⚠ Format-shift: arguments (object) → arguments (JSON-string), tool_call moved from in-content to sibling. Both reversible via meta hints + JSON.parse. |
| `assistant` block order (text+tool_call)            | `messages[2].meta.codex_render.original_block_order` | ✅ |

**Codex tool_call has no native `id`** — synthesized as `synth_t_001`. The synthesized IDs are explicitly listed in `meta.codex_render.synthesized_tool_call_ids` so a roundtrip can drop them (Codex doesn't expect them).

### Reverse projection (OCF → Codex)

1. **system event**: `cwd` ← `meta.codex_render.cwd`; `model` ← `messages[0].model`; `timestamp` ← `meta.codex_render.raw_timestamp` (preserved Unix), or convert `created_at` ISO → Unix if missing.
2. **user event**: `message.role` ← `role`; `message.content` ← content array; `timestamp` ← `raw_timestamp`.
3. **assistant event**: `message.role` ← `role`; `message.model` ← `model`; reconstruct `message.content[]` by walking `original_block_order`:
   - For each entry: pick text blocks from OCF content, or pick a tool_call from `tool_calls[]`;
   - For tool_call entries: emit `{"type":"tool_call","name":<function.name>,"arguments":<JSON.parse(function.arguments)>}`;
   - **Discard synthesized tool_call IDs** (Codex doesn't expect them) — `synthesized_tool_call_ids` lists exactly which to drop.

**Verdict**: ✅ **Round-trip lossless** for the data the adapter normalizes. The synthesized IDs case is correctly handled via the `synthesized_tool_call_ids` hint.

**Internal ID divergence**: All synthetic OCF message IDs (`msg_001`/`msg_002`/`msg_003`) are OCF-only; on roundtrip back to Codex they're discarded since Codex assigns its own. The synthetic tool_call ID is similarly discarded.

---

## Cross-source observations

### What works well in OCF v0.1.0

1. **Wire-strict `messages[].message`** — projects cleanly from all three platforms. Tool-related restructuring (Anthropic in-content → sibling, Codex in-content → sibling) is mechanical and reversible via `meta.<source>_render.original_block_order`.
2. **`raw_native` resource** — provides the safety net. If a converter mapping has bugs, the original bytes are preserved and re-projection is possible.
3. **`meta.<source>_render`** namespacing — the right pattern for source-specific structural hints (block order, synthesized fields, schema versions). Doesn't pollute the OCF core.
4. **`conversation.project`** — adequately captures the workspace-folder concept across all three platforms. The `description` field doubles as "absolute path at capture" for non-Git workspaces.
5. **`produced_by` + `mapping_id`** — lets future tooling identify which converter version produced this OCF and target re-projection accordingly.

### Gaps closed in v0.1.0 (after first review pass)

1. **`code` content block** — OCF v0.1.0 now includes a structured `code` block (`{type: "code", code, language?, filename?}`) alongside `image_url`/`input_audio`/`file`. Cursor's codeBlocks project directly without the markdown-fence redundancy. The `language` is preserved as a first-class field, not buried in meta. `strip_to_wire` translates code blocks to Markdown fenced text for OpenAI/Anthropic/Gemini, with `language` becoming the fence info-string.
2. **`id_origin` field** — `messages[].id_origin` and `tool_calls[].id_origin` (`"source"` / `"synthesized"` / null) replace the ad-hoc `meta.<source>_render.id_synthesized: true` and `synthesized_tool_call_ids: [...]` patterns. Roundtrip targets read this directly.
3. **Tool-call argument format shift** — documented in `mapping.md § Tool-call argument format conventions` and `tool-conventions.md § Argument format conventions`. The canonical rule: OCF stores `function.arguments` as JSON-string (OpenAI wire convention); reverse projection to Anthropic/Codex/Gemini uses `JSON.parse`.

### Remaining minor items (v0.2+ candidates, non-blocking)

1. **Unix-second timestamps in source need explicit preservation** if the source uses them, otherwise re-conversion to ISO 8601 loses the original integer precision. Currently using `meta.<source>_render.raw_timestamp` — works fine, just per-converter convention.

### What's potentially lost without `raw_native`

If a converter chooses NOT to emit a `raw_native` resource:

- **Cursor**: the `agentKv:*` and `checkpointId:*` keys (out-of-scope for the M1 adapter) become unrecoverable from OCF alone. ChatSyncer's design correctly requires `raw_native` for this reason.
- **Claude Code**: any unknown event types beyond `system`/`user`/`assistant`/`tool_use`/`tool_result`/`result` would also be lost.
- **Codex**: any future event types or fields would be lost.

`raw_native` is therefore the right pattern. Implementations SHOULD always include it for sources whose schema may evolve.

---

## Final verdict

**OCF v0.1.0 supports bi-directional projection for all three ChatSyncer source adapters tested**, with the following understanding:

- ✅ The OCF document carries enough information to reconstruct the source format fields the adapter normalizes.
- ✅ Internal-ID divergence is handled via OCF-internal IDs that are stable within OCF but can be regenerated on import.
- ✅ Source-specific structural hints (block order, synthesized fields, schema versions) live in `meta.<source>_render` namespaces.
- ✅ The `raw_native` safety net catches anything the current adapter mapping doesn't normalize.
- ⚠ Two minor v0.2 cleanup items identified (id-origin marking, tool-call-argument-format documentation) — neither blocks v0.1.0.

The format is **production-ready for use as ChatSyncer's interchange layer**, including for roundtrip back to other Claude Code / Cursor / Codex installations.
