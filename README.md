# Clafer skills

v0.5.7

Agent skills for modeling in [Clafer](https://github.com/gsdlab/clafer), a lightweight modeling language for structure and behaviour.

| Skill | What it does |
|---|---|
| [clafer-modeling](skills/clafer-modeling/SKILL.md) | Write, structure, fix and check Clafer models (`.cfr`), static or temporal, and run them with clafer, claferIG and chocosolver |

The skills are for Clafer 0.5.7. They expect `clafer`, `claferIG` and `chocosolver` on the `PATH`; see the [Clafer Tools installation](https://github.com/gsdlab/clafer#installation).

## Installation

### Claude Code

```
/plugin marketplace add gsdlab/clafer-skills
/plugin install clafer@clafer
```

Or from a shell:

```
claude plugin marketplace add gsdlab/clafer-skills
claude plugin install clafer@clafer
```

The skill is then available as `/clafer:clafer-modeling`, and Claude uses it on its own for Clafer models. To use the skills of a given release, add the marketplace at its tag: `gsdlab/clafer-skills#0.5.7`.

To update:

```
claude plugin marketplace update clafer
claude plugin update clafer@clafer
```

### Other agents

A skill is a folder with a `SKILL.md` file. Copy `skills/clafer-modeling` to the skills folder of your agent, e.g., `~/.claude/skills/` for Claude Code without the plugin, or upload it as a skill to claude.ai.

## Known problems

The skills describe problems of the tools that a model may run into, with a workaround and a link to the issue. They are removed when the issue is fixed.

## License

MIT, see [LICENSE](LICENSE).
