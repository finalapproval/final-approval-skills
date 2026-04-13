# FinalApproval Skills

Skills for adding human-in-the-loop approval to AI agent workflows.

## Install

### Claude Code (recommended)

```
/plugin marketplace add pmccurley87/final-approval-skills
/plugin install create-channel@final-approval-skills
```

### All tools (npx)

Auto-detects Claude Code, Cursor, Windsurf, or Codex and installs in the right format:

```bash
npx final-approval-skills
```

Global install (available in all projects):

```bash
npx final-approval-skills --global
```

## Available Skills

| Skill | Description |
|---|---|
| `create-channel` | Create a FinalApproval channel to gate an agent action behind human approval |

## Contributing a Skill

1. Fork this repo
2. Create your skill directory under `skills/your-skill-name/`
3. Add a `SKILL.md` with YAML frontmatter + instructions
4. (Optional) Add a `templates/` directory for template files
5. Open a PR — we'll set up the marketplace plugin entry

### SKILL.md format

```yaml
---
name: my-skill-name
description: Short description of what the skill does
argument-hint: [what arguments the skill accepts]
---

# Skill Title

Instructions for the AI to follow when this skill is invoked.
```

## License

MIT
