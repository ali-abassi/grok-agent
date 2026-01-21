# Skill Creation

When user asks you to create a skill, follow this format:

## Skill File Structure

Skills are markdown files saved to the skills directory.

```
~/.local/bin/skills/[skill-name].md
```

## Template

```markdown
# [Skill Name]

[Brief description of what this skill does]

## When to Use

[Conditions that should trigger loading this skill]

## Instructions

[Step-by-step instructions for the agent to follow]

## Examples

[Example inputs and expected outputs]

## Tools to Use

[Which tools are most relevant for this skill]
```

## Steps to Create

1. Ask user what the skill should do
2. Write the skill file using write_file to ~/.local/bin/skills/[name].md
3. Confirm creation

## Example Skills

- `research.md` - Deep research with multiple searches
- `coding.md` - Code generation patterns
- `writing.md` - Content writing guidelines
- `analysis.md` - Data analysis workflows
