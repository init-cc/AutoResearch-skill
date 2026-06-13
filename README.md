<div align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)">
    <img alt="AutoResearch" src="https://img.shields.io/badge/AutoResearch-Autonomous%20Experiment%20Loop-1f6feb?style=for-the-badge&logo=robotframework&logoColor=white">
  </picture>
</div>

<div align="center">
  <a href="https://github.com/init-cc/AutoResearch-skill/stargazers"><img src="https://img.shields.io/github/stars/init-cc/AutoResearch-skill?style=flat-square&color=yellow" alt="Stars"></a>
  <a href="https://github.com/init-cc/AutoResearch-skill/blob/master/LICENSE"><img src="https://img.shields.io/github/license/init-cc/AutoResearch-skill?style=flat-square&color=blue" alt="License"></a>
  <a href="https://github.com/init-cc/AutoResearch-skill"><img src="https://img.shields.io/badge/version-1.0.0-blue?style=flat-square" alt="Version"></a>
  <a href="https://github.com/karpathy/autoresearch"><img src="https://img.shields.io/badge/inspired%20by-karpathy%2Fautoresearch-orange?style=flat-square" alt="Inspired by"></a>
</div>

<br>

<details open>
<summary><b>🇬🇧 English</b> &nbsp;|&nbsp; Click to switch language / 点击切换语言</summary>

<br>

<h1 align="center">🤖 AutoResearch Skill</h1>
<p align="center">
  <em>Tell the agent what to optimize. Answer a few questions. Go to sleep.<br>
  It experiments all night — you wake up to a log of improvements.</em>
</p>

---

## What It Does

