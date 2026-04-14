# ReyemTech Skills

Curated collection of Claude Code skills for fractional CTO workflows. Battle-tested across client engagements, pipeline management, and internal tooling. Requires MCP servers for HubSpot, Linear, Google Workspace, and/or MS365 depending on the skill.

## Skills

### Orchestration

| Skill | Description |
|-------|-------------|
| [check-mcp-health](check-mcp-health.skill.md) | Verify all MCP servers are connected; auto-open auth URLs on failure |
| [deep-research](deep-research.skill.md) | Multi-agent parallel research with upfront scoping discussion |
| [draft-email](draft-email.skill.md) | Draft professional HTML emails via Gmail or MS365 MCP with full thread context |
| [generate-contract](generate-contract.skill.md) | Generate contracts from Google Docs templates with auto-filled client details |
| [teardown](teardown.skill.md) | Reverse-engineer any web app's architecture from its frontend |

### HubSpot

| Skill | Description |
|-------|-------------|
| [hubspot-contact](hubspot-contact.skill.md) | Create or update a HubSpot contact and company |
| [hubspot-deal](hubspot-deal.skill.md) | Suggest a HubSpot deal from context signals (confirms before creating) |
| [hubspot-email](hubspot-email.skill.md) | Log an email engagement to HubSpot for actionable emails |
| [hubspot-meeting](hubspot-meeting.skill.md) | Log a meeting to HubSpot with dedup and org-type filtering |
| [hubspot-note](hubspot-note.skill.md) | Log an HTML note to HubSpot with org-type filtering |

### Linear

| Skill | Description |
|-------|-------------|
| [linear-issue](linear-issue.skill.md) | Create, update, or close a Linear issue |
| [linear-assign](linear-assign.skill.md) | Assign or reassign a Linear issue |
| [linear-comment](linear-comment.skill.md) | Add a comment to a Linear issue |
| [linear-label](linear-label.skill.md) | Add or remove labels on a Linear issue |

### QuickBooks

| Skill | Description |
|-------|-------------|
| [qbo](qbo.skill.md) | QuickBooks API integration with managed OAuth — customers, invoices, payments, reports |
| [qbo-cleanup](qbo-cleanup.skill.md) | Audit QBO expenses for duplicates, missing FX rates, and unassigned vendors |

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
