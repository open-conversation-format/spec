# OCF Roadmap

## The pain that makes this exist

You're four hours into a race-condition bug with Claude. You finally crack it. You commit. Two weeks later your tech lead asks why the lock is fair and not greedy. You squint at the diff. You open Claude. That conversation is... somewhere. Maybe it auto-archived. Maybe you switched machines. Maybe your monthly cap rolled and the history is fuzzy. Maybe you finished it in Cursor because Claude's budget hit zero.

The code is in git. The reasoning that produced it is not.

It lives in `~/.claude/projects/`, in Cursor's `state.vscdb`, in `~/.codex/sessions/`, in a ChatGPT tree-shaped tarball, in a browser DOM that no extension scrapes losslessly. Different formats. Different machines. Different subscriptions. Six months later, the conversation that designed your concurrency model is gone or impractical to find — even though every line of that code is right there in `git blame`.

OCF (Open Conversation Format) is the smallest format that lets you change that. One JSON shape. AI conversations from Claude, ChatGPT, Gemini, Cursor, Codex projected into a wire-strict structure that round-trips back without losing the parts that matter — tool calls, file edits, thinking blocks, attached files, code blocks with language hints, model and token telemetry.

The end-state goal: the conversation lives where the code lives. In the repo. Versioned with the commit. Reviewable in the PR. Resumable from any checkout.

This roadmap is how we get there.

---

## Phase 0 — Spec stable (this repo, now)

The format itself. JSON Schema draft-07, platform mappings, tool-call conventions, reference examples.

**What's in this repo right now:**
- [`schema/ocf-v0.1.0.schema.json`](schema/ocf-v0.1.0.schema.json) — wire-strict against OpenAI Chat Completions, role-specific content validation, additionalProperties: false where applicable
- [`mapping.md`](mapping.md) — Claude Code, Cursor, Codex, ChatGPT export, Claude export, Gemini, agentic IDE sessions; canonical JSON contract; redaction; raw_native preservation; bundle path security; partial-tail-safe event log; project organization
- [`tool-conventions.md`](tool-conventions.md) — six canonical tools (read_file, write_file, edit_file, bash, code_execution, web_fetch), MCS-compatible
- [`examples/`](examples/) — including [`sync-tests/`](examples/sync-tests/) with full roundtrip-analysis from real Claude/Cursor/Codex source data

**Status: 0.1.0-draft.** Schema may change incompatibly until 0.1.0 stable. After 0.1.0 stable, semver applies.

---

## Phase 1 — Reference Python library (next)

**Repo:** `ocf-py` (sibling under this org, not yet public)

The thing every implementation needs: schema validation, canonical JSON, hashing, strip_to_wire. Plus the first three exporters from real sources.

**Scope:**
- Schema validator (jsonschema-based, with helpful error messages)
- Canonical JSON serializer (orjson + NFC normalization)
- `strip_to_wire(ocf, target)` — provider-specific translation for OpenAI / Anthropic / Gemini
- Hash computation (sha256 over canonical JSON, with the documented contract)
- Redaction layer with default policy (AWS keys, OpenAI/Anthropic keys, .env lines)
- Bundle reader/writer (`.ocf.json`, `.ocf.jsonl`, `.ocf.zip` with mandatory path-validation)
- Exporters: `claude_code → ocf`, `cursor → ocf`, `codex → ocf`

**Out of scope on purpose:**
- Sync semantics, conflict resolution, multi-device. That's a sync engine, not a format library. ChatSyncer-style projects build that on top.

---

## Phase 2 — Pre-commit hook (the actual reason most people will care)

**Repo:** `ocf-precommit`

This is the killer use case. The reason the spec exists.

**The flow:**
1. You commit code.
2. The hook detects which AI tool you used during this work session (auto from path heuristics, or explicit config).
3. It projects the relevant conversations to OCF, scoped to sessions that touched files in `git diff --cached`.
4. It writes them to `.ocf/sessions/<conv_id>.ocf.json` in the repo, updating existing files when the same conversation evolved further.
5. Optional: runs the test suite, blocks the commit if anything's broken.
6. `git add .ocf/`, commit proceeds.

**Why this changes the developer workflow:**
- `git checkout <old_commit>` → the AI reasoning that produced this state is right there, ready to resume
- `git blame` a confusing line → you can find the OCF document where that line was discussed
- Pull request review includes the conversation — reviewers see what was tried, what failed, what got picked, why
- Onboarding to a codebase means reading code AND the conversations that designed it

**Configurable per-project:**
```toml
[ocf.precommit]
detect = "auto"               # auto | claude_code | cursor | codex | all
scope = "diff"                # diff | session | all-active
output = ".ocf/sessions/"
update_existing = true        # append to existing OCF when same conv_id
redaction_policy = "default"
include_raw = false
test_after = true
```

This is what makes OCF not just a format, but a workflow change.

---

## Phase 3 — Importers (symmetric round-trip)

**Repo:** `importers`

The reverse of Phase 1 exporters. Take an OCF document and reconstruct the source platform's session format.

**The use case:** budget on Claude is exhausted mid-project. You want to continue in Cursor. Export Claude session → OCF → import OCF as a Cursor composer entry. Keep working with the full conversation context intact.

Symmetric for all three sources at minimum: Claude Code, Cursor, Codex.

---

## Phase 4 — Ecosystem fit

- **`ocf-js`** — TypeScript/JavaScript implementation for browser tooling and Node-based exporters
- **`ocf-cli`** — command-line wrapper around export/import/validate, bundle inspect/extract, redaction dry-run
- **Editor integrations** where they earn their keep (VSCode/Zed extension that shows `.ocf/` sessions inline next to commits, scrolling through the reasoning that produced each line)

---

## What's deliberately NOT on the roadmap

To keep OCF small, predictable, and not-yet-another-everything-platform:

- **A new tool protocol.** Use [MCS](https://github.com/modelcontextstandard) — recommended companion. Tool definitions there, conversation persistence here.
- **A new wire transport.** OCF persists conversations. Runtime protocols (MCP, A2A, OpenTelemetry GenAI, ATIF) live elsewhere and stay there.
- **A viewer or chat UI.** Other projects exist. OCF stays a format. If anyone wants a viewer, build it on top.
- **A sync engine.** Use a separate sync layer. OCF is the document; sync is what you do with the document.
- **Project-level OCF documents** (project.ocf.json with shared instructions and resources). v0.2 candidate if usage demands it.
- **OpenAI Responses API mapping.** v0.2 candidate. Same data, different wire shape — not a v0.1 blocker.
- **Multi-writer event log** with global ordering. v0.2+ when single-writer assumptions break in practice.

These aren't "won't ever do" — they're "not earning their complexity yet". Resist scope creep, ship the small thing first.

---

## How to follow along

Spec changes happen in this repo. Watch it for tag releases (0.1.0 stable will be a real milestone — schema-breaking changes stop after that).

Implementation work happens in sibling repos under [open-conversation-format](https://github.com/open-conversation-format) once they go live. They'll be linked from the org profile when content lands.

Spec issues belong here. Implementation bugs belong in the relevant implementation repo when those exist.
