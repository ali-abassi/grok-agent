# Grok Agent

A minimal agentic harness for xAI's Grok models. Designed to be bare bones so you can learn how agent loops work and build upon it.

## Why Grok?

- **2 million token context window** - Significantly larger than Claude, GPT-4, or other models
- **Incredibly fast** - grok-4-1-fast lives up to its name
- **Smart** - Excellent reasoning and tool use
- **Cheap** - $2/M input, $10/M output tokens

## What Makes This Unique

This harness demonstrates core agentic patterns:

1. **JSON Structured Output** - The model outputs valid JSON that controls whether the loop continues or stops
2. **Self-Critique** - The detailed mode includes self-review with grading and improvement suggestions
3. **Exit Conditions** - Define completion criteria upfront, track progress toward them
4. **Skills System** - Loadable knowledge modules the agent uses silently when relevant
5. **Bare Bones Design** - Intentionally minimal so you can understand and extend it

## Quick Start

```bash
# Set your API key
echo 'GROK_API_KEY=your-key-here' >> ~/.claude/.env

# Copy to PATH
cp grok grok-detailed ~/.local/bin/
chmod +x ~/.local/bin/grok ~/.local/bin/grok-detailed

# Create skills directory
mkdir -p ~/.local/bin/skills

# Run
grok           # minimal mode
grok-detailed  # verbose mode
```

## Modes

### `grok` - Minimal Mode

Clean back-and-forth conversation. Shows only:
- Your input
- Tool calls (one line each)
- Agent response (in blue)

```
> what's the weather in sf
[web_search] weather san francisco today

It's 62°F and partly cloudy in San Francisco.

>
```

### `grok-detailed` - Detailed Mode

Full visibility into agent reasoning. Shows:
- Session ID, cost tracking, context usage
- Task description per iteration
- Goal and exit conditions
- Progress tracking (completed/current/remaining)
- Thinking/reasoning process
- Confidence scores
- Self-review grades

Use this mode when debugging, learning, or understanding agent behavior.

---

## How It Works

### The JSON Contract

The agent always responds with structured JSON. This creates a predictable contract between the model and the harness.

**Minimal mode:**
```json
{
  "tool_calls": [{"tool": "bash", "args": {"command": "ls"}}],
  "response": null,
  "done": false
}
```

**Detailed mode (extended):**
```json
{
  "thinking": "Need to search for current info",
  "task": "Search weather data",
  "goal": "Get SF weather",
  "exit_conditions": ["Weather data retrieved", "Answer provided"],
  "progress": {
    "completed": [],
    "current": "Weather data retrieved",
    "remaining": ["Answer provided"]
  },
  "tool_calls": [{"tool": "web_search", "args": {"query": "SF weather"}}],
  "response": null,
  "confidence": 0,
  "done": false
}
```

### The Loop

```
USER INPUT
    │
    ▼
┌─────────────────────────┐
│  Send to Grok API       │
│  (with JSON mode)       │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│  Parse JSON response    │
└───────────┬─────────────┘
            │
    ┌───────┴───────┐
    │               │
    ▼               ▼
tool_calls?     response?
    │               │
    ▼               ▼
┌──────────┐  ┌──────────────┐
│ Execute  │  │ Display      │
│ tools    │  │ response     │
│          │  │ done=true    │
└────┬─────┘  └──────────────┘
     │
     ▼
 Append results
 to messages
     │
     └──────► LOOP BACK
```

**Key insight:** The `done` field controls the loop. When `done=false`, keep iterating. When `done=true`, show the response and wait for next user input.

---

## Skills System

Skills are markdown files that provide domain-specific knowledge. The agent loads them silently when relevant - it never asks the user about skills.

### Setting Up Skills

```bash
# Create skills directory (same location as the scripts)
mkdir -p ~/.local/bin/skills

# Skills are markdown files
~/.local/bin/skills/
├── skill-creation.md    # Default: how to create new skills
├── research.md          # Your custom skills
├── coding.md
└── writing.md
```

