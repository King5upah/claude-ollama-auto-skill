# ollama-auto — Claude Code Skill

A Claude Code skill that automatically delegates cheap text tasks to local [Ollama](https://ollama.com) (Mistral), saving Claude tokens for complex reasoning.

**No MCP server. No plugins. Pure CLI** — Claude calls Ollama directly via `curl` from the terminal. Zero infrastructure overhead.

Includes an **audit mode** to track whether Ollama is actually helping over time — something no similar project does.

---

## Why not MCP?

Most Ollama+Claude integrations require setting up an MCP server. This skill skips all that:

- Claude calls `http://localhost:11434` directly via `curl` in Bash
- No server process to manage
- No config files
- Works anywhere Ollama is running

---

## What it does

- **Auto-delegates** mechanical tasks (list classification, text filtering, formatting) to local Ollama
- **Logs every call** to a local audit file
- **Audit mode** (`--audit`) shows stats: total calls, success rate, estimated tokens saved

## Install

1. Copy `SKILL.md` into your Claude Code skills directory:
   ```
   ~/.claude/skills/ollama-auto/SKILL.md
   ```

2. Make sure Ollama is running with Mistral:
   ```bash
   ollama pull mistral
   ollama serve
   ```

3. In Claude Code, the skill activates automatically on eligible tasks, or manually:
   ```
   /ollama-auto
   /ollama-auto --audit
   ```

## Audit log

Each Ollama call is logged to `~/.claude/ollama_audit.jsonl`:

```json
{"ts":"2026-05-27T01:00:00Z","task_type":"classify_list","input_chars":2048,"output_chars":512,"success":true,"model":"mistral","duration_s":8.2}
```

Run `/ollama-auto --audit` to see a summary report.

## Requirements

- [Claude Code](https://claude.ai/code)
- [Ollama](https://ollama.com) running locally
- `mistral` model pulled (`ollama pull mistral`)
- Python 3 (for log parsing)

## License

MIT
