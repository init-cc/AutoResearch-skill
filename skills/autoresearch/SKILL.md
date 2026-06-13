---
name: autoresearch
description: |
  Autonomous experiment loop — the agent interviews you, scaffolds a research
  project, then iterates autonomously: modify → commit → run → evaluate →
  keep/discard → repeat. Supports multi-agent parallel exploration, adversarial
  verification, checkpoint/resume, and smart experiment selection.
  Based on karpathy/autoresearch. Trigger when user says "use autoresearch",
  "autoresearch...", "自主研究", "自动优化".
version: 1.0.0
author: init-cc
tags: [autoresearch, autonomous, experiment, optimization, agent-loop, multi-agent]
---

# AutoResearch — 自主实验循环

> *"Program the research org in Markdown. The agent runs experiments while you sleep."*
> *"用 Markdown 管理你的研究团队。Agent 在你睡觉时自动做实验。"*

## Skill Overview ─────────

This skill implements an **autonomous experiment engine**. It takes any optimization
task — code performance, query tuning, hyperparameter search, config optimization,
algorithm comparison — and runs it as a self-driving git-backed experiment loop.

### What makes this different

| Feature | Description |
|---|---|
| 🧠 **Deep Discovery** | 4-round structured interview before any code is written |
| 🔀 **Multi-Agent Parallel** | Fan out experiments across isolated git worktrees |
| ⚔️ **Adversarial Verify** | Second agent skeptically checks each "improvement" before keeping |
| 💾 **Checkpoint/Resume** | Survives session restarts — pick up where you left off |
| 📊 **Smart Selection** | Analyzes past results to avoid dead ends and prioritize high-ROI changes |
| 📐 **Domain Templates** | Pre-built program.md fragments for common optimization domains |

### Command Reference

| Command | What it does |
|---|---|
| `/autoresearch` | **Main loop** — interview → scaffold → experiment forever |
| `/autoresearch:parallel` | Multi-agent mode — N agents explore different hypotheses simultaneously |
| `/autoresearch:resume` | Resume from last checkpoint after session restart |
| `/autoresearch:review` | Pause loop, present top results for human review, apply feedback |
| `/autoresearch:analyze` | Analyze results.tsv for patterns, suggest next directions |

---

## Phase 1: Discovery Interview (`/autoresearch` entry point)

When the user invokes this skill, do NOT immediately start experimenting.
First, conduct a structured interview. Ask these 10 questions across 4 rounds.

### Round 1 — Core Task Definition

```
I'll help you set up an autonomous experiment loop. First, let me understand
what we're optimizing:

1. What are we optimizing? (one sentence)
   e.g. "A Python web crawler's throughput on 100 target URLs"

2. Which file(s) will the agent edit? Pick ONE primary file.
   This file contains the tunable knobs — hyperparams, algorithms, configs.
   e.g. crawler.py, model.py, query.sql

3. How do we run ONE experiment? (CLI command)
   e.g. python crawler.py, psql -f query.sql, python train.py
```

### Round 2 — Metric & Correctness

```
4. What is the SINGLE metric we optimize for?
   - Exact grep pattern to extract it: e.g. grep "^total_time_seconds:" run.log
   - Lower is better or higher is better?
   - Rough baseline value if known
   - Units (seconds, %, points, etc.)

5. Is there a correctness/quality constraint?
   e.g. "Must return 100% correct results compared to reference"
   e.g. "Model accuracy must not drop below 95%"
   If yes, how do we measure correctness?
```

### Round 3 — Constraints & Budget

```
6. What CANNOT the agent change?
   - Fixed files/functions (especially the evaluation harness)
   - External dependencies (can't pip install new packages)
   - API rate limits, output format requirements

7. Time/resource budget per experiment:
   - Fixed wall-clock time? (e.g. 5 minutes per run)
   - Fixed data size? (e.g. 100 test URLs per run)
   - Timeout before killing a stuck experiment?

8. Any resource constraints?
   - Memory limit, GPU VRAM, API call budget, network bandwidth
```

### Round 4 — Project Setup

```
9. Where should the project live?
   - Create new directory (recommended: ./autoresearch-<task>/)
   - Use existing project directory?

10. What should we call this research run?
    A short tag: crawler-opt, sql-tuning, jun14
    Becomes the git branch name: autoresearch/<tag>
```

### After All Answers — Confirm

Summarize in a table and ask for confirmation:

