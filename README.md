# Grok Agent

A minimal agentic harness for xAI's Grok models. Two modes: clean conversational (`grok`) and detailed debugging (`grok-detailed`).

## Quick Start

```bash
# Set your API key
echo 'GROK_API_KEY=your-key-here' >> ~/.claude/.env

# Copy to PATH
cp grok grok-detailed ~/.local/bin/
chmod +x ~/.local/bin/grok ~/.local/bin/grok-detailed

# Run
grok           # minimal mode
grok-detailed  # verbose mode
```

## Modes

### `grok` - Minimal Mode

Clean back-and-forth conversation. Shows only:
- Your input
- Tool calls (one line each)
- Agent response

```
> what's the weather in sf
[web_search] weather san francisco today

It's 62°F and partly cloudy in San Francisco.

>
```

### `grok-detailed` - Detailed Mode

Full visibility into agent reasoning:
- Session ID, cost, context usage, tool count
- Task description per iteration
- Goal and exit conditions
- Progress tracking (completed/current/remaining)
- Thinking/reasoning
- Confidence score
- Timestamps

Use this mode when debugging or understanding agent behavior.

---

## How It Works

### The Structured JSON Contract

The agent is instructed to always respond with valid JSON. This creates a predictable contract between the model and the harness.

```json
{
  "tool_calls": [
    {"tool": "tool_name", "args": {"param": "value"}}
  ],
  "response": "Final answer to user (null if calling tools)",
  "done": true
}
```

#### Fields

| Field | Type | Description |
|-------|------|-------------|
| `tool_calls` | array | Tools to execute. Empty `[]` when giving final response |
| `response` | string/null | The answer to show user. `null` while tools are running |
| `done` | boolean | `false` = keep iterating, `true` = conversation turn complete |

#### Detailed Mode Additional Fields

`grok-detailed` uses an extended schema:

```json
{
  "thinking": "Brief strategic reasoning",
  "task": "Current action (5 words max)",
  "goal": "High-level objective",
  "exit_conditions": ["Condition 1", "Condition 2"],
  "progress": {
    "completed": ["Done items"],
    "current": "Working on now",
    "remaining": ["Still to do"]
  },
  "blockers": null,
  "tool_calls": [...],
  "response": "...",
  "confidence": 0-100,
  "done": true
}
```

### The Harness Loop

```
┌─────────────────────────────────────────────────────────┐
│                    USER INPUT                           │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              BUILD MESSAGES ARRAY                       │
│  [system_prompt, ...history, user_input]                │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              CALL GROK API                              │
│  POST /v1/chat/completions                              │
│  response_format: {type: "json_object"}                 │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              PARSE JSON RESPONSE                        │
└─────────────────────┬───────────────────────────────────┘
                      │
            ┌─────────┴─────────┐
            │                   │
            ▼                   ▼
     tool_calls?            response?
            │                   │
            ▼                   ▼
┌───────────────────┐   ┌───────────────────┐
│  EXECUTE TOOLS    │   │  DISPLAY RESPONSE │
│  Collect results  │   │  done=true        │
└─────────┬─────────┘   └───────────────────┘
          │
          ▼
┌───────────────────┐
│ APPEND RESULTS    │
│ TO MESSAGES       │
└─────────┬─────────┘
          │
          └──────► LOOP BACK TO API CALL
```

### Key Design Decisions

1. **JSON Mode** - Using `response_format: {type: "json_object"}` forces valid JSON output
2. **Iteration Loop** - Agent keeps calling tools until `done=true`
3. **Tool Results as User Messages** - Tool outputs are appended as user messages so the agent sees them
4. **System Prompt Contract** - The JSON schema is defined in the system prompt

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

In the main loop where tools are executed:

```bash
case "$tn" in
    bash) tr=$(tool_bash "$(echo "$ta"|jq -r '.command')") ;;
    web_search) tr=$(tool_web_search "$(echo "$ta"|jq -r '.query')") ;;
    # ... existing tools ...
    my_tool) tr=$(tool_my_tool "$(echo "$ta"|jq -r '.arg1')" "$(echo "$ta"|jq -r '.arg2')") ;;
    *) tr="Unknown tool" ;;
esac
```

#### 3. Add to System Prompt

Update the TOOLS section in the system prompt:

```
TOOLS:
- bash: {"command": "..."}
- web_search: {"query": "..."}
- my_tool: {"arg1": "...", "arg2": "..."}  # Add your tool
```

#### Example: Adding a Calculator Tool

```bash
# 1. Function
tool_calc() {
    local expr="$1"
    echo "$expr" | bc -l 2>/dev/null || echo "Error"
}

# 2. In execute switch
calc) tr=$(tool_calc "$(echo "$ta"|jq -r '.expression')") ;;

# 3. In system prompt
- calc: {"expression": "..."} - Evaluate math expression
```

### Tool Guidelines

- **Return strings** - Tool output is concatenated into messages
- **Truncate output** - Use `| head -N` to limit large outputs
- **Handle errors** - Return error messages, don't crash
- **Be stateless** - Tools shouldn't depend on previous tool calls
- **Timeout long operations** - Wrap slow commands

---

## Skills System

Both modes support a skills system for domain-specific knowledge.

```
~/.local/bin/skills/
├── coding.md
├── research.md
└── writing.md
```

Skills are markdown files that get injected into the context when relevant. The agent can read them with `read_file`.

---

## Configuration

### Environment Variables

```bash
# Required
GROK_API_KEY=xai-...

# In script (edit to customize)
MODEL="grok-4-1-fast"      # or grok-3-fast for speed
COST_LIMIT=10.00           # Max spend per session (detailed mode)
COMPACT_THRESHOLD=75       # Auto-compact at N% context (detailed mode)
```

### Commands

| Command | Description |
|---------|-------------|
| `/clear` | Reset conversation |
| `/exit` | Quit |
| `/help` | Show commands |
| `/cd DIR` | Change working directory (detailed) |
| `/sessions` | List saved sessions (detailed) |
| `/resume ID` | Resume session (detailed) |
| `/compact` | Force context compaction (detailed) |
| `/cost N` | Set cost limit (detailed) |

---

## API Reference

### Request

```bash
curl https://api.x.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $XAI_API_KEY" \
  -d '{
    "messages": [...],
    "model": "grok-4-1-fast",
    "temperature": 0.5,
    "max_tokens": 4000,
    "response_format": {"type": "json_object"}
  }'
```

### Response

```json
{
  "choices": [{
    "message": {
      "content": "{\"tool_calls\":[],\"response\":\"...\",\"done\":true}"
    }
  }],
  "usage": {
    "prompt_tokens": 1234,
    "completion_tokens": 567
  }
}
```

---

## License

MIT