### Skill File Format

```markdown
# Skill Name

Brief description of what this skill does.

## When to Use

Conditions that trigger this skill (e.g., "when user asks for research")

## Instructions

Step-by-step guide for the agent:
1. First do this
2. Then do that
3. Finally return this

## Examples

Input: "research topic X"
Output: [what the agent should produce]

## Tools to Use

- web_search for finding info
- write_file for saving results
```

### Default Skill: skill-creation

The harness includes a `skill-creation.md` skill. Just ask:

```
> create a skill for writing tweets
```

The agent will create `~/.local/bin/skills/tweets.md` with the proper format.

### How Skills Work

1. On each message, available skills are injected (names only)
2. If relevant, agent uses `read_file` to load the full skill
3. Agent follows the skill's instructions silently
4. User never sees skill mechanics - just better results

---

## Tools

### Built-in Tools

| Tool | Args | Description |
|------|------|-------------|
| `bash` | `{"command": "..."}` | Run shell commands |
| `web_search` | `{"query": "..."}` | Search the web via Grok |
| `read_file` | `{"path": "..."}` | Read file contents |
| `write_file` | `{"path": "...", "content": "..."}` | Write/create files |
| `list_files` | `{"path": "..."}` | List directory contents |
| `ask_user` | `{"question": "...", "options": [...]}` | Ask user for input |

### Creating a New Tool

Adding a tool requires 3 changes:

#### 1. Add the Tool Function

```bash
tool_my_tool() {
    local arg1="$1"
    local arg2="$2"

    # Your logic here
    echo "Result to return to agent"
}
```

#### 2. Add to the Execute Switch

```bash
case "$tn" in
    bash) tr=$(tool_bash "$(echo "$ta"|jq -r '.command')") ;;
    # ... existing tools ...
    my_tool) tr=$(tool_my_tool "$(echo "$ta"|jq -r '.arg1')" "$(echo "$ta"|jq -r '.arg2')") ;;
    *) tr="Unknown tool" ;;
esac
```

#### 3. Add to System Prompt

```
TOOLS:
- bash: {"command": "..."}
- my_tool: {"arg1": "...", "arg2": "..."}
```

#### Example: Calculator Tool

```bash
# 1. Function
tool_calc() {
    echo "$1" | bc -l 2>/dev/null || echo "Error"
}

# 2. In switch
calc) tr=$(tool_calc "$(echo "$ta"|jq -r '.expression')") ;;

# 3. In prompt
- calc: {"expression": "..."} - Evaluate math
```

### Tool Guidelines

- Return strings (output goes into messages)
- Truncate large outputs with `| head -N`
- Handle errors gracefully
- Keep tools stateless
- Add timeouts for slow operations

---

## Configuration

### Environment

```bash
# Required
GROK_API_KEY=xai-...
```

### Script Settings

Edit the scripts to customize:

```bash
MODEL="grok-4-1-fast"      # or grok-3-fast
COST_LIMIT=10.00           # Max spend (detailed mode)
COMPACT_THRESHOLD=75       # Auto-compact at N% context
```

### Commands

**Both modes:**
| Command | Description |
|---------|-------------|
| `/clear` | Reset conversation |
| `/exit` | Quit (also `/q`, `/quit`) |
| `/help` | Show available commands |

**Detailed mode only:**
| Command | Description |
|---------|-------------|
| `/cd DIR` | Change working directory |
| `/sessions` | List saved sessions |
| `/resume ID` | Resume a saved session |
| `/compact` | Force context compaction |
| `/cost N` | Set cost limit (e.g., `/cost 20`) |
| `/tools` | List available tools |

---

## Extending the Harness

This is intentionally minimal. Ideas for extending:

- **Add memory** - Persist facts across sessions
- **Add RAG** - Vector search over documents
- **Add more tools** - APIs, databases, services
- **Add planning** - Multi-step task decomposition
- **Add evaluation** - Track success rates
- **Add streaming** - Show responses as they generate

---

## License

MIT