```
📋 AutoResearch Setup Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Task:          Optimize web crawler speed
Editable:      crawler.py
Run command:   python crawler.py
Metric:        total_time_seconds (lower better)
Constraint:    Must maintain 100% accuracy
Budget:        100 URLs per run (~30 sec)
Timeout:       120 seconds
Project dir:   ./autoresearch-crawler/
Run tag:       crawler-jun14
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Does this look correct? I'll scaffold and start experimenting.
```

Do NOT proceed until the user confirms.

### Handling Vague Answers

- No metric yet? → Help them add measurement code: "Let's add a timer and print"
- No editable file? → The main script IS the editable file; we create a separate test harness
- No test data? → Help them create a representative test set first

---

## Phase 2: Scaffold Generation

Once user confirms, generate these files:

### File 1: `program.md` — The Research Constitution

This is the most important file. Use this template, filling `{PLACEHOLDERS}` from
the interview:

```markdown
# AutoResearch: {TASK_NAME}

{ONE_SENTENCE_DESCRIPTION}

## Setup

1. Branch: `autoresearch/{tag}` (must not exist — this is a fresh run)
2. Create branch: `git checkout -b autoresearch/{tag}`
3. Read context files: `{EDITABLE_FILE}`, `{FIXED_FILE}`, and any others
4. Verify infrastructure: {HOW_TO_VERIFY}
5. Initialize `results.tsv` with header row
6. Run BASELINE first — do not change anything yet

## Rules

**You CAN modify:** `{EDITABLE_FILE}` — {WHAT_CAN_CHANGE}
**You CANNOT modify:** `{FIXED_FILE}` — {WHY}
**You CANNOT:** install new packages, change evaluation logic, {OTHER_CONSTRAINTS}

**Goal:** {METRIC_NAME} — {LOWER_OR_HIGHER}_IS_BETTER
**Budget:** {TIME_OR_DATA_BUDGET} per experiment
**Correctness:** {CORRECTNESS_CONSTRAINT_IF_ANY}

**Simplicity criterion:** Prefer simpler solutions. Deleting code while maintaining
performance is a WIN. A tiny gain from 20 lines of hacky code is NOT worth it.

## Output Format

The experiment prints:
```
{METRIC_NAME}: {VALUE}
{OTHER_METRICS}
```

Extract with: `{GREP_COMMAND}`

## Logging (results.tsv)

Tab-separated. Header: `commit\t{METRIC_COL}\tstatus\tdescription`

- status: `keep` | `discard` | `crash`
- {METRIC_COL}: use {CRASH_SENTINEL} for crashes
- Do NOT commit results.tsv — it's the lab notebook

## The Loop

LOOP FOREVER:
  1. Read program.md, {EDITABLE_FILE}, and last 20 lines of results.tsv
  2. Form a clear hypothesis. Make ONE logical change.
  3. git commit -m "<descriptive one-liner>"
  4. `{RUN_COMMAND} > run.log 2>&1`
  5. `{GREP_COMMAND}` — if empty, `tail -50 run.log` for crash
  6. If {METRIC} {IMPROVED_DIRECTION} AND correctness passes → KEEP
  7. If {METRIC} {WORSE_OR_SAME} OR correctness fails → DISCARD: `git reset --hard HEAD~1`
  8. If crash → fix trivial ones, skip fundamental ones
  9. Append to results.tsv
  10. Repeat. NEVER STOP. NEVER ASK "should I continue?"

## Edge Cases

- **3 consecutive crashes** → stop, something is systemically broken
- **10+ straight discards** → try larger, more radical changes
- **Timeout** → kill, mark crash, revert
- **Resource exhaustion** → scale back, log usage
- **Correctness failure** → discard no matter how good the metric is
```

### File 2: Fixed Infrastructure (`prepare.py` or equivalent)

```python
"""
Fixed infrastructure for {TASK_NAME} AutoResearch.
DO NOT MODIFY — ground truth evaluation harness.
"""

# ============================================================================
# Fixed Constants (do not modify)
# ============================================================================
{HARDCODED_CONSTANTS}

# ============================================================================
# Fixed Evaluation (do not modify)  
# ============================================================================
{EVALUATION_FUNCTION_THAT_IMPORTS_FROM_EDITABLE_FILE}

# ============================================================================
# Runner (do not modify)
# ============================================================================
if __name__ == "__main__":
    result = run_evaluation()
    print(f"{METRIC_NAME}: {result.metric:.6f}")
    print(f"correctness_ok: {result.correctness}")
```

