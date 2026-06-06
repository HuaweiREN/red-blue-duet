# Red-Blue-Duet

A **multi-agent structured debate skill** for [Claude Code](https://claude.ai/code). When you face a decision with no clear answer — competing approaches each with real pros/cons — Red-Blue-Duet deploys independent AI debaters to argue each side, then delivers an impartial, evidence-backed judgment.

## What It Does

Instead of asking Claude for a quick opinion (which tends toward shallow consensus), this skill orchestrates a **3-round adversarial debate**:

1. **Phase 0 — Problem Definition**: The moderator analyzes your dilemma, enumerates solution paths (2–3), defines evaluation dimensions, conducts neutral baseline research, and asks you to confirm before proceeding.
2. **Phase 1 — Opening Statements**: Independent AI debaters research and prepare opening arguments, including evidence with cross-validation annotations.
3. **Phase 2 — Three Debate Rounds**: Debaters speak in rotating order (eliminating fixed-position advantage), research new evidence between rounds, challenge each other's claims with fact-checks, and respond to challenges. The moderator checks for topic drift after every round and requires your confirmation to continue.
4. **Phase 3 — Independent Judgment**: A **fresh judge subagent** (with no access to moderator private knowledge) scores each path across 5 weighted dimensions: evidence quality, logical coherence, response to challenges, refutation of opponents, and practical applicability.
5. **Phase 4 — Record Delivery**: A complete debate record is compiled into a structured Markdown file, including the judgment, scoring breakdown, actionable recommendations, and risk mitigations.

## When to Use

| Trigger | Example |
|---------|---------|
| Trade-off decisions | "Should I use Rust or Go for this service?" |
| Architecture choices | "Monolith vs. microservices for our scale?" |
| Technology selection | "React vs. Vue for this project?" |
| Career decisions | "Join a startup or stay at big tech?" |
| Strategy comparison | "Build in-house vs. buy SaaS?" |

### When NOT to Use

- Questions with a single verifiable factual answer — just ask Claude directly
- Purely subjective/preference questions — this isn't a poll
- You just want a quick take — say so, and Claude will skip the debate

## Requirements

- [Claude Code](https://claude.ai/code) CLI or IDE extension
- A Claude.ai account (the skill uses sub-agents for debaters, which consume more tokens than a normal query)

## Installation

### Method 1: Install from GitHub (Recommended)

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/YOUR_USERNAME/red-blue-duet.git ~/.claude/skills/red-blue-duet
```

Then restart Claude Code, or run `/install red-blue-duet` in the Claude Code REPL.

### Method 2: Manual Copy

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills/red-blue-duet

# Copy SKILL.md and references/
cp SKILL.md ~/.claude/skills/red-blue-duet/
cp -r references/ ~/.claude/skills/red-blue-duet/
```

### Method 3: One-Click Install (if hosted on a skill registry)

In Claude Code, run:

```
/install red-blue-duet
```

If the skill is registered in a community skill registry, Claude Code will fetch and install it automatically.

### Verify Installation

After installation, the skill will auto-trigger when you describe a trade-off decision. You can verify it's installed by checking:

```bash
ls ~/.claude/skills/red-blue-duet/
# Should show: SKILL.md  references/
```

## Usage

Simply describe your dilemma to Claude Code. The skill auto-triggers when it detects a decision with multiple viable paths. For example:

```
I need to choose between PostgreSQL and MongoDB for a new analytics platform.
We expect 10TB of data, mixed structured/unstructured, with a team of 3 backend engineers.
```

The moderator will:
1. Analyze your situation and propose distinct solution paths
2. Present the paths and evaluation dimensions for your confirmation
3. Dispatch debaters, run 3 rounds, and deliver a judgment with actionable recommendations

You stay in control throughout — the skill pauses after every round for your feedback and direction.

## File Structure

```
red-blue-duet/
├── SKILL.md                              # Skill definition & debate protocol
├── README.md                             # This file
├── LICENSE                               # MIT License
└── references/
    └── debate-record-template.md         # Template for the final debate record
```

## How It's Different from a Normal Claude Chat

| Normal Chat | Red-Blue-Duet |
|-------------|---------------|
| One agent, one perspective | Multiple agents arguing opposing views |
| May converge on consensus too quickly | Structured adversarial process |
| No systematic evidence validation | Cross-validation annotations, baseline facts |
| Single response | 3-round debate + independent judgment |
| No deliverable | Complete debate record with actionable plan |

## Key Design Decisions

**Sequential speaking with rotation.** Debaters speak one at a time per round, and the speaking order rotates each round. This prevents parallel monologues (each debater attacking a strawman of the other's position) and ensures every debater experiences the advantage and disadvantage of each speaking position.

**Fresh judge subagent.** The moderator accumulates private knowledge during the debate (baseline research, drift assessments). To keep the judgment impartial, Phase 3 spawns a brand-new subagent that sees only the shared baseline facts and the debate record — nothing the moderator learned privately.

**Mandatory user confirmations.** The skill pauses after problem definition and after every debate round. You're never a passive observer; you can course-correct, add context, or end early at any time.

## Token Usage

Expect a full 3-round, 3-path debate to consume significant tokens (comparable to a long, research-heavy coding session). The trade-off is depth: you get a thoroughly examined decision with cited evidence and an independent judgment, not a quick answer.

## License

MIT — see [LICENSE](LICENSE).
