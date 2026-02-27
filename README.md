# ReyemTech Skills

Curated collection of Claude Code skills for ReyemTech workflows. Battle-tested skills used across fractional CTO engagements, client projects, and internal tooling.

## Skills

| Skill | Description |
|-------|-------------|
| [deep-research](deep-research.skill.md) | Multi-agent parallel research with upfront scoping discussion. Clarifies goal, thesis, and audience before dispatching parallel agents. |

## Installation

Copy a skill to your Claude Code skills directory:

```bash
# Single skill
cp deep-research.skill.md ~/.claude/skills/deep-research/SKILL.md

# All skills
for f in *.skill.md; do
  name="${f%.skill.md}"
  mkdir -p ~/.claude/skills/"$name"
  cp "$f" ~/.claude/skills/"$name"/SKILL.md
done
```

## Usage

Once installed, invoke any skill from Claude Code:

```bash
/deep-research "topic to research"
```

## Adding a New Skill

1. Create `skill-name.skill.md` with YAML frontmatter (`name`, `description`, `argument-hint`)
2. Add to the table above
3. Commit and push

## License

MIT
