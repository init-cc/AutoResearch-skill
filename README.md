# AutoResearch Skill

A Claude Code skill that implements the **autonomous experiment loop** pattern — originally designed by [Andrej Karpathy](https://github.com/karpathy/autoresearch) for LLM training research, generalized here to work for **any optimization task**.

## What It Does

When you say `/autoresearch` or "use autoresearch to help me optimize X", the agent will:

1. **Interview you** — ask structured questions about your optimization target, metric, constraints, and budget
2. **Scaffold a project** — generate `program.md` (research agenda), a fixed evaluation harness, and an editable sandbox file
3. **Run experiments autonomously** — iterating in a loop: modify → commit → run → evaluate → keep/discard → repeat, until you interrupt it

## The Three-File Architecture

| File | Role | Who Touches It |
|------|------|----------------|
| `program.md` | Research agenda — goals, rules, constraints | **Human** (you) |
| `prepare.py` (equiv) | Fixed evaluation harness — test data, metrics | **Nobody** (locked after setup) |
| `train.py` (equiv) | Editable sandbox — code the agent optimizes | **Agent** (autonomous) |

This is the pattern from [karpathy/autoresearch](https://github.com/karpathy/autoresearch), generalized.

## The Experiment Loop

```
LOOP FOREVER:
  1. Read context
  2. Form hypothesis, edit sandbox file
  3. git commit
  4. Run experiment > run.log
  5. grep metric from run.log
  6. If improved → keep commit, advance branch
  7. If worse → git reset
  8. Record to results.tsv
  9. Repeat (never stop until interrupted)
```

## Example Use Cases

- "Use autoresearch to optimize my web crawler's speed"
- "Use autoresearch to tune PostgreSQL query performance"
- "Use autoresearch to find the best hyperparameters for my ML model"
- "Use autoresearch to optimize my React app's bundle size"
- "Use autoresearch to improve my trading strategy's Sharpe ratio"

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/init-cc/AutoResearch-skill.git \
  ~/.claude/skills/autoresearch
```

Or manually:

```bash
mkdir -p ~/.claude/skills/autoresearch
cp -r .claude-plugin skills ~/.claude/skills/autoresearch/
```

Then restart Claude Code, and the skill will be available as `/autoresearch`.

## Key Design Principles

1. **Single editable file** — agent only touches one file, diffs stay reviewable
2. **Fixed evaluation** — metric defined in locked file, prevents "cheating"
3. **Git as state machine** — commits are experiments, resets are discards
4. **Fixed budget per experiment** — makes all results directly comparable
5. **Never-stop autonomy** — agent runs until human interrupts, no asking "should I continue?"
6. **Simplicity bias** — simpler solutions preferred; deleting code that maintains performance is a win

## File Structure

```
autoresearch/
├── .claude-plugin/
│   └── plugin.json          # Skill metadata
└── skills/
    └── autoresearch/
        └── SKILL.md         # Full skill instructions
```

## License

MIT — same as the original [karpathy/autoresearch](https://github.com/karpathy/autoresearch).

## Credits

Pattern by [Andrej Karpathy](https://github.com/karpathy/autoresearch). Skill packaging by [init-cc](https://github.com/init-cc).
