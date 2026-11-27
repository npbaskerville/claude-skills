# claude-skills

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) of skills by Nicholas Baskerville.

## Plugins

| Plugin | Skills | Description |
|---|---|---|
| [`paper-reading`](plugins/paper-reading) | `academic-paper-reading-assistant` | Turn a technical paper (pure maths, theory, ML or other empirical work; arXiv link/id, PDF and/or LaTeX source) into one reading-copy PDF: a front reading guide (key references, argument graph, reading route, exercises) plus the paper with route sections boxed. |

## Installation

### Claude Code

Install in one step (Claude Code 2.1.275 or later):

```
/plugin install paper-reading --marketplace npbaskerville/claude-skills
```

Or add the marketplace first, then install from it:

```
/plugin marketplace add npbaskerville/claude-skills
/plugin install paper-reading@npbaskerville
```

The skill is then available as `/paper-reading:academic-paper-reading-assistant`, and Claude also uses it automatically when you ask for a reading copy of a paper.

### Without plugins

Each skill is a folder containing a `SKILL.md` under `plugins/<plugin>/skills/`. Copy that folder into `~/.claude/skills/` (personal) or `.claude/skills/` in a project. On claude.ai, zip the folder and upload it under Settings → Capabilities → Skills.

## Layout

```
.claude-plugin/marketplace.json        # marketplace catalog
plugins/<plugin>/.claude-plugin/plugin.json
plugins/<plugin>/skills/<skill>/SKILL.md
```

To add a skill, put it in an existing plugin's `skills/` folder, or create a new plugin folder and add an entry for it to `marketplace.json`. Check the result with `claude plugin validate .` and `claude plugin validate plugins/<plugin>`.

## License

Licensed under the [Apache License, Version 2.0](LICENSE). See [NOTICE](NOTICE).