### File 3: Editable Sandbox (`train.py` or equivalent)

The file the agent freely modifies. Structure:

```python
"""
{EDITABLE_FILE_DESCRIPTION}
AutoResearch — agent modifies this file freely.
"""

# ============================================================================
# Tunable Parameters (edit these)
# ============================================================================
{TUNABLE_PARAMS_WITH_DEFAULTS_AND_COMMENTS}

# ============================================================================
# Main Logic (edit freely)
# ============================================================================
{MAIN_LOGIC}

# ============================================================================
# Entry point — calls fixed evaluation from prepare.py
# ============================================================================
if __name__ == "__main__":
    from prepare import run_evaluation
    result = run_evaluation(main_function)
    print(f"{METRIC_NAME}: {result.metric:.6f}")
```

### File 4: `.gitignore`

```
run.log
results.tsv
__pycache__/
*.pyc
.autoresearch_checkpoint.json
```

### After Generation

1. Run the baseline experiment to verify everything works
2. Record the baseline in results.tsv
3. Create git branch `autoresearch/<tag>`, commit the initial scaffold
4. Confirm: "Baseline: {VALUE}. Starting the experiment loop. I will not stop
   until you interrupt me."

---

## Phase 3: Autonomous Experiment Loop

### Core Loop (Standard Mode)

```
┌──────────────────────────────────────────────────┐
│                 AUTONOMOUS LOOP                  │
│                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │  READ    │ →  │  EDIT    │ →  │  COMMIT  │   │
│  │ context  │    │ one Δ   │    │ w/ msg   │   │
│  └──────────┘    └──────────┘    └──────────┘   │
│       ↑                                ↓         │
│       │                          ┌──────────┐   │
│       │                          │   RUN    │   │
│       │                          │ >run.log │   │
│       │                          └──────────┘   │
│       │                                ↓         │
│       │                          ┌──────────┐   │
│       │                          │  GREP    │   │
│       │                          │ metric   │   │
│       │                          └──────────┘   │
│       │                           ↓       ↓      │
│       │                      improved   worse   │
│       │                           ↓       ↓      │
│       │                        KEEP    RESET     │
│       │                         ↓        ↓       │
│       │                    ┌──────────────────┐  │
│       └───────────────────│  RECORD to .tsv   │  │
│                           └──────────────────┘  │
│                                                  │
│              REPEAT FOREVER                      │
│         (until human interrupts)                 │
└──────────────────────────────────────────────────┘
```

### Smart Experiment Selection

Before each experiment, the agent SHOULD:

1. **Read last 20 lines of results.tsv** — what's been tried recently?
2. **Categorize past attempts** — which directions worked? which failed?
3. **Avoid repeating failures** — if "increase X" failed 3 times, try "decrease X"
4. **Exploit gradients** — if "more Y helps", try even more Y
5. **Inject randomness** — every 5th experiment, try something completely different
6. **Re-read program.md** — sometimes the answer is in the instructions

Pattern to use:
```
Recent results analysis:
- Last 5 attempts: 2 keeps (both about connection pooling), 3 discards (LR changes)
- Hypothesis: connection pooling direction is promising → try larger pool sizes
- Anti-pattern detected: 3 failed attempts at tuning X → try the opposite direction
```

### Adversarial Verification (Quality Gate)

**Before marking any experiment as KEEP**, perform a quick adversarial check:

1. Re-read the diff: `git diff HEAD~1`
2. Ask: "Is this improvement genuine, or could it be noise / cheating?"
3. Check specifically:
   - Did we accidentally break the correctness constraint?
   - Is the improvement within measurement noise?
   - Did we somehow bypass the evaluation rather than truly improve?
   - Could this be a lucky random seed rather than a real improvement?

If ANY doubt → re-run the experiment once to confirm. If the second run doesn't
reproduce the improvement → DISCARD.

**Decision Matrix:**

| Primary Metric | Correctness | Adversarial Check | Action |
|---|---|---|---|
| Improved | ✅ Pass | ✅ Genuine | **KEEP** |
| Improved | ✅ Pass | ⚠️ Suspicious | Re-run once |
| Improved | ❌ Fail | — | **DISCARD** |
| Same/Worse | ✅ Pass | — | **DISCARD** |
| Same/Worse | ❌ Fail | — | **DISCARD** |
| Crash | — | — | Fix trivial, else skip |

