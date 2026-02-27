---
name: deep-research
description: Multi-agent parallel research with upfront scoping discussion. Clarifies what to research and how before dispatching parallel agents.
argument-hint: "<topic>" [--depth quick|normal|deep] [--agents 1-10] [--skip-discuss]
---

# Deep Research

> Scope the research through discussion, then dispatch parallel agents to investigate.

## Quick Reference

| Option | Values | Default |
|--------|--------|---------|
| `topic` | Required | - |
| `--depth` | quick / normal / deep | normal |
| `--agents` | 1-10 | 5 |
| `--skip-discuss` | Skip scoping, go straight to research | off |

## Usage

```bash
/deep-research "delivery aggregation platforms for SMBs"
/deep-research "RAG best practices" --depth deep
/deep-research "React state management" --skip-discuss
```

## Behavior

### Step 0: Scoping Discussion (unless --skip-discuss)

Before dispatching any agents, have a focused conversation to shape the research. This prevents wasted effort and ensures agents look in the right places.

**0a. Restate understanding.** Briefly summarize what you think the user wants to research and why it matters. Ask: "Is this right, or should I adjust the framing?"

**0b. Clarify intent and angle.** Ask three focused questions:

1. **"What's the goal?"** — Options: learning, decision-making, building a case/argument, writing a report, competitive intelligence, due diligence
2. **"Is there a thesis or angle to support?"** — e.g. "prove this market is investable", "find reasons NOT to build this", "justify using Nash over building in-house". If yes, agents will be directed to find supporting evidence, counterarguments, and data points that strengthen or challenge the thesis.
3. **"Who's the audience?"** — e.g. investor, client, internal team, yourself. This shapes the tone and type of evidence gathered (investors want TAM/comps/exits, clients want ROI/case studies, internal teams want technical feasibility).

If the user has a thesis, all research directions should be framed to serve that thesis — finding supporting data, addressing likely objections, and surfacing comparable success stories.

**0c. Surface assumptions.** List 3-5 assumptions you're making about the research scope:
- What domain/industry does this apply to?
- Are we looking for technical depth, market landscape, or both?
- What time horizon matters (cutting edge vs established)?
- Any known players or prior research to build on?

Ask the user to confirm, correct, or add to these.

**0d. Propose research directions.** Based on the goal, thesis, audience, and discussion, propose 3-7 specific research directions (replacing the defaults). For each direction, explain in one line what the agent will look for and why.

When a thesis is provided, shape directions to build the argument. For example, if the thesis is "SMB delivery aggregation is investable":
```
1. Market size and growth — TAM/SAM/SOM for SMB last-mile, growth projections
2. Comparable exits and funding — Who raised, at what valuation, who acquired whom
3. Demand signals — SMB pain points, willingness to pay, adoption trends
4. Competitive whitespace — Where incumbents are weak, what's underserved
5. Unit economics validation — Proven pricing models, margin benchmarks
6. Risks and objections — What an investor would push back on, with rebuttals
```

Example:
```
Proposed directions for "delivery aggregation platforms for SMBs":

1. Direct competitors — Who does multi-carrier aggregation for SMBs today? (Nash, Shipday, etc.)
2. Failed attempts — Who tried this and shut down? What killed them?
3. Pricing models — What pricing structures have been validated in market?
4. Tech/UX patterns — What onboarding, dashboard, and integration patterns work?
5. Market sizing — How big is the SMB last-mile delivery market?
```

Ask: "Want to adjust these directions, add any, or remove any?"

**0e. Confirm and proceed.** Once the user approves the directions, move to Step 1.

### Step 1: Finalize Directions

Use the agreed directions from the scoping discussion (or defaults if --skip-discuss):

Default directions (used when skipping discussion):
1. GitHub projects
2. HuggingFace models
3. Papers / articles
4. Competitors
5. Best practices

### Step 2: Dispatch Agents

Launch one agent per research direction in parallel:
```
Task(
    prompt="Research {direction} for {topic}. Context: {scoping_notes}",
    subagent_type="general-purpose",
    model="sonnet",
    run_in_background=True
)
```

Include any scoping context from the discussion so agents search smarter.

### Step 3: Collect Findings

Wait for all agents, then compile findings into a structured report organized by direction.

### Step 4: Review and Iterate

Present findings to the user. Ask:
- "Any directions that need deeper digging?"
- "New questions that came up from these findings?"

If follow-up needed → refine directions → Back to Step 2 for another wave.

### Step 5: Final Report

Compile all waves into a single comprehensive report with:
- Executive summary
- Findings by direction
- Key insights and recommendations
- Sources with links
- Open questions for further research

## Example Flow

```
User: /deep-research "SMB delivery aggregation"

Claude: Let me make sure I understand what we're researching.

You want to research multi-carrier delivery aggregation platforms
targeting small/medium businesses. Is this right?

Three quick questions to shape the research:

1. What's the goal? Learning, building a case, competitive intel?
2. Is there a thesis or angle? e.g. "this market is investable"
3. Who's the audience? Investor, client, internal team?

User: Build a case for an investor to put money into DNC's
delivery aggregation platform

Claude: Got it — building an investment case. Audience is an investor.
Thesis: "SMB delivery aggregation is a fundable opportunity and
DNC is positioned to capture it."

Assumptions:
- US market focus (starting with El Paso)
- DNC's angle is vertical SMBs (flower shops, pizza, auto parts)
- Investor cares about TAM, comps, unit economics, defensibility
- We should also surface objections an investor would raise

Proposed research directions:

1. Market size — TAM/SAM for SMB last-mile delivery, growth rate
2. Comparable funding and exits — Nash ($27M a16z), Deliverect
   ($1.9B), OneRail ($110M) — who raised, at what stage
3. Demand signals — SMB delivery pain points, adoption data
4. Competitive whitespace — Where Nash/Shipday are weak, what
   DNC's vertical focus unlocks
5. Unit economics — Proven pricing, margin benchmarks, path to
   profitability
6. Investor objections — Common pushback and rebuttals

Ready to dispatch 6 agents? Or want to adjust?

User: go

Claude: Dispatching 6 agents...
[research proceeds with investor-focused framing]
```

## Notes

- Uses `haiku` model for agents to save cost
- Max 5 agents per wave (increase with --agents)
- Deep mode loops until user is satisfied
- Scoping discussion typically takes 2-3 exchanges before dispatching
- Agents receive scoping context so they search smarter than generic queries

## Related Skills

- **research-scout** — Single-agent research (no parallel dispatch)

---

*Based on Infinite Gratitude by [Washin Village](https://washinmura.jp)*
