# FinalApproval Skills

Community-built skills for AI coding tools.

## Install

### Claude Code (recommended)

Install from the Claude marketplace:

```
code-reviewer@final-approval-skills
example-greeting@final-approval-skills
```

Enable in your project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "code-reviewer@final-approval-skills": true,
    "example-greeting@final-approval-skills": true
  }
}
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
| `code-reviewer` | Structured code review for staged changes or files |
| `example-greeting` | Generate a greeting message for READMEs or onboarding |

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

### Directory structure for a skill

```
skills/my-skill/
├── SKILL.md              # Required — skill definition
└── templates/            # Optional — reference files
    └── example.md
```

The marketplace plugin structure (`plugins/`) is maintained by the FinalApproval team — contributors only need to add to `skills/`.

### Guidelines

- Keep skills focused — one skill, one job
- Write clear, step-by-step instructions
- Include example output where helpful
- Test with at least one AI tool before submitting

## License

MIT