A **Claude Code skill** that generalizes the
[karpathy/autoresearch](https://github.com/karpathy/autoresearch) pattern
into a **universal autonomous experiment loop** for any optimization task.

<table>
<tr><td width="80"><b>1. Interview</b></td><td>4 rounds, 10 questions — target, metric, constraints, budget, project path.</td></tr>
<tr><td><b>2. Scaffold</b></td><td>Generates <code>program.md</code> (research constitution), fixed evaluation harness, editable sandbox, <code>.gitignore</code>.</td></tr>
<tr><td><b>3. Loop</b></td><td>Autonomous mode: edit → commit → run → grep metric → keep/discard → record → repeat. <b>Never asks "should I continue?"</b></td></tr>
</table>

## ⚡ Key Features

| Feature | Description |
|---|---|
| 🔀 **Multi-Agent Parallel** | Fan out N agents across isolated git worktrees, each exploring a different hypothesis direction. Merge the best results. |
| ⚔️ **Adversarial Verification** | Before marking "keep", a skeptic's checklist validates the improvement is genuine — not noise, seed luck, or metric gaming. |
| 💾 **Checkpoint/Resume** | Loop survives session restarts. Every experiment is logged to `.autoresearch_checkpoint.json`. `/autoresearch:resume` picks up exactly where you left off. |
| 🧠 **Smart Selection** | Analyzes patterns in results.tsv to avoid dead ends, exploit promising directions, and inject controlled randomness when stuck. |
| 📐 **Domain Templates** | Pre-built interview scaffolds for web perf, DB tuning, ML hyperparams, bundle size, CI/CD, and algorithm optimization. |
| 🛡️ **Correctness Gate** | Multi-metric support — if your task requires correctness, the agent auto-discards any change that breaks it, no matter how fast. |

## The Three-File Architecture

| File | Role | Who Touches It |
|------|------|:---:|
| `program.md` | Research agenda — goals, rules, operational constraints | 🧑 **Human** |
| `prepare.py` (equiv) | Fixed evaluation harness — test data, metric, ground truth | 🔒 **Nobody** |
| `train.py` (equiv) | Editable sandbox — the code being optimized | 🤖 **Agent** |

The human <em>programs the research org</em> by iterating on `program.md`.
The agent <em>does the work</em> by iterating on the sandbox.
The fixed harness <em>keeps everyone honest</em>.

## The Experiment Loop

```
LOOP FOREVER:
  1. Read context (program.md + current code + recent results)
  2. Form hypothesis, make ONE logical change to the sandbox
  3. git commit with a descriptive message
  4. Run experiment > run.log 2>&1
  5. grep the metric from run.log
  6. If improved → keep commit, branch advances
  7. If worse   → git reset, discard
  8. If crashed → classify (typo? resource? fundamental?), fix or skip
  9. Append row to results.tsv (never commit this file)
  10. Repeat — never stop until interrupted
```

## Example Use Cases

- 🎯 *"autoresearch optimize my web crawler's throughput"*
- 🎯 *"autoresearch tune this PostgreSQL query's execution time"*
- 🎯 *"autoresearch find the best hyperparameters for my XGBoost model"*
- 🎯 *"autoresearch shrink my React app's bundle size"*
- 🎯 *"autoresearch maximize my trading strategy's Sharpe ratio"*
- 🎯 *"autoresearch minimize my CI pipeline's runtime"*

## Installation

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/init-cc/AutoResearch-skill.git \
  ~/.claude/skills/autoresearch

# Restart Claude Code — the skill auto-registers
```

Or via plugin marketplace (coming soon):

```bash
/plugin install init-cc/AutoResearch-skill
```

## Key Design Principles

1. **Single editable file** — diffs stay short and reviewable
2. **Fixed evaluation** — metric lives in locked code, no "cheating"
3. **Git as state machine** — commits = experiments, resets = discards
4. **Fixed budget per experiment** — every run directly comparable
5. **Never-stop autonomy** — runs until human interrupts, zero permission-asking
6. **Simplicity bias** — deleting code while keeping performance is a win
7. **grep-friendly output** — one metric per line, trivial to parse

## Edge Case Handling

| Scenario | Agent Behavior |
|---|---|
| Experiment crashes | Read last 50 log lines; quick-fix typos, skip fundamental flaws |
| 3 consecutive crashes | Stop and report — something is systemically broken |
| 10+ straight discards | Detect stagnation; try larger, more radical changes |
| Resource exhaustion | Scale back, log usage alongside metric |
| Correctness constraint violated | Auto-discard regardless of primary metric improvement |
| Timeout exceeded | Kill process, mark crash, revert |

## File Structure

```
autoresearch/
├── .claude-plugin/
│   └── plugin.json          # Skill metadata
├── skills/
│   └── autoresearch/
│       └── SKILL.md         # Full skill logic (~700 lines)
└── README.md                # You are here
```

## License

MIT — same as the original [karpathy/autoresearch](https://github.com/karpathy/autoresearch).

## Credits

Pattern by [Andrej Karpathy](https://github.com/karpathy).
Skill design & generalization by [init-cc](https://github.com/init-cc).

</details>

<details>
<summary><b>🇨🇳 中文</b> &nbsp;|&nbsp; 点击切换语言 / Click to switch language</summary>

<br>

<h1 align="center">🤖 AutoResearch 技能</h1>
<p align="center">
  <em>告诉 Agent 你要优化什么，回答几个问题，然后去睡觉。<br>
  它自己实验一整晚——你早上起来看优化记录。</em>
</p>

---

## 这个技能做什么

一个 **Claude Code 技能**，将
[karpathy/autoresearch](https://github.com/karpathy/autoresearch)
的自主实验模式泛化成**通用自主实验循环**，适用于任何优化任务。

<table>
<tr><td width="80"><b>1. 采访</b></td><td>4 轮 10 个结构化问题：优化目标、评价指标、约束条件、时间预算、项目路径。</td></tr>
<tr><td><b>2. 搭建</b></td><td>生成 <code>program.md</code>（研究章程）、固定评估代码、可编辑沙盒、<code>.gitignore</code>。</td></tr>
<tr><td><b>3. 循环</b></td><td>自主模式：改代码 → 提交 → 跑实验 → grep 指标 → 保留/丢弃 → 记录 → 重复。<b>绝不问"要不要继续"。</b></td></tr>
</table>

## ⚡ 核心特性

| 特性 | 说明 |
|---|---|
| 🔀 **多智能体并行** | 同时启动 N 个 Agent，在隔离的 git worktree 中探索不同方向，最后合并最优结果。 |
| ⚔️ **对抗性验证** | 每个"keep"决策前，用怀疑论者检查清单验证改进是真实的——而非噪声、随机种子运气或指标作弊。 |
| 💾 **断点续跑** | 实验状态持久化到 `.autoresearch_checkpoint.json`，会话重启后 `/autoresearch:resume` 无缝接续。 |
| 🧠 **智能选题** | 分析 results.tsv 中的模式，避免死胡同，放大有希望的方向，卡住时注入受控随机性。 |
| 📐 **领域模板** | 预置的采访脚手架：Web 性能、数据库调优、ML 超参、打包体积、CI/CD、算法优化。 |
| 🛡️ **正确性门禁** | 多指标支持——如果有正确性约束，任何破坏正确性的改动无论跑多快都自动丢弃。 |

## 三文件架构

| 文件 | 角色 | 谁来改 |
|------|------|:---:|
| `program.md` | 研究章程 — 目标、规则、操作约束 | 🧑 **人类** |
| `prepare.py`（等价物） | 固定评估基础设施 — 测试数据、指标、真相标准 | 🔒 **谁都不能动** |
| `train.py`（等价物） | 可编辑沙盒 — 被优化的代码 | 🤖 **Agent** |

人类通过迭代 `program.md` 来<em>管理研究组织</em>。
Agent 通过迭代沙盒来<em>做实验</em>。
固定评估代码<em>保证公平公正</em>。

## 实验循环

```
LOOP FOREVER（无限循环）:
  1. 重新阅读上下文（program.md + 当前代码 + 近期实验结果）
  2. 形成假设，对沙盒做<strong>一个</strong>逻辑变更
  3. git commit，写清楚实验动机
  4. 运行实验 > run.log 2>&1
  5. 从 run.log 中 grep 提取指标
  6. 如果改善 → 保留 commit，分支前进
  7. 如果变差 → git reset，丢弃
  8. 如果崩溃 → 分类（typo？资源？思路错误？），快速修复或跳过
  9. 追加一行到 results.tsv（绝不提交此文件）
  10. 重复 — 永不自动停止
```

## 适用场景举例

- 🎯 *"autoresearch 优化我的爬虫吞吐量"*
- 🎯 *"autoresearch 调优这个 PostgreSQL 查询"*
- 🎯 *"autoresearch 找出 XGBoost 最佳超参数"*
- 🎯 *"autoresearch 缩小 React 打包体积"*
- 🎯 *"autoresearch 最大化交易策略夏普比率"*
- 🎯 *"autoresearch 缩短 CI 流水线运行时间"*

## 安装

```bash
# 克隆到 Claude Code 的 skills 目录
git clone https://github.com/init-cc/AutoResearch-skill.git \
  ~/.claude/skills/autoresearch

# 重启 Claude Code 即可自动注册
```

或通过插件市场安装（即将上线）：

```bash
/plugin install init-cc/AutoResearch-skill
```

## 核心设计原则

1. **单一可编辑文件** — diff 简短可审查，scope 可控
2. **固定评估标准** — 指标定义在锁定文件中，杜绝"作弊"
3. **Git 作为状态机** — commit = 实验记录，reset = 丢弃坏实验
4. **固定实验预算** — 每次实验成本恒定，结果直接可比
5. **永不停止的自主性** — 跑到人类手动中断，绝不请求许可
6. **简单性偏好** — 在保持性能的前提下删代码，算一次成功的实验
7. **grep 友好的输出** — 每行一个指标，解析零成本

## 边界情况处理

| 场景 | Agent 行为 |
|---|---|
| 实验崩溃 | 读取最后 50 行日志；typo 类快速修复，思路性错误直接跳过 |
| 连续 3 次崩溃 | 停止并报告——系统性问题 |
| 连续 10+ 次 discard | 检测停滞；尝试更大、更激进的改动 |
| 资源耗尽 | 自动缩减规模，记录资源使用 |
| 正确性约束被打破 | 无论主指标多好，一律 discard |
| 超时 | 杀掉进程，标记 crash，回退 |

## 文件结构

```
autoresearch/
├── .claude-plugin/
│   └── plugin.json          # Skill 元数据
├── skills/
│   └── autoresearch/
│       └── SKILL.md         # 完整 skill 逻辑（~700 行）
└── README.md                # 你在这里
```

## 许可证

MIT — 与原始 [karpathy/autoresearch](https://github.com/karpathy/autoresearch) 一致。

## 致谢

模式源自 [Andrej Karpathy](https://github.com/karpathy)。
Skill 设计与泛化由 [init-cc](https://github.com/init-cc) 完成。

</details>
