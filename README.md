# Strix-Mini 🦅

> A hardened, single-agent, local-LLM-optimized fork of the [Strix](https://github.com/usestrix/strix-agent) penetration testing framework.

Strix-Mini strips out the multi-agent overhead, compacts the system prompt, enforces tool-first workflows, and adds targeted single-check mode — making it practical to run full pentest automation with local models like **Gemma 4**, **Qwen 2.5**, or **DeepSeek** via LM Studio or Ollama.

---

## What's Different from Upstream

| Feature | Upstream Strix | Strix-Mini |
|---|---|---|
| Agent model | Multi-agent (spawns subagents) | **Single-agent enforced** |
| System prompt | ~500 lines verbose | **Compact Jinja template** |
| LLM target | Cloud APIs (Claude, GPT) | **Local models via LM Studio** |
| HTTP testing | Writes `python requests` scripts | **Prefers `curl` via terminal_execute** |
| Scan modes | Full pentest only | **Full pentest + `--check` single-task mode** |
| Clipboard (macOS) | Broken in terminal | **Fixed via `pbcopy`** |
| Thinking display | Only `<think>` tags stripped | **Strips Gemma/DeepSeek/Qwen/all formats** |
| Docker | Silent fallback to local | **Fails loudly if SSH host unreachable** |
| Token usage | ~3M+ per scan | **~100-200K per scan** |

---

## Installation

```bash
git clone https://github.com/h0tak88r/Strix-Mini.git
cd Strix-Mini
pip install -e .
```

### Requirements

- Python 3.11+
- Docker (local or remote via SSH)
- [LM Studio](https://lmstudio.ai) or Ollama running locally
- SSH key loaded: `ssh-add ~/.ssh/id_rsa`

---

## Configuration

Copy and edit `cli-config.json`:

```json
{
  "env": {
    "DOCKER_HOST": "ssh://user@your-vps-ip",
    "LLM_API_BASE": "http://127.0.0.1:1234/v1",
    "STRIX_LLM": "openai/google/gemma-4-26b-a4b",
    "LLM_API_KEY": "not-needed",
    "STRIX_SINGLE_AGENT": "true",
    "STRIX_COMPACT_PROMPT": "true",
    "LLM_TIMEOUT": "600",
    "STRIX_MAX_OUTPUT_TOKENS": "2048"
  }
}
```

> **Note:** Always run `ssh-add ~/.ssh/id_rsa` before starting if using a remote Docker host.

---

## Usage

### Full Pentest (TUI mode)

```bash
export PYTHONPATH=$PYTHONPATH:.
python3 strix/interface/main.py \
  --target "https://target.com/" \
  --config cli-config.json
```

### Full Pentest (Non-interactive / CLI mode)

```bash
python3 strix/interface/main.py \
  --target "https://target.com/" \
  --config cli-config.json \
  --non-interactive
```

### ⚡ Single Check Mode (`--check`)

Run **one targeted test** instead of a full pentest workflow. The agent does exactly the task you specify, reports success/failure, and stops immediately.

```bash
# Test for SSRF
python3 strix/interface/main.py \
  --target "https://app.com/" \
  --config cli-config.json \
  --check "test if /api/validate-url is vulnerable to SSRF using http://169.254.169.254/"

# Test for XSS
python3 strix/interface/main.py \
  --target "https://app.com/" \
  --config cli-config.json \
  --check "test XSS on /search?q= parameter"

# Test for IDOR
python3 strix/interface/main.py \
  --target "https://app.com/" \
  --config cli-config.json \
  --check "check if /api/users/{id} has IDOR between accounts"
```

In check mode the agent:
1. Runs the check directly — no recon phases, no todo list
2. Validates the result with one confirmation step
3. Files a `create_vulnerability_report` if vulnerable, or calls `finish_scan` with "not found"
4. Stops completely

---

## Optimizations Made

### 1. Single-Agent Enforcement
`create_agent` is hard-blocked in `agents_graph_actions.py`. The `STRIX_SINGLE_AGENT=true` flag prevents any subagent spawning at the code level, not just the prompt level.

### 2. Compact System Prompt
`system_prompt_compact.jinja` replaces the 500-line verbose prompt with a ~100-line focused template that:
- Has explicit tool priority (curl > execute_skill > python)
- Forbids writing Python for HTTP requests
- Enforces `finish_scan` with one line instead of an essay
- Has a `check_mode` branch for `--check` flag

### 3. Tool Priority Hierarchy
The agent is explicitly instructed to prefer:
```
1. terminal_execute  (curl, httpx, nmap, nuclei)
2. execute_skill     (pre-built security scripts)
3. browser_action    (JS-rendering only)
4. python            (data processing only)
```
Writing `python requests` for HTTP calls is **explicitly forbidden** in the prompt.

### 4. LM Studio Timeout Fixes
- Per-chunk timeout raised to `max(LLM_TIMEOUT, 120)` seconds to prevent mid-stream disconnects during thinking
- `reasoning_effort` parameter skipped for local models (127.0.0.1 / localhost) — it caused infinite thinking loops with Gemma
- `max_tokens=2048` cap prevents runaway generation

### 5. Gemma 4 Thinking Tag Stripping
The streaming parser and LLM response cleaner now strip all thinking formats:
- `<think>` / `<thinking>` — DeepSeek, Qwen
- `<thought>` — Gemma 4 variant
- `<|channel>` — Gemma 4 channel blocks

### 6. Remote Docker Hardening
`check_docker_connection()` in `utils.py` now:
- Prints the active Docker host at startup
- Fails immediately with an actionable error if the SSH host is unreachable
- No longer silently falls back to local Docker

### 7. macOS Clipboard Fix
`copy_to_clipboard()` overridden in `StrixTUIApp` to use `pbcopy` on macOS via subprocess, bypassing Textual's broken internal clipboard.

### 8. Operations Log Panel
New left-side panel in the TUI showing real-time tool operations:
- ✓ green = success, ● yellow = running, ✗ red = failed
- Shows tool name, key argument preview, error snippet on failure
- Layout: `[Ops Log 20%] | [Chat 60%] | [Agents/Vulns 20%]`

---

## Custom Skills

Pre-built attack scripts live in `strix/skills/custom/`. The agent calls them via `execute_skill` instead of writing Python from scratch.

Current skills:
- `testing-for-xss-vulnerabilities` → `scripts/xss_scanner.py`
- `directory-enumeration` → `scripts/dir_fuzzer.py`

### Adding a Skill

```
strix/skills/custom/
└── my-skill-name/
    ├── skill.md          # Skill description loaded into system prompt
    └── scripts/
        └── my_script.py  # Script the agent runs via execute_skill
```

---

## Environment Variables Reference

| Variable | Default | Description |
|---|---|---|
| `STRIX_LLM` | — | LiteLLM model name (required) |
| `LLM_API_BASE` | — | Local LLM server URL |
| `LLM_API_KEY` | — | API key (use "not-needed" for local) |
| `DOCKER_HOST` | local | Docker host (`ssh://user@host`) |
| `STRIX_SINGLE_AGENT` | false | Disable subagent spawning |
| `STRIX_COMPACT_PROMPT` | false | Use compact system prompt |
| `LLM_TIMEOUT` | 300 | LLM request timeout in seconds |
| `STRIX_MAX_OUTPUT_TOKENS` | 2048 | Max tokens per LLM response |
| `STRIX_CHECK_MODE` | false | Single-task mode (set by `--check`) |

---

## Tested Models

| Model | Works? | Notes |
|---|---|---|
| `google/gemma-4-26b-a4b` | ✅ | Best local results. Use `STRIX_COMPACT_PROMPT=true` |
| `qwen2.5-coder-14b` | ✅ | Good tool following |
| `deepseek-r1-7b` | ⚠️ | Thinking loops without token cap |
| Claude Sonnet 3.7 | ✅ | Original target model |

---

## License

This is a fork/optimization of [Strix Agent](https://github.com/usestrix/strix-agent). Original license applies.
