---
name: ollama-auto
description: >
  Auto-delegate cheap text tasks to local Ollama (Mistral) — saves Claude tokens for complex reasoning.
  Automatically triggers on: large list classification, text formatting/cleaning, output pre-filtering,
  boilerplate generation, simple text decisions. Includes audit mode to track effectiveness over time.
  Trigger: "/ollama-auto", "--audit" to see stats, or auto-triggered by Claude on eligible tasks.
---

# ollama-auto — Smart Ollama Delegation + Audit

Delegates cheap mechanical tasks to local Ollama instead of burning Claude tokens.
Logs every call. Audit mode shows if it's actually saving tokens.

---

## Auto-Trigger Rules

Claude MUST invoke Ollama automatically (without user asking) when ALL conditions are true:

| Condition | Threshold |
|-----------|-----------|
| Task is mechanical (classify, sort, format, filter, describe) | Yes |
| No codebase context needed | Yes |
| Input is plain text / list / JSON | Yes |
| Output doesn't need Claude reasoning | Yes |

**Auto-trigger examples:**
- Scan result with 30+ folders → classify each as `bloatware / app / game / dev / media / system`
- winget list → tag each as `keeper / remove / update`
- List of file paths → group by type/purpose
- Raw command output → extract only relevant lines
- Template generation with repetitive structure

**Never auto-delegate:**
- Tasks needing repo/file context
- Debugging or architecture decisions
- Image/video analysis
- Anything requiring Claude reasoning or judgment

---

## How to Call Ollama

```bash
# Simple prompt
curl -s http://localhost:11434/api/generate \
  -d "{\"model\":\"mistral\",\"prompt\":\"PROMPT_HERE\",\"stream\":false}" \
  | python -c "import sys,json; print(json.load(sys.stdin)['response'])"
```

```bash
# Long prompt via temp file (avoids shell escaping hell)
cat > /tmp/ollama_prompt.json << 'EOF'
{
  "model": "mistral",
  "prompt": "YOUR LONG PROMPT HERE",
  "stream": false
}
EOF
curl -s http://localhost:11434/api/generate -d @/tmp/ollama_prompt.json \
  | python -c "import sys,json; print(json.load(sys.stdin)['response'])"
```

---

## Logging (Required After Every Call)

After every Ollama call, append to audit log:

```bash
python -c "
import json, time, sys
entry = {
  'ts': time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime()),
  'task_type': 'TASK_TYPE',        # e.g. classify_list, format_text, filter_output
  'input_chars': INPUT_CHAR_COUNT,
  'output_chars': OUTPUT_CHAR_COUNT,
  'success': True,                  # False if output was wrong/unusable
  'model': 'mistral',
  'duration_s': DURATION_SECONDS,
  'note': 'optional context'
}
with open(r'C:\Users\CURRENTUSER\.claude\ollama_audit.jsonl', 'a') as f:
    f.write(json.dumps(entry) + '\n')
"
```

Replace `CURRENTUSER` with actual username. Replace TASK_TYPE, INPUT_CHAR_COUNT, etc. with real values.

---

## Audit Mode — `/ollama-auto --audit`

When user runs `/ollama-auto --audit`, read the audit log and report:

```bash
python -c "
import json, os
log_path = os.path.expanduser('~/.claude/ollama_audit.jsonl')
if not os.path.exists(log_path):
    print('No audit log yet. Use Ollama first.')
    exit()

entries = [json.loads(l) for l in open(log_path) if l.strip()]
if not entries:
    print('Log empty.')
    exit()

total = len(entries)
success = sum(1 for e in entries if e.get('success'))
total_in = sum(e.get('input_chars', 0) for e in entries)
total_out = sum(e.get('output_chars', 0) for e in entries)
est_tokens_saved = int((total_in + total_out) / 4)

from collections import Counter
types = Counter(e.get('task_type','unknown') for e in entries)

print(f'=== OLLAMA AUDIT ===')
print(f'Total calls:       {total}')
print(f'Success rate:      {success/total*100:.0f}% ({success}/{total})')
print(f'Chars processed:   {total_in + total_out:,}')
print(f'Est tokens saved:  ~{est_tokens_saved:,}')
print(f'')
print(f'Task breakdown:')
for t, c in types.most_common():
    print(f'  {t}: {c}')
print(f'')
print(f'Last 5 calls:')
for e in entries[-5:]:
    status = \"OK\" if e.get(\"success\") else \"FAIL\"
    print(f'  [{status}] {e[\"ts\"][:10]} {e.get(\"task_type\",\"?\")} ({e.get(\"input_chars\",0)} chars in)')
"
```

---

## Prompt Templates

### Classify list
```
You are a classifier. For each item below, output ONLY the item name and its category tag.
No explanations. Format: "item — CATEGORY"

Categories: BLOATWARE, APP, GAME, DEV_TOOL, MEDIA, SYSTEM, UNKNOWN

Items:
{LIST}
```

### Filter output (keep relevant lines only)
```
From the text below, extract ONLY lines that contain {WHAT_TO_KEEP}.
Output ONLY the extracted lines, nothing else.

Text:
{TEXT}
```

### Format/clean text
```
Fix the formatting of the text below. Output ONLY the fixed text, no commentary.
Rules: {RULES}

Text:
{TEXT}
```

---

## Diagnostics

| Error | Fix |
|-------|-----|
| `Connection refused` | Run `ollama serve` first |
| Slow first call (~12s) | Model loading into RAM, normal |
| Wrong output format | Add "respond ONLY with X, no extra text" to prompt |
| Hallucinated data | Verify output before applying — always |