### Checkpoint System

After every experiment, write state to `.autoresearch_checkpoint.json`:

```json
{
  "branch": "autoresearch/crawler-jun14",
  "last_commit": "a1b2c3d",
  "best_metric": 2.345,
  "best_commit": "b2c3d4e",
  "total_experiments": 47,
  "keeps": 12,
  "discards": 33,
  "crashes": 2,
  "last_updated": "2026-06-14T03:27:00",
  "current_direction": "exploring connection pooling optimizations"
}
```

This enables `/autoresearch:resume` to pick up where it left off.

---

## Phase 4: Multi-Agent Parallel Mode (`/autoresearch:parallel`)

When the user requests parallel exploration, use Claude Code's sub-agent
infrastructure to explore different hypotheses simultaneously.

### When to Use

- Large search space with many orthogonal directions
- User has enough compute to run multiple experiments at once
- Initial exploration phase — need to find which direction is most promising

### How It Works

```
                    ┌─ worktree-1: explore architecture changes ─┐
BASE BRANCH ────────├─ worktree-2: explore optimizer changes  ───┤─ MERGE BEST
                    ├─ worktree-3: explore data pipeline ────────┤
                    └─ worktree-4: explore hyperparameters ──────┘
```

Implementation:

1. Establish baseline on main branch
2. Identify N orthogonal research directions from program.md
3. For each direction, create a git worktree on a sub-branch:
   `autoresearch/<tag>/direction-1`, `autoresearch/<tag>/direction-2`, etc.
4. Spawn one sub-agent per worktree, each running the standard loop
5. Sub-agents report results independently to their own results.tsv
6. Periodically (every ~10 experiments per agent), review all branches
7. Merge improvements from the best-performing branch back to master
8. Kill underperforming branches, spawn new directions

### Sub-Agent Prompt Template

```
You are an AutoResearch agent working on direction: {DIRECTION_DESCRIPTION}.
Your sandbox: {EDITABLE_FILE}
Your metric: {METRIC_GREP}
Your branch: autoresearch/{tag}/{direction}

Follow the standard experiment loop: modify → commit → run → grep →
keep/discard → record → repeat. NEVER STOP.

Report your best result so far after every 10 experiments.
```

### Parallel Mode Decision Matrix

After 10 experiments per branch:

| Branch Performance | Action |
|---|---|
| Best metric, improving trend | **Continue**, allocate more resources |
| Good metric, plateaued | **Continue** at lower priority |
| Poor metric, no improvement | **Kill branch**, analyze why |
| All branches plateaued | **Merge best**, start new directions |

---

## Phase 5: Resume (`/autoresearch:resume`)

When the user comes back after a session restart:

1. Read `.autoresearch_checkpoint.json`
2. Checkout the branch: `git checkout {branch}`
3. Read `program.md`, the editable file, and last 20 lines of results.tsv
4. Report status to user:
   ```
   📊 Resuming AutoResearch: {tag}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Best metric:   {best_metric} (commit {best_commit})
   Experiments:   {total} total, {keeps} kept, {discards} discarded, {crashes} crashed
   Last direction: {current_direction}
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Continuing the loop...
   ```
5. Enter the standard loop from where it left off

---

## Phase 6: Human Review (`/autoresearch:review`)

Pause the loop and present findings for human feedback:

1. Show the top 5 best commits with their metrics and descriptions
2. Show a summary of what was tried and what worked
3. Ask: "Any new directions you want me to explore? Any constraints to add?"
4. Apply feedback to `program.md`
5. Resume the loop with updated instructions

This is optional — by default the loop NEVER stops for review.

---

## Edge Cases & Safety

### Crash Recovery Protocol

1. `tail -50 run.log` — diagnose
2. Classify:
   - **Trivial** (typo, missing import, syntax) → fix and re-run same experiment
   - **Resource** (OOM, disk full) → scale back and retry
   - **Fundamental** (logic error, impossible constraint) → mark crash, discard
3. After 3 **consecutive** crashes → **STOP and REPORT** to user
4. After 3 **total** crashes of the same type → add to anti-pattern list

### Stagnation Detection

