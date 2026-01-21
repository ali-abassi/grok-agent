# Grok Agent

A minimal agentic harness for xAI's Grok models. 

## Why Grok?

- **2M token context** - Largest context window available (vs 200k for Claude/GPT)
- **Fast** - grok-4-1-fast 
- **Smart** - Excellent reasoning and tool use
- **Cheap** - $2/M input, $10/M output

## What Makes This Unique

- **JSON-controlled loop** - Model outputs JSON that decides whether to continue or stop, includes confidence, self check, and exit conditions
- **Same engine, two views** - `grok` (minimal) and `grok-verbose` (verbose) share identical logic
- **Skills system** - Loadable knowledge files the agent uses silently
- **Bare bones** - Easy to understand, easy to extend

---

## Installation

```bash
# 1. Clone the repo
git clone https://github.com/ali-abassi/grok-agent.git
cd grok-agent

# 2. Set up your API key
mkdir -p ~/.grok
cp .env.example ~/.grok/.env
# Edit ~/.grok/.env and add your key from https://console.x.ai

# 3. Install scripts
cp grok grok-verbose ~/.local/bin/
chmod +x ~/.local/bin/grok ~/.local/bin/grok-verbose

# 4. (Optional) Install default skill
mkdir -p ~/.local/bin/skills
cp skills/skill-creation.md ~/.local/bin/skills/

# 5. Run
grok           # minimal output
grok-verbose  # verbose output
```

---

## Usage

### `grok` - Minimal Mode

Clean conversation. Shows: your input, tool calls (one line), response.

```
> what's the weather in SF?
thinking...
[web_search] weather san francisco today

It's 58°F and foggy in San Francisco.

>
```

### `grok-verbose` - Verbose Mode

Full visibility: session stats, task progress, thinking, confidence scores.

Use when debugging or learning how the agent works.

---

## How It Works

### The JSON Contract

The agent always outputs JSON. The `done` field controls the loop:

```json
{
  "thinking": "Need to search for current weather",
  "tool_calls": [{"tool": "web_search", "args": {"query": "SF weather"}}],
  "response": null,
  "done": false
}
```

- `done: false` → execute tools, loop back
- `done: true` → show response, wait for input

### The Loop

```
User Input
    ↓
Send to Grok API (JSON mode)
    ↓
Parse JSON Response
    ↓
┌─────────────────┬──────────────────┐
│ tool_calls?     │ response?        │
│      ↓          │      ↓           │
│ Execute tools   │ Display response │
│ Add results     │ done=true        │
│ Loop back ←─────┘                  │
└────────────────────────────────────┘
```

---

## Tools

| Tool | Args | Description |
|------|------|-------------|
| `bash` | `{"command": "..."}` | Run shell commands (3min timeout) |
| `web_search` | `{"query": "..."}` | Search the web |
| `read_file` | `{"path": "..."}` | Read file contents |
| `write_file` | `{"path": "...", "content": "..."}` | Write/create files |
| `list_files` | `{"path": "..."}` | List directory |
| `ask_user` | `{"question": "...", "options": [...]}` | Ask user for input |

### Adding a Tool

1. **Add function:**
```bash
tool_calc() {
    echo "$1" | bc -l 2>/dev/null || echo "Error"
}
```

2. **Add to switch:**
```bash
calc) tr=$(tool_calc "$(echo "$ta"|jq -r '.expression')") ;;
```

3. **Add to system prompt:**
```
- calc: {"expression": "..."} - Evaluate math
```

---

## Skills

Skills are markdown files that provide domain knowledge. The agent loads them silently when relevant.

### Setup

```bash
mkdir -p ~/.local/bin/skills
```

### Creating a Skill

Just ask:
```
> create a skill for writing tweets
```

Or manually create `~/.local/bin/skills/my-skill.md`:

```markdown
# My Skill

Brief description.

## When to Use
Conditions that trigger this skill.

## Instructions
1. Step one
2. Step two

## Tools to Use
- web_search for research
- write_file for output
```

---

## Commands

| Command | Description |
|---------|-------------|
| `/clear` | Reset conversation |
| `/exit` | Quit (also `/q`) |
| `/help` | Show commands |

**Detailed mode only:**

| Command | Description |
|---------|-------------|
| `/cd DIR` | Change directory |
| `/sessions` | List saved sessions |
| `/resume ID` | Resume session |
| `/compact` | Force context compaction |
| `/cost N` | Set cost limit |

---

## Context Compaction

Grok has a 2M token context window, but long conversations can still get expensive. The verbose mode includes automatic context compaction.

### How It Works

1. **Auto-trigger** - When context usage hits 75% (configurable), compaction runs automatically
2. **Split** - Messages are split: oldest 50% get compacted, newest 50% kept in full
3. **Preserve** - System prompt always preserved, recent context stays intact
4. **Resume** - A placeholder marks where compaction happened, conversation continues seamlessly

```
Before: [system] [msg1] [msg2] [msg3] [msg4] [msg5] [msg6] [msg7] [msg8]
After:  [system] [compacted] [msg5] [msg6] [msg7] [msg8]
```

### Manual Compaction

In verbose mode, force compaction anytime:
```
> /compact
Compacting (45% context used)...
Compacted: 24 -> 14 messages
```

### Configuration

```bash
COMPACT_THRESHOLD=75   # Auto-compact at 75% context usage
```

---

## Configuration

### Environment

The scripts check these locations for `.env` (in order):
1. `./.env` (current directory)
2. `~/.env`
3. `~/.grok/.env` (recommended)

```bash
# ~/.grok/.env
GROK_API_KEY=xai-your-key-here
```

### Script Settings

Edit the scripts to change:

```bash
MODEL="grok-4-1-fast"      # or grok-3-fast for speed
COST_LIMIT=10.00           # max spend per session
COMPACT_THRESHOLD=75       # auto-compact at N% context
```

---

## Extending

Ideas for building on this:

- Add memory (persist across sessions)
- Add RAG (vector search)
- Add more tools (APIs, databases)
- Add planning (multi-step decomposition)
- Add streaming (show tokens as they generate)

---

## License

MIT
