<style>
  /* ── Language toggle ── */
  #lang-en:checked ~ .lang-bar label[for="lang-en"],
  #lang-zh:checked ~ .lang-bar label[for="lang-zh"] {
    background: #1f6feb;
    color: #fff;
    border-color: #1f6feb;
  }
  .en, .zh { display: none; }
  #lang-en:checked ~ .en { display: block; }
  #lang-zh:checked ~ .zh { display: block; }
  /* ── Shared styles ── */
  .lang-bar {
    display: flex;
    justify-content: center;
    gap: 0;
    margin: 24px 0 32px;
  }
  .lang-bar label {
    padding: 6px 22px;
    border: 1px solid #30363d;
    cursor: pointer;
    font-weight: 600;
    font-size: 14px;
    user-select: none;
    background: #0d1117;
    color: #c9d1d9;
  }
  .lang-bar label:first-of-type { border-radius: 6px 0 0 6px; }
  .lang-bar label:last-of-type  { border-radius: 0 6px 6px 0; }
  .lang-bar label:hover { background: #161b22; }
  hr { margin: 32px 0; }
</style>

<input type="radio" name="lang" id="lang-en" checked hidden>
<input type="radio" name="lang" id="lang-zh" hidden>

<div class="lang-bar">
  <label for="lang-en">🇬🇧 English</label>
  <label for="lang-zh">🇨🇳 中文</label>
</div>

<!-- ═══════════════════════════════════════════ -->
<!--              ENGLISH                       -->
<!-- ═══════════════════════════════════════════ -->

<div class="en">

<h1 align="center">🤖 AutoResearch Skill</h1>
<p align="center">
  <em>Tell the agent what to optimize, answer a few questions, then go to sleep.<br>
  It experiments all night. You wake up to results.</em>
</p>

<hr>

<h2>What It Does</h2>

<p>This is a <strong>Claude Code skill</strong> that generalizes the
<a href="https://github.com/karpathy/autoresearch">karpathy/autoresearch</a> pattern —
originally for LLM training — into a <strong>universal autonomous experiment loop</strong>
for any optimization task.</p>

<p>When you invoke <code>/autoresearch</code> or say
<em>"use autoresearch to help me optimize X"</em>, the agent will:</p>

<table>
<tr><td><strong>1. Interview</strong></td><td>Ask you 10 structured questions across 4 rounds: target, metric, constraints, budget, and project path.</td></tr>
<tr><td><strong>2. Scaffold</strong></td><td>Generate four files: <code>program.md</code> (research constitution), a fixed evaluation harness, an editable sandbox, and <code>.gitignore</code>.</td></tr>
<tr><td><strong>3. Loop</strong></td><td>Enter autonomous mode: edit → commit → run → grep metric → keep/discard → record → repeat. <strong>It never asks "should I continue?"</strong></td></tr>
</table>

<h2>The Three-File Architecture</h2>

<p>This is the core design pattern — three roles, three files:</p>

<table>
<tr><th>File</th><th>Role</th><th>Who Touches It</th></tr>
<tr><td><code>program.md</code></td><td>Research agenda — goals, rules, operational constraints</td><td align="center">🧑 <strong>Human</strong></td></tr>
<tr><td><code>prepare.py</code> (equiv)</td><td>Fixed evaluation harness — test data, metric, ground truth</td><td align="center">🔒 <strong>Nobody</strong></td></tr>
<tr><td><code>train.py</code> (equiv)</td><td>Editable sandbox — the code being optimized</td><td align="center">🤖 <strong>Agent</strong></td></tr>
</table>

<p>The human <em>programs the research org</em> by iterating on <code>program.md</code>.
The agent <em>does the work</em> by iterating on the sandbox file.
The fixed harness <em>keeps everyone honest</em>.</p>

<h2>The Experiment Loop</h2>

<pre><code>LOOP FOREVER:
  1. Read context (program.md + current code + recent results)
  2. Form hypothesis, make ONE logical change to the sandbox
  3. git commit with a descriptive message
  4. Run experiment &gt; run.log 2&gt;&amp;1
  5. grep the metric from run.log
  6. If improved → keep commit, branch advances
  7. If worse   → git reset, discard
  8. If crashed → classify (typo? resource? fundamental?), fix or skip
  9. Append row to results.tsv (never commit this file)
  10. Repeat — never stop until interrupted</code></pre>

<h2>Example Use Cases</h2>

<ul>
<li>🎯 <em>"Use autoresearch to optimize my web crawler's throughput"</em></li>
<li>🎯 <em>"Use autoresearch to tune this PostgreSQL query's execution time"</em></li>
<li>🎯 <em>"Use autoresearch to find the best hyperparameters for my XGBoost model"</em></li>
<li>🎯 <em>"Use autoresearch to shrink my React app's bundle size"</em></li>
<li>🎯 <em>"Use autoresearch to maximize my trading strategy's Sharpe ratio"</em></li>
<li>🎯 <em>"Use autoresearch to minimize my CI pipeline's runtime"</em></li>
</ul>

<h2>Installation</h2>

<pre><code># Clone into your Claude Code skills directory
git clone https://github.com/init-cc/AutoResearch-skill.git \
  ~/.claude/skills/autoresearch

# Restart Claude Code — the skill auto-registers
</code></pre>

<p>Manual install:</p>

<pre><code>mkdir -p ~/.claude/skills/autoresearch
cp -r .claude-plugin skills ~/.claude/skills/autoresearch/
</code></pre>

<h2>Key Design Principles</h2>

<ol>
<li><strong>Single editable file</strong> — diffs stay short and reviewable</li>
<li><strong>Fixed evaluation</strong> — metric lives in locked code, no "cheating"</li>
<li><strong>Git as state machine</strong> — commits = experiments, resets = discards</li>
<li><strong>Fixed budget per experiment</strong> — makes every run directly comparable</li>
<li><strong>Never-stop autonomy</strong> — runs until human interrupts, zero permission-asking</li>
<li><strong>Simplicity bias</strong> — deleting code while keeping performance is counted as a win</li>
<li><strong>grep-friendly output</strong> — one metric per line, trivial to parse</li>
</ol>

<h2>File Structure</h2>

<pre><code>autoresearch/
├── .claude-plugin/
│   └── plugin.json          # Skill metadata
├── skills/
│   └── autoresearch/
│       └── SKILL.md         # Full skill logic (~700 lines)
└── README.md                # You are here
</code></pre>

<h2>Edge Case Handling</h2>

<table>
<tr><th>Scenario</th><th>Agent Behavior</th></tr>
<tr><td>Experiment crashes</td><td>Read last 50 log lines; quick-fix typos, skip fundamental flaws</td></tr>
<tr><td>3 consecutive crashes</td><td>Stop and report — something is systemically broken</td></tr>
<tr><td>10+ straight discards</td><td>Detect stagnation; try larger, more radical changes</td></tr>
<tr><td>Resource exhaustion</td><td>Scale back, log usage alongside metric</td></tr>
<tr><td>Correctness constraint violated</td><td>Auto-discard regardless of primary metric improvement</td></tr>
<tr><td>Timeout exceeded</td><td>Kill process, mark crash, revert</td></tr>
</table>

<h2>License</h2>

<p>MIT — same as the original <a href="https://github.com/karpathy/autoresearch">karpathy/autoresearch</a>.</p>

<h2>Credits</h2>

<p>Pattern by <a href="https://github.com/karpathy">Andrej Karpathy</a>.
Skill design &amp; generalization by <a href="https://github.com/init-cc">init-cc</a>.</p>

</div>

<!-- ═══════════════════════════════════════════ -->
<!--              中文                          -->
<!-- ═══════════════════════════════════════════ -->

<div class="zh">

<h1 align="center">🤖 AutoResearch 技能</h1>
<p align="center">
  <em>告诉 Agent 你要优化什么，回答几个问题，然后去睡觉。<br>
  它自己实验一整晚，你早上起来看结果。</em>
</p>

<hr>

<h2>这个技能做什么</h2>

<p>这是一个 <strong>Claude Code 技能</strong>，将
<a href="https://github.com/karpathy/autoresearch">karpathy/autoresearch</a>
的自主实验模式——原本用于 LLM 训练——泛化成一个<strong>通用的自主实验循环</strong>，
适用于任何优化任务。</p>

<p>当你输入 <code>/autoresearch</code> 或者说
<em>"使用 autoresearch 帮我优化 X"</em>，Agent 会：</p>

<table>
<tr><td><strong>1. 采访你</strong></td><td>4 轮共 10 个结构化问题：优化目标、评价指标、约束条件、时间预算、项目路径。</td></tr>
<tr><td><strong>2. 搭建脚手架</strong></td><td>生成 4 个文件：<code>program.md</code>（研究章程）、固定评估代码、可编辑沙盒、<code>.gitignore</code>。</td></tr>
<tr><td><strong>3. 自主循环</strong></td><td>进入自主模式：改代码 → 提交 → 跑实验 → grep 指标 → 好就留/差就撤 → 记录 → 重复。<strong>绝不问"要不要继续"。</strong></td></tr>
</table>


<h2>三文件架构</h2>

<p>这是核心设计模式——三种角色，三个文件：</p>

<table>
<tr><th>文件</th><th>角色</th><th>谁来改</th></tr>
<tr><td><code>program.md</code></td><td>研究章程 — 目标、规则、操作约束</td><td align="center">🧑 <strong>人类</strong></td></tr>
<tr><td><code>prepare.py</code>（等价物）</td><td>固定评估基础设施 — 测试数据、指标、真相标准</td><td align="center">🔒 <strong>谁都不能动</strong></td></tr>
<tr><td><code>train.py</code>（等价物）</td><td>可编辑沙盒 — 被优化的代码</td><td align="center">🤖 <strong>Agent</strong></td></tr>
</table>

<p>人类通过迭代 <code>program.md</code> 来<em>管理研究组织</em>。
Agent 通过迭代沙盒文件来<em>做实验</em>。
固定评估代码<em>保证公平公正</em>。</p>

<h2>实验循环</h2>

<pre><code>LOOP FOREVER（无限循环）:
  1. 重新阅读上下文（program.md + 当前代码 + 近期实验结果）
  2. 形成假设，对沙盒文件做<strong>一个</strong>逻辑变更
  3. git commit，写清楚这个实验的动机
  4. 运行实验 &gt; run.log 2&gt;&amp;1
  5. 从 run.log 中 grep 提取指标
  6. 如果指标改善 → 保留 commit，分支前进
  7. 如果指标变差 → git reset，丢弃
  8. 如果崩溃   → 分类（typo？资源耗尽？思路错误？），快速修复或跳过
  9. 追加一行到 results.tsv（绝不提交此文件）
  10. 重复 — 永不自动停止</code></pre>

<h2>适用场景举例</h2>

<ul>
<li>🎯 <em>"autoresearch 帮我优化爬虫的吞吐量"</em></li>
<li>🎯 <em>"autoresearch 调优这个 PostgreSQL 查询的执行时间"</em></li>
<li>🎯 <em>"autoresearch 找出 XGBoost 模型的最佳超参数"</em></li>
<li>🎯 <em>"autoresearch 缩小 React 应用的打包体积"</em></li>
<li>🎯 <em>"autoresearch 最大化交易策略的夏普比率"</em></li>
<li>🎯 <em>"autoresearch 缩短 CI 流水线的运行时间"</em></li>
</ul>

<h2>安装</h2>

<pre><code># 克隆到 Claude Code 的 skills 目录
git clone https://github.com/init-cc/AutoResearch-skill.git \
  ~/.claude/skills/autoresearch

# 重启 Claude Code 即可自动注册
</code></pre>

<p>也可以手动安装：</p>

<pre><code>mkdir -p ~/.claude/skills/autoresearch
cp -r .claude-plugin skills ~/.claude/skills/autoresearch/
</code></pre>

<h2>核心设计原则</h2>

<ol>
<li><strong>单一可编辑文件</strong> — diff 简短可审查，scope 可控</li>
<li><strong>固定评估标准</strong> — 指标定义在锁定文件中，杜绝"作弊"</li>
<li><strong>Git 作为状态机</strong> — commit = 实验记录，reset = 丢弃坏实验</li>
<li><strong>固定实验预算</strong> — 每次实验成本恒定，结果直接可比</li>
<li><strong>永不停止的自主性</strong> — 跑到人类手动中断，绝不请求许可</li>
<li><strong>简单性偏好</strong> — 在保持性能的前提下删代码，算一次成功的实验</li>
<li><strong>grep 友好的输出</strong> — 每行一个指标，解析零成本</li>
</ol>

<h2>文件结构</h2>

<pre><code>autoresearch/
├── .claude-plugin/
│   └── plugin.json          # Skill 元数据
├── skills/
│   └── autoresearch/
│       └── SKILL.md         # 完整 skill 逻辑（~700 行）
└── README.md                # 你在这里
</code></pre>

<h2>边界情况处理</h2>

<table>
<tr><th>场景</th><th>Agent 行为</th></tr>
<tr><td>实验崩溃</td><td>读取最后 50 行日志；typo 类快速修复，思路性错误直接跳过</td></tr>
<tr><td>连续 3 次崩溃</td><td>停止并报告——说明存在系统性问题</td></tr>
<tr><td>连续 10+ 次 discard</td><td>检测停滞；尝试更大、更激进的改动</td></tr>
<tr><td>资源耗尽</td><td>自动缩减规模，并记录资源使用</td></tr>
<tr><td>正确性约束被打破</td><td>无论主指标多好，一律 discard</td></tr>
<tr><td>超时</td><td>杀掉进程，标记 crash，回退</td></tr>
</table>

<h2>许可证</h2>

<p>MIT — 与原始 <a href="https://github.com/karpathy/autoresearch">karpathy/autoresearch</a> 一致。</p>

<h2>致谢</h2>

<p>模式源自 <a href="https://github.com/karpathy">Andrej Karpathy</a>。
Skill 设计与泛化由 <a href="https://github.com/init-cc">init-cc</a> 完成。</p>

</div>