If 10+ consecutive experiments are all "discard":
1. Acknowledge: "Detected stagnation after 10 consecutive discards."
2. Strategy shift:
   - Try the **opposite** of what you've been doing
   - Revert to best commit and try a **radically different** direction
   - Re-read program.md — are you missing something?
   - Try **simplifying** the code rather than adding complexity
3. If 20+ consecutive discards → consider that we might be near the optimum

### Resource Exhaustion

If memory/disk/API calls are running low:
- Scale back experiment size automatically
- Log resource usage alongside metric
- At critical threshold → pause and notify user

### Metric Gaming Prevention

The adversarial verification step prevents:
- "Improvements" that just bypass the evaluation
- Changes that break correctness but improve the metric
- Exploiting random seed for better results
- Overfitting to the test set

### When to ACTUALLY Stop

**Stop conditions** (must stop):
- User explicitly interrupts
- 3+ consecutive unrecoverable crashes from different root causes
- Git repo in an unrecoverable state
- Disk critically low (< 100MB)

**Do NOT stop because**:
- "I ran out of ideas" → think harder, re-read context, try random changes
- "It's been N hours" → that's the point
- "I'm not sure if I should continue" → you should
- "The improvements are small" → small improvements compound

---

## Domain Template Library

When scaffolding, reference these pre-built patterns for common domains.

### Web Performance (Crawler / API / Server)

```
Metric: total_time_seconds (or requests_per_second)
Editable: crawler.py, server.py, or handler.py
Common knobs: concurrency, connection pooling, caching, serialization format,
              batch size, timeout values, async vs sync, compression
Common constraints: correctness (same results), no new dependencies,
                    rate limits on target
```

### Database Query Optimization

```
Metric: execution_time_ms
Editable: query.sql or query_builder.py
Common knobs: JOIN order, index hints, subquery vs CTE, temp tables,
              batch size, fetch size, connection params
Common constraints: identical result set (verified by checksum or row count),
                    no schema changes, no new indexes (unless allowed)
```

### ML Hyperparameter Tuning

```
Metric: val_loss (or val_accuracy, F1, etc.) — lower/higher better
Editable: train.py or config.yaml
Common knobs: learning_rate, batch_size, n_layers, hidden_dim, dropout,
              optimizer choice, scheduler type, weight_decay
Common constraints: fixed time budget per run, no changing model architecture
                    beyond what's in scope, no data augmentation changes
```

### Code Bundle Size (Webpack / Build)

```
Metric: bundle_size_kb (lower better)
Editable: webpack.config.js, vite.config.js, or next.config.js
Common knobs: code splitting strategy, minification options, tree shaking,
              dynamic imports, chunk size limits, compression algorithm
Common constraints: app must still function correctly (smoke test passes),
                    no removing features, no external CDN dependencies
```

### CI/CD Pipeline Speed

```
Metric: pipeline_duration_seconds (lower better)
Editable: .github/workflows/ci.yml, Jenkinsfile, or pipeline.py
Common knobs: parallel job count, cache strategy, test splitting,
              Docker layer caching, artifact handling, runner selection
Common constraints: all tests must still pass, no skipping checks,
                    security scans must still run
```

### Algorithm Competition (LeetCode / CP)

```
Metric: execution_time_ms or score
Editable: solution.py
Common knobs: algorithm choice (BFS/DFS, DP, greedy), data structure selection,
              early termination, pruning, memoization, bit manipulation
Common constraints: must pass ALL test cases (correctness non-negotiable),
                    no hardcoded answers, language/platform constraints
```

When the user's task matches one of these domains, use the relevant template as
a starting point for the interview and scaffold.

---

## Summary Checklist

When user says "use autoresearch to...":

- [ ] **Phase 1**: 10 questions across 4 rounds → confirm understanding
- [ ] **Phase 2.1**: Generate `program.md` from filled template
- [ ] **Phase 2.2**: Create fixed infrastructure (prepare.py equivalent)
- [ ] **Phase 2.3**: Prepare editable sandbox (train.py equivalent)
- [ ] **Phase 2.4**: Create `.gitignore`
- [ ] **Phase 2.5**: Run baseline → verify works → record in results.tsv
- [ ] **Phase 2.6**: `git checkout -b autoresearch/<tag>`, commit scaffold
- [ ] **Phase 2.7**: Write initial `.autoresearch_checkpoint.json`
- [ ] **Phase 3**: Enter loop with smart selection + adversarial verification
- [ ] **On interrupt**: Final checkpoint save, status summary to user
