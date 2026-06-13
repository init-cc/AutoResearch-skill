---
name: autoresearch
description: |
  Autonomous experiment loop — the agent interviews you to understand the optimization
  target, scaffolds a self-running research project (program.md + fixed harness +
  editable sandbox), then iterates autonomously: modify → commit → run → evaluate →
  keep/discard → repeat. Based on the karpathy/autoresearch pattern.
  Trigger when user says "use autoresearch to...", "autoresearch...",
  "自动研究...", "自主实验...", or asks to set up an autonomous optimization loop.
version: 1.0.0
author: init-cc
tags: [autoresearch, autonomous, experiment, optimization, agent-loop, automation]
---

# AutoResearch — Autonomous Experiment Loop

> *"You program the research org in Markdown. The agent runs experiments while you sleep."*

## What This Skill Does

This skill implements the **autoresearch pattern** originally designed by Andrej Karpathy
([karpathy/autoresearch](https://github.com/karpathy/autoresearch)) for autonomous LLM
training research, generalized to work for **any optimization task**.

The core insight: three roles, three files.

| Role | File | Who Touches It |
|------|------|----------------|
| 🧠 **Research Agenda** | `program.md` | **Human** — writes and iterates the research plan |
| 🔒 **Fixed Infrastructure** | `prepare.py` (or equiv) | **Nobody** — locked after initial setup |
| 🛠️ **Editable Sandbox** | `train.py` (or equiv) | **Agent** — freely modifies within constraints |

The agent runs a git-backed experiment loop:

```
LOOP FOREVER:
  1. Read context (program.md + current code)
  2. Form hypothesis, edit the sandbox file
  3. git commit the change
  4. Run experiment, capture output to run.log
  5. Parse metric from run.log
  6. If improved → keep commit, advance branch
  7. If worse → git reset (discard)
  8. If crashed → attempt fix, or mark crash and move on
  9. Record to results.tsv
  10. Repeat until human interrupts
```

---

## Phase 0: Trigger Detection

This skill triggers when the user says any of:

- "use autoresearch to ..."
- "autoresearch ..."
- "自主研究 ..." / "自动研究 ..."
- "set up an autonomous experiment loop for ..."
- "帮我自动优化 ..." + context implying autonomous iteration

When triggered, **do NOT immediately start the loop**. Go to Phase 1 (Discovery Interview).

---

## Phase 1: Discovery Interview

Before creating any files, you MUST interview the user to understand the task.
Ask these questions. You can batch 2–3 at a time, but cover ALL of them
before moving to Phase 2.

### Round 1: Core Task Definition (ask first)

```
To set up your AutoResearch project, I need to understand what we're optimizing.
Please answer these:

1. **What are we optimizing?** Describe the task in one sentence.
   (e.g., "A Python web crawler that scrapes 100 target URLs as fast as possible",
    "A PostgreSQL query that joins 5 tables and returns aggregated results",
    "A PyTorch image classifier on CIFAR-10")

2. **Which file(s) will the agent edit?** Pick ONE primary file that contains
   the code/config the agent should modify.
   (e.g., `crawler.py`, `query.sql`, `model.py`)
   - This file should contain tunable knobs (hyperparameters, algorithms, configs)
   - The agent will freely edit anything in this file

3. **How do we run ONE experiment?** What CLI command executes a single trial?
   (e.g., `python crawler.py`, `psql -f query.sql`, `python train.py`)
```

### Round 2: Metric & Evaluation (ask second)

```
4. **What is the SINGLE metric we optimize for?**
   Give me:
   - The exact grep pattern to extract it from output
     (e.g., `grep "^total_time_seconds:" run.log`)
   - Whether **lower is better** or **higher is better**
   - A rough baseline value if you know it
   - Units (seconds, %, points, etc.)

5. **Is there a correctness/quality constraint?**
   Some optimizations can't sacrifice correctness. For example:
   - "Crawler must still return 100% correct results"
   - "Query must return identical rows to the reference"
   - "Model accuracy must not drop below 95%"
   If yes, how is correctness measured?
```

### Round 3: Constraints & Budget (ask third)

```
6. **What CANNOT the agent change?**
   Which files, functions, or configurations are off-limits?
   - Fixed test data / evaluation harness
   - External dependencies (can't `pip install` new packages)
   - API rate limits
   - Output format requirements

7. **Time/resource budget per experiment:**
   - Fixed wall-clock time? (e.g., 5 minutes)
   - Fixed data size? (e.g., 100 test URLs, 10K rows)
   - Or both?
   If time-based, what's the timeout before we kill a stuck experiment?

8. **Any resource constraints?**
   - Memory limit
   - GPU VRAM limit
   - API call budget
   - Network bandwidth
```

### Round 4: Project Setup (ask fourth)

```
9. **Where should the project live?**
   - Create a new directory? (recommended: `./autoresearch-<task>/`)
   - Use an existing project directory?
   If existing, I'll add the autoresearch scaffold files without disrupting your code.

10. **What should we call this research run?**
    A short tag like `crawler-opt`, `sql-tuning`, `jun14`.
    This becomes the git branch name: `autoresearch/<tag>`
```

### After All Answers Are Collected

Before moving to Phase 2, **summarize your understanding** back to the user and ask
for confirmation. Show a table like:

```
📋 AutoResearch Setup Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Task:          Optimize web crawler speed
Editable:      crawler.py
Run command:   python crawler.py
Metric:        total_time_seconds (lower better)
Constraint:    Must maintain 100% accuracy
Budget:        100 URLs per run (~30 sec)
Project dir:   ./autoresearch-crawler/
Run tag:       crawler-jun14
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Does this look correct? I'll then scaffold the project and start experimenting.
```

Do NOT proceed to Phase 2 until the user confirms.

### Handling Vague Answers

If the user can't specify a metric or run command:
- Help them design one: "Let's add a timer and print statement to your script"
- Offer to instrument their code with measurement before starting
- If no clear single metric exists, propose a composite or the most important one

If the user can't identify the editable file:
- Ask which file has the most "knobs" (parameters, algorithms, configs)
- If everything is one file, that file IS the editable sandbox — the fixed
  infrastructure can be a separate test runner you create

---

## Phase 2: Scaffold Generation

Once the user confirms, generate these files in the project directory.

### File 1: `program.md` — The Research Agenda

This is the MOST IMPORTANT file. It's the "constitution" the agent follows.
Generate it using the template below, filling in `{PLACEHOLDERS}` from the
discovery interview.

```markdown
# AutoResearch: {TASK_NAME}

{ONE_SENTENCE_TASK_DESCRIPTION}

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `{tag}`).
   The branch `autoresearch/{tag}` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/{tag}` from current master/main.
3. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `{EDITABLE_FILE}` — the file you modify. {WHAT_THIS_FILE_CONTAINS}.
   - `{FIXED_FILE}` — {WHAT_THE_FIXED_FILE_IS}. Do not modify.
   - Any other context files: `{OTHER_CONTEXT_FILES}`
4. **Verify infrastructure**: {HOW_TO_VERIFY_THINGS_ARE_READY}.
5. **Initialize results.tsv**: Create `results.tsv` with just the header row.
   The baseline will be recorded after the first run.
6. **Confirm and go**: Confirm setup looks good, then kick off experimentation.

## Experimentation

Each experiment runs {BUDGET_DESCRIPTION}. You launch it as: `{RUN_COMMAND} > run.log 2>&1`

**What you CAN do:**
- Modify `{EDITABLE_FILE}` — this is the only file you edit. {WHAT_CAN_BE_CHANGED}.

**What you CANNOT do:**
- Modify `{FIXED_FILE}`. It is read-only. {WHY_IT_CANT_BE_CHANGED}.
- Install new packages or add dependencies. You can only use what's already available.
- Modify the evaluation harness. The {EVAL_FUNCTION} in `{FIXED_FILE}` is the ground truth.
- {ANY_OTHER_CONSTRAINTS}

**The goal is simple: {GOAL_STATEMENT}.**

**{RESOURCE_CONSTRAINT_NOTE}**

**Simplicity criterion**: All else being equal, simpler is better. A small improvement
that adds ugly complexity is not worth it. Conversely, removing something and getting
equal or better results is a great outcome — that's a simplification win. When evaluating
whether to keep a change, weigh the complexity cost against the improvement magnitude.

**The first run**: Your very first run should always be to establish the baseline,
so you will run the {EDITABLE_FILE} as is, without any modifications.

## Output format

Once the experiment finishes it prints a summary. Extract the key metric:

```
{METRIC_GREP_COMMAND}
```

{ADDITIONAL_METRICS_TO_GREP_IF_ANY}

{METRIC_DIRECTION_NOTE} (Lower is better / Higher is better)

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated).

The TSV has a header row and 5 columns:

```
commit	{METRIC_COLUMN}	{ANY_OTHER_COLUMNS}	status	description
```

1. git commit hash (short, 7 chars)
2. {METRIC_COLUMN_DESCRIPTION} — use {CRASH_PLACEHOLDER} for crashes
3. {OTHER_COLUMN_DESCRIPTIONS}
4. status: `keep`, `discard`, or `crash`
5. short text description of what this experiment tried

Example:

```
commit	{METRIC_COLUMN}	status	description
a1b2c3d	{BASELINE_VALUE}	keep	baseline (no changes)
b2c3d4e	{IMPROVED_VALUE}	keep	{EXAMPLE_IMPROVEMENT}
c3d4e5f	{WORSENED_VALUE}	discard	{EXAMPLE_DISCARD}
d4e5f6g	{CRASH_VALUE}	crash	{EXAMPLE_CRASH}
```

## The experiment loop

The experiment runs on a dedicated branch (e.g. `autoresearch/{tag}`).

LOOP FOREVER:

1. Look at the git state: the current branch/commit we're on
2. Tune `{EDITABLE_FILE}` with an experimental idea by directly editing the code.
3. git commit the change with a descriptive one-line message
4. Run the experiment: `{RUN_COMMAND} > run.log 2>&1` (redirect everything — do NOT
   use tee or let output flood your context)
5. Read out the results: `{METRIC_GREP_COMMAND}`
6. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read
   the error and attempt a fix. If you can't get things to work after more than a
   few attempts, give up on this idea.
7. Record the results in results.tsv (NOTE: do not commit results.tsv — leave it
   untracked by git)
8. If {METRIC_NAME} {IMPROVED_DIRECTION}, you "advance" the branch, keeping the
   git commit
9. If {METRIC_NAME} is {EQUAL_OR_WORSE_DIRECTION}, you git reset back to where
   you started (keeping the working tree changes discarded)
10. {ADDITIONAL_CHECKS} (e.g., "If accuracy dropped below threshold, discard regardless of metric")

The idea is that you are a completely autonomous researcher trying things out.
If they work, keep. If they don't, discard. And you're advancing the branch so
that you can iterate.

**Timeout**: Each experiment should take ~{EXPECTED_DURATION}. If a run exceeds
{TIMEOUT_DURATION}, kill it and treat it as a failure (discard and revert).

**Crashes**: If a run crashes (resource exhaustion, bug, etc.), use your judgment:
If it's something trivial (a typo, a missing import), fix it and re-run. If the
idea itself is fundamentally broken, log "crash" as the status in results.tsv
and move on.

**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do
NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?"
or "is this a good stopping point?". The human might be asleep, or gone from the
computer and expects you to continue working *indefinitely* until you are manually
stopped. You are autonomous.

**If stuck**: If you run out of ideas, don't give up. Re-read the in-scope files
for new angles. Try more aggressive changes. Try simplifying. Try combining previous
near-misses. Try reversing assumptions. The loop runs until the human interrupts
you, period.
```

### File 2: Fixed Infrastructure (`prepare.py` equivalent)

This file contains everything the agent **cannot change**:

```python
"""
Fixed infrastructure for {TASK_NAME} AutoResearch.
DO NOT MODIFY — this is the ground truth evaluation harness.
"""

# ============================================================================
# Fixed Constants (do not modify)
# ============================================================================

{HARDCODED_CONSTANTS}

# ============================================================================
# Fixed Test Data / Evaluation (do not modify)
# ============================================================================

{EVALUATION_FUNCTION}

# ============================================================================
# Runner (do not modify)
# ============================================================================

if __name__ == "__main__":
    # This is the fixed entry point that calls into the editable code
    {RUNNER_LOGIC}
```

Key design rules for this file:
1. It imports from the editable file (if needed)
2. It defines the evaluation metric precisely
3. It prints results in a grep-friendly format
4. It does NOT contain any tunable parameters
5. Comment `# DO NOT MODIFY` on every section

### File 3: Editable Sandbox (`train.py` equivalent)

This is the file the agent will edit. It should:

1. Have clear hyperparameter/config constants at the top (easy to tune)
2. Contain the main logic the agent can restructure
3. NOT contain the fixed evaluation — that's in prepare.py
4. Print metrics at the end in a grep-friendly format

```python
"""
{EDITABLE_FILE_DESCRIPTION}
AutoResearch editable file — the agent modifies this freely.
"""

# ============================================================================
# Tunable Parameters (edit these freely)
# ============================================================================

{TUNABLE_PARAMS_WITH_DEFAULTS}

# ============================================================================
# Main Logic (edit freely)
# ============================================================================

{MAIN_LOGIC}

# ============================================================================
# Run (typically calls fixed evaluation from prepare.py)
# ============================================================================

if __name__ == "__main__":
    {RUN_LOGIC_THAT_PRINTS_METRICS}
```

### File 4: `.gitignore`

```
run.log
results.tsv
__pycache__/
*.pyc
.DS_Store
```

### Generating Tips

- These files should be **functional from the start** — the baseline run must work
- If the project already has code, adapt the existing files instead of overwriting
- Keep files small (< 500 lines each) so the agent can read them quickly
- The metric output format must be consistent: `metric_name: <value>` one per line

---

## Phase 3: Autonomous Experiment Loop

After scaffolding is confirmed and the baseline runs successfully, enter the loop.
This phase follows `program.md`'s LOOP FOREVER exactly.

### Important Loop Behaviors

#### Before Each Experiment

1. **Read context files** — at minimum, re-read `program.md` and the editable file
   to have fresh context
2. **Review recent results** — check `results.tsv` (last 10-20 lines) to avoid
   repeating failed ideas
3. **Form a clear hypothesis** — "I think changing X from A to B will improve
   the metric because Y"

#### Making Changes

- Make ONE logical change per experiment (not 5 unrelated things)
- Write a descriptive git commit message explaining the hypothesis
- If the change is complex, add a brief comment in the code explaining the reasoning

#### Running the Experiment

```bash
{RUN_COMMAND} > run.log 2>&1
```

- Always redirect both stdout and stderr
- Do NOT use `tee` — it fills your context window
- Set a timeout alarm if the command might hang

#### Evaluating Results

```bash
grep "^{METRIC_PATTERN}" run.log
```

- If empty output → the run crashed, check `tail -n 50 run.log`
- Parse the numeric value carefully
- Compare to the current best (from `results.tsv`)

#### Decision Matrix

| Metric | Correctness | Action |
|--------|-------------|--------|
| Improved | Passes | ✅ **KEEP** — advance branch |
| Improved | Fails | ❌ **DISCARD** — correctness is non-negotiable |
| Same | Passes | ❌ **DISCARD** — no improvement, revert |
| Same | Fails | ❌ **DISCARD** |
| Worse | Passes | ❌ **DISCARD** — revert to last good commit |
| Worse | Fails | ❌ **DISCARD** |
| Crash | N/A | 🔧 **FIX or SKIP** — try quick fix (typo, import), else mark crash |

#### Advancing the Branch (KEEP)

```bash
# Commit is already made — just continue
# The branch now points to the improved commit
```

#### Discarding (DISCARD)

```bash
git reset --hard HEAD~1  # Revert the bad commit entirely
```

#### After Each Experiment

Always append to results.tsv:
```
<commit_hash>\t<metric_value>\t<status>\t<description>
```

Do NOT commit results.tsv — it's the raw lab notebook.

#### Experiment Naming

Good descriptions:
- `increase learning rate from 0.01 to 0.05`
- `replace BFS with A* search`
- `add connection pooling (pool_size=20)`
- `switch from requests to aiohttp`

Bad descriptions:
- `try something`
- `fix`
- `experiment 42`

---

## Edge Cases & Safety

### Crash Recovery Protocol

When `grep` returns empty (run.log has no metric line):

1. `tail -n 50 run.log` — read the error
2. Classify the error:
   - **Typo / syntax error** → fix immediately, re-run same experiment
   - **Resource exhaustion** → reduce resource usage, try again
   - **Fundamental design flaw** → mark crash, discard, move on
   - **Timeout** → kill process, mark crash, discard
3. After 3 consecutive crashes → **stop and report to user**
   (something is fundamentally broken)

### Metric Stagnation Detection

If 10+ consecutive experiments are all "discard":
- The agent is stuck in a local optimum
- Strategy: try larger, more radical changes
- Consider reverting to an earlier commit and trying a different direction
- Re-read `program.md` — maybe the search space needs redefining

### Resource Exhaustion

If memory/disk/API calls are running out:
- Scale back the experiment size
- Add a resource check before each run
- Log resource usage alongside metrics

### When to Actually Stop

The agent should ONLY stop when:
1. The user explicitly interrupts
2. 3+ consecutive unrecoverable crashes (different causes)
3. The git branch has diverged in an unfixable way
4. Disk space is critically low

The agent should NEVER stop because:
- "I ran out of ideas" → think harder
- "It's been running for N hours" → that's the point
- "I'm not sure if I should continue" → yes you should

---

## Advanced Patterns

### Multi-Metric Experiments

If the user has both a primary metric AND a correctness constraint:

```
1. Run experiment
2. Check correctness metric first
3. If correctness failed → DISCARD regardless of primary metric
4. If correctness passed → evaluate primary metric normally
```

The correctness check goes in the fixed infrastructure file.

### Warmup/Cooldown

Some experiments need warmup (JIT compilation, cache warming). Account for this:
- Specify in program.md: "Ignore first N steps/metrics"
- The evaluation function in prepare.py handles this

### Parallel Experiment Branches

Advanced: run multiple git branches in parallel from the same base:
- `autoresearch/<tag>-gpu0` — explores architecture changes
- `autoresearch/<tag>-gpu1` — explores optimizer changes
- Merge the best results from each

This requires multiple agent instances.

### Human-in-the-Loop Review

After the agent stops, the human reviews:
1. `git log autoresearch/<tag>` — clean history of only successful changes
2. `results.tsv` — complete lab notebook of every attempt
3. `run.log` — output of the last (best) experiment
4. The editable file itself — final state after all improvements

The human then:
- Updates `program.md` with lessons learned
- Starts a new branch for the next night's research
- The process compounds over time

---

## Summary Checklist

When user says "use autoresearch to...":

- [ ] Phase 1: Interview user (all 10 questions across 4 rounds)
- [ ] Phase 1: Summarize and get confirmation
- [ ] Phase 2: Generate `program.md`
- [ ] Phase 2: Generate fixed infrastructure (prepare equivalent)
- [ ] Phase 2: Prepare/verify editable sandbox (train equivalent)
- [ ] Phase 2: Create `.gitignore`
- [ ] Phase 2: Run baseline experiment to verify everything works
- [ ] Phase 2: Create `results.tsv` with header and baseline row
- [ ] Phase 2: Create git branch `autoresearch/<tag>`
- [ ] Phase 3: Enter autonomous loop (NEVER STOP until interrupted)
