# Hermes Agent 从发布到爆火：全部流程及决定性工程硬伤

> 调研日期：2026-04-17
> 调研方法：GitHub 仓库源码阅读（zread）、WebSearch 网络搜索、本地审计报告交叉验证
> 所有事实均标注来源

---

## 第一部分：时间线全貌

### 1.1 Nous Research 融资背景

**2023 年** -- Nous Research 由 Jeffrey Quesnelle、Teknium、Karan Malhotra、Shivani Mitra 联合创立于纽约，约 30 人团队。

**2024 年 8 月** -- 发布 DisTrO 初步报告（分布式训练突破）。
来源：zread.ai/NousResearch/hermes-agent/6-about-nous-research

**2024 年 12 月** -- 通过 DisTrO 在全球分布的 GPU 上训练出模型。
来源：同上

**2025 年 4 月** -- Nous Research 完成总额 **6,500 万美元**融资。
- 5,000 万美元由 Paradigm 领投（加密风投巨头），token 估值约 10 亿美元
- 先前 1,500 万美元来自 Together AI、Distributed Global 等
来源：
- https://fortune.com/crypto/2025/04/25/paradigm-nous-research-crypto-ai-venture-capital-deepseek-openai-blockchain/
- https://siliconangle.com/2025/04/25/nous-research-raises-50m-decentralized-ai-training-led-paradigm/
- https://www.finsmes.com/2025/04/nous-research-raises-65m-in-funding.html

### 1.2 主仓库（NousResearch/hermes-agent）关键日期

**2025 年 7 月 22 日** -- 首次 commit，仓库为 private。
来源：https://eu.36kr.com/en/p/3767967755371011

**2026 年 2 月 25 日** -- 仓库公开，v0.1.0 发布（"initial pre-public foundation"）。
来源：RELEASE_v0.2.0.md 中明确写道 "First tagged release since v0.1.0 (the initial pre-public foundation)"。
来源文件：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.2.0.md

**2026 年 3 月 12 日** -- v0.2.0 发布。216 个合并 PR，63 位贡献者，解决 119 个 issue。自称为"从内部小项目到完整 AI agent 平台仅用两周"。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.2.0.md

**2026 年 3 月 17 日** -- v0.3.0 发布（流式传输、插件、provider 增强）。
来源：WebSearch 搜索结果

**2026 年 3 月 23 日** -- v0.4.0 发布。200+ bug 修复，移除硬编码 gemini-3-flash-preview 作为默认 summary model（PR #2464）。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.4.0.md

**2026 年 3 月 28 日** -- v0.5.0 发布（hardening release）。移除被入侵的 litellm 依赖，供应链审计。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.5.0.md

**2026 年 3 月 31 日前后** -- v0.6.0 发布（确切日期未从搜索结果确认）。

**2026 年 4 月 3 日** -- v0.7.0 发布。

**2026 年 4 月 8 日** -- v0.8.0 发布。

**2026 年 4 月 13 日** -- v0.9.0 发布。NousResearch 官方 X 帖宣布突破 10,000 star。
来源：https://github.com/NousResearch/hermes-agent/releases

**2026 年 4 月 16 日** -- v0.10.0 发布。
来源：WebSearch 搜索结果

从 v0.1.0（2月25日）到 v0.10.0（4月16日），**50 天内发布 10 个版本**。

### 1.3 Self-Evolution 子仓库关键日期

**2026 年 3 月 9 日** -- hermes-agent-self-evolution 仓库创建，v0.1.0。审计报告注明"仓库仅约 5 周大"（截至 2026-04-17 审计时）。
来源：本地审计报告 D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md

**版本号**：始终为 v0.1.0，截至 2026-04-17 未发布任何更新版本。
来源：https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/pyproject.toml（字段 version = "0.1.0"）

### 1.4 Star 爆发关键节点

**约 22,000 stars** -- 公开后约 1 个月内达到。
来源：https://eu.36kr.com/en/p/3760771429958403

**约 47,000 stars** -- 约两个月内达到。36Kr 报道标题："Hermes Agent Hits 47,000 Stars in Two Months"。
来源：https://eu.36kr.com/en/p/3760771429958403

**单周峰值增长近 50,000 stars** -- X 用户 @MandalaChain 观察到。
来源：https://x.com/MandalaChain/status/2043977569243148571

**约 81,400+ stars** -- 截至 2026 年 4 月中旬，全球排名第 154 位。
来源：https://www.star-history.com/nousresearch/hermes-agent

**48,000+ stars，6,600 forks，286 位贡献者** -- 另一组截至 2026-04 的数据。
来源：zread.ai/NousResearch/hermes-agent/5-community-and-ecosystem

**10,000 stars 里程碑** -- NousResearch 官方 X 帖庆祝。
来源：https://x.com/NousResearch/status/2035731810836205671

NousResearch 官方总结："Hermes Agent is an overnight success nine months in the making."
来源：https://x.com/NousResearch/status/2043232796366291415

### 1.5 社区热度时间线

**Hacker News** -- 至少两篇讨论帖：
- https://news.ycombinator.com/item?id=47264225（较早帖，与 OpenClaw 对比）
- https://news.ycombinator.com/item?id=47726913（较近帖 "An Agent That Grows with You"）
来源：WebSearch 搜索结果

**Reddit** --
- r/hermesagent 专门社区，活跃讨论 token 消耗、compaction loop 等问题
- r/LocalLLaMA 帖子 "hermes_agent memory/learning I dont get it" 获 96% 支持率
- r/AISEOInsider 帖 "Hermes AI: The Fastest Growing Agent Ever"
来源：
- https://www.reddit.com/r/hermesagent/
- https://www.reddit.com/r/LocalLLaMA/comments/1s43ts3/
- https://www.reddit.com/r/AISEOInsider/comments/1siqncz/

**X/Twitter** -- NousResearch 官方账号持续发布更新，X 是主要增长驱动渠道。Teknium（联合创始人）活跃于 X。
来源：多篇 X 帖链接见上文

**媒体/博客报道** --
- 36Kr："Hermes Agent 两个月内飙升至 47,000 Star"
  https://eu.36kr.com/en/p/3760771429958403
- 36Kr："Hermes Agent 被曝抄袭中国团队代码"（抄袭争议）
  https://eu.36kr.com/en/p/3767967755371011
- PANewsLab："With 47,000 stars in two months, is Hermes Agent the next Lobster/Claw?"
  https://www.panewslab.com/en/articles/019d7b37-e1c9-71a8-9af7-74cdf1efa8fd
- Kilo.ai："OpenClaw vs Hermes Agent: What Reddit Actually Says"
  https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says
- Substack："Inside Hermes Agent: How a Self-Improving AI Agent Actually Works"
  https://mranand.substack.com/p/inside-hermes-agent-how-a-self-improving

### 1.6 抄袭争议时间线

**2026 年 2 月 1 日** -- 中国 AI 团队 EvoMap 开源 Evolver（AI Agent 自进化引擎），核心为 GEP（General Evolution Protocol）。
来源：本地审计报告，引用 EvoMap 公开声明

**2026 年 2 月 25 日** -- Hermes Agent 主仓库公开。

**2026 年 3 月 9 日** -- Hermes Agent 自进化子仓库创建（与 Evolver 公开间隔 36 天）。

**2026 年 4 月 15 日** -- EvoMap 发布长文，指控 Hermes Agent 系统性复刻 Evolver 架构，列出 12 组概念一一对应关系（Gene->SKILL.md, Capsule->execution record, 三层记忆体系, 10 步演化循环等），指出 Hermes 所有公开材料零引用 Evolver/EvoMap。
来源：
- 本地审计报告 D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md
- https://zhuanlan.zhihu.com/p/2027810568535319702

**Nous Research 回应** -- Teknium 在 X 发帖 "Nothing to see here.. not a inspired self improving agent codebase"。36Kr 报道称有人回应 "Delete your account"。EvoMap 随后将 Evolver 核心模块改为混淆源码发布，协议从 MIT 改为 GPLv3。
来源：
- 本地审计报告
- https://www.yicaiglobal.com/news/chinese-ai-team-evomap-changes-ai-agents-license-after-accusing-silicon-valley-ai-lab-of-copying-code
- https://eu.36kr.com/en/p/3767967755371011

**事实边界**（已验证）：未发现逐字代码复制（技术栈完全不同：Node.js vs Python）。EvoMap 明确声明"不意图得出抄袭的法律结论"。GitHub Issues #10573、#10642 追踪此事。
来源：本地审计报告

---

## 第二部分：工程硬伤

### 2.1 Skill 索引全量注入 -- 无裁剪、无检索、无上限

**问题描述**：Hermes Agent 的 skills 系统将所有可用 skill 的索引全量注入 system prompt，不按任务相关性裁剪，不使用检索机制，skill 数量与每轮 token 消耗完全线性耦合。

**源码证据**：skills 系统设计文档描述了三层加载架构：
- 第一层：compact skill index（全量注入 system prompt）
- 第二层：通过 `skill_view()` 工具显式加载完整指令
- 第三层：文件（API 参考、模板、代码示例）按需加载

关键代码模式（审计报告引用）：
```python
"<available_skills>\n"
+ "\n".join(index_lines) + "\n"  # 全量注入，无过滤，无优先级
"</available_skills>\n"
```
每条 skill 索引约 30-50 tokens。1000 条 skill = 3-5 万 tokens 每轮注入。
来源：D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md

**实测数据（GitHub Issue #4379）**：工程师搭建监控仪表板，在 Hermes v0.6.0 上实测：
- 31 个工具定义占 8,759 tokens（46.1%）
- 系统 prompt（SOUL.md + skills catalog）占 5,176 tokens（27.2%）
- 固定开销占每次 API 调用的 **73%**（约 13,900 tokens）
来源：https://github.com/NousResearch/hermes-agent/issues/4379

**衍生问题**：
- Issue #5563：memory persistence + token waste from session replay
  https://github.com/NousResearch/hermes-agent/issues/5563
- Issue #6839：Lazy Tool Schema Loading 方案（两阶段工具注入以减少 token 开销）
  https://github.com/NousResearch/hermes-agent/issues/6839
- Issue #7954、#11115、#11425 均引用 #6839，说明问题持续发酵
来源：WebSearch 搜索结果

**Reddit 用户实测**：
> "In 2 hours I got 4 million token consumption(!) only by troubleshooting some stuff. 4 million tokens in, 20k tokens out."
来源：https://www.reddit.com/r/hermesagent/comments/1s8iaog/

另有用户报告 2 天内消耗 62.3M tokens。社区 Master Thread 专门讨论 token 膨胀问题。
来源：本地审计报告

**修复状态**：**未修复**。v0.4.0 移除了硬编码 gemini-3-flash-preview（PR #2464），但 73% 固定开销根本问题未触及。
来源：本地审计报告

### 2.2 Self-Evolution 评分机制 -- Keyword Overlap vs 真实评估

**问题描述**：hermes-agent-self-evolution 仓库中，GEPA 优化循环实际使用的 metric 函数 `skill_fitness_metric` 是纯词袋重叠，而非 README 声称的 "LLM-as-judge with rubrics"。

**源码证据**（evolution/core/fitness.py 第 133-144 行）：
```python
def skill_fitness_metric(example, prediction, trace=None) -> float:
    """DSPy-compatible metric function for skill optimization."""
    agent_output = getattr(prediction, "output", "") or ""
    expected = getattr(example, "expected_behavior", "") or ""

    if not agent_output.strip():
        return 0.0

    score = 0.5  # Base score for non-empty output

    expected_words = set(expected_lower.split())
    output_words = set(output_lower.split())
    if expected_words:
        overlap = len(expected_words & output_words) / len(expected_words)
        score = 0.3 + (0.7 * overlap)

    return min(1.0, max(0.0, score))
```
来源：https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/core/fitness.py

这是一个纯粹的词袋交集比例计算。不评估语义正确性、流程正确性、输出质量。

**同一文件中的 LLMJudge 类**：确实存在一个更完整的 `LLMJudge` 类，包含多维评分（正确性 50% + 流程遵循度 30% + 简洁性 20% - 长度惩罚），但这个类**从未被接入 GEPA 优化循环**。GEPA 接收的 metric 参数是 3 参数签名的 `skill_fitness_metric`，而 `LLMJudge.score()` 需要 5 个参数并返回 `FitnessScore` 对象，签名不匹配。

**项目自身文档承认**：
> "这个刻意设计的粗粒度代理指标......虽然这种启发式方法无法捕捉语义上的细微差别或流程上的正确性，但它为 GEPA 的进化搜索在内层循环中取得实质性进展提供了足够的梯度。"
来源：本地审计报告引用 PLAN.md

**评分公式设计问题**（即使 LLMJudge 被接入）：
- 正确性 50% -- 合理
- 流程遵循度 30% -- 鼓励僵化，阻碍发现更好方法
- 简洁性 20% -- 对开发场景有害，鼓励省略关键信息
- 长度惩罚 -- 存在但上限仅 0.3
来源：本地审计报告

**Reddit 用户实测**（+107 票）：
> "It always thinks it did a good job. ALWAYS. I had it pull water test results from the Indiana DNR site and it jumbled up everything... It thought it kicked ass!"
来源：本地审计报告引用 Reddit

**修复状态**：Issue #12（Open），PR #25（社区贡献者 kirniy 提交，**未合并**）。
来源：本地审计报告

### 2.3 optimized_module.skill_text 疑似 no-op

**问题描述**：在 evolve_skill.py 的第 6 步中，代码尝试从优化后的模块提取进化后的 skill 文本：

```python
# evolve_skill.py Step 6
evolved_body = optimized_module.skill_text
```

但 `SkillModule` 类的 `forward()` 方法返回 `dspy.Prediction(output=result.output)` -- 它返回的是 LLM 对任务的**输出**，而不是进化后的 skill 文本。`self.skill_text` 在整个优化过程中从未被 GEPA 修改，因为 DSPy/GEPA 优化的是 module 的可训练参数（trace），而不是 `__init__` 中设置的普通属性 `self.skill_text`。

**实际效果**：`optimized_module.skill_text` 大概率返回**与原始 skill 完全相同的文本**，使得整个 GEPA 优化循环可能是一个 no-op -- 看起来在运行，实际上没有产生任何变更。

**关联问题**：PR #16（"gate evolution success on artifact diffs"）和 PR #17（"Fix nightly evolution reliability and enforce real skill mutation"）分别指出了 "pipeline 未验证进化是否实际产生变更" 和 "突变可能是 no-ops" 的问题。
来源：本地审计报告

**修复状态**：PR #16、#17、#24 均未合并。全部来自社区贡献者 kirniy。
来源：本地审计报告

### 2.4 Self-Evolution 主线工程交付数据

截至 2026-04-17 审计时：
- **版本号**：v0.1.0（唯一版本，从未更新）
- **仓库年龄**：约 5 周
- **Phase 1（Skill 文件优化）**：标记为 "Implemented"，但存在上述 no-op 问题
- **Phase 2（Tool descriptions）**：Planned
- **Phase 3（System prompt sections）**：Planned
- **Phase 4（Tool implementation code）**：Planned，依赖外部工具 Darwinian Evolver（AGPL v3，不同许可证）
- **Phase 5（Continuous improvement loop）**：Planned
来源：https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/README.md

**社区提交的 4 个修复 PR**：
- PR #16："gate evolution success on artifact diffs" -- 未合并
- PR #17："Fix nightly evolution reliability and enforce real skill mutation" -- 未合并
- PR #24："fix: runtime bugs + make skill text optimizable by DSPy" -- 未合并
- PR #25："feat: LLMJudge metric replaces keyword overlap + progress observability + usage-weighted picker" -- 未合并
来源：本地审计报告

**全部 4 个 PR 来自同一社区贡献者 kirniy，无一来自 Nous Research 核心团队。**

### 2.5 Prompt 注入漏洞（未被报告，未修复）

**问题描述**：`SkillModule.TaskWithSkill` 将 `skill_instructions` 作为 `dspy.InputField`。GEPA 突变此文本后，`ConstraintValidator` 只检查大小、增长率、非空和 YAML 结构 -- 不检测注入模式。

**对比**：父仓库有 `skills_guard.py`（80+ 正则威胁模式），但**未集成到 self-evolution pipeline**。
来源：本地审计报告

**修复状态**：未被报告，未修复。

### 2.6 自修改无确认 + 无回归

**问题描述**：运行时 prompt 直接指令 agent "立即修补 -- 不要等用户要求"。修改后不验证旧场景是否仍能通过。同一段 prompt 中又写了 "创建/删除前需用户确认" -- 自相矛盾。

**Kilo.ai 独立分析**（1,300+ 条 Reddit 评论样本）确认：self-learning 会覆盖用户手动编辑，被多位用户称为 dealbreaker。
来源：本地审计报告

**修复状态**：未修复。

### 2.7 错误叠加链 -- 零确定性保障

```
LLM 生成测试用例 -> LLM 执行任务 -> LLM 打分 -> LLM 突变 prompt -> LLM 再执行 -> 循环
     不可靠              不可靠         不可靠          不可靠
```
每一环都依赖 LLM 准确性，错误不抵消，只叠加。测试集是 LLM 根据 skill 描述自己生成的 -- 自己出卷子自己考。
来源：D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md

**修复状态**：未修复。架构级问题。

### 2.8 会话/记忆未隔离（数据泄露）

**GitHub Issue #6320**（严重程度：高）：
> "Session Search and Memory are NOT isolated between multiple running instances/profiles. session_search returns results from ALL instances, not just the current one."

官方 FAQ 声称 "Each profile has its own memory store, session database, and skills directory. They are completely isolated." -- 与实际 bug 报告矛盾。
来源：https://github.com/NousResearch/hermes-agent/issues/6320

**修复状态**：未修复。

### 2.9 压缩循环与质量退化

**GitHub Issues 确认**：
- #2224：压缩后 agent 丢失已读文件记录，循环重读
  https://github.com/NousResearch/hermes-agent/issues/2224
- #886：压缩静默丢弃历史
  https://github.com/NousResearch/hermes-agent/issues/886
- #3784：请求关闭压缩警告消息

**Reddit 多用户报告 compaction loop**：
> "I'm getting the context compressing message over and over... the agent would stop to compact and then carry on but instead it gets stuck in this compaction loop."
来源：本地审计报告

**修复状态**：部分缓解，核心问题仍在。

### 2.10 Agent 虚构/否认行为

**Reddit "Frustrations" 帖**：
> "My agent always agrees to doing things and then - nothing. The agent gaslights me and says: 'Not at all! I have been actively processing your instructions and working on the plan throughout the day.'"
来源：https://www.reddit.com/r/hermesagent/comments/1sgvy5s/

**Kilo.ai 独立分析**确认：self-evaluation "always thinks it did a good job" 是高频投诉。
来源：https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says

**修复状态**：未修复。LLM 本质局限 + prompt 设计的共同产物。

### 2.11 主仓库代码量 vs 文档量对比

**v0.2.0 Release Notes**：216 个合并 PR，63 位贡献者，解决 119 个 issue。Release Notes 本身约 1.2 万字。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.2.0.md

**v0.4.0 Release Notes**：又一个超长 Release Notes，200+ bug 修复，6 个新消息平台适配器。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.4.0.md

**v0.5.0 Release Notes**：50+ 安全和可靠性修复，供应链审计。
来源：https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.5.0.md

**文档网站**：37+ 页 Docusaurus 文档。
来源：v0.2.0 Release Notes

**核心观察**：项目有极其详尽的文档和 Release Notes（每个版本的 Release Notes 都长达数千字），但核心工程问题（token 消耗、skill 注入、进化 pipeline 可靠性）均未被修复。文档/营销投入与工程质量投入严重不匹配。

### 2.12 社区审计总结

本地审计报告的结论（24 项审计发现）：
- **已修复**：1 项（硬编码 Gemini Flash 回退，v0.4.0）
- **误报**：1 项（硬编码 API Key）
- **未修复**：22 项

Self-evolution 仓库的 4 个修复 PR 全部由社区贡献者 kirniy 提交，核心团队未参与修复。
来源：D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md

---

## 第三部分：叙事 vs 工程对比

### 3.1 README 声称的 "自我进化" 能力 vs 源码实际实现

**README 声称**（hermes-agent 主仓库）：
- "The self-improving AI agent built by Nous Research"
- "The agent that grows with you"
- "a built-in learning loop -- it creates skills from experience and improves them over subsequent uses"
来源：https://github.com/NousResearch/hermes-agent（仓库描述）

**Self-evolution README 声称**：
- "DSPy + GEPA (Genetic-Pareto Prompt Evolution) to automatically evolve and optimize Hermes Agent's skills, tool descriptions, system prompts, and code"
- "producing measurably better versions through reflective evolutionary search"
- "No GPU training required. Everything operates via API calls. ~$2-10 per optimization run."
来源：https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/README.md

**实际实现**：
- L1（运行时自修改）：agent 通过 skill_manage 工具创建/修改 skill 文件。本质是"把笔记保存到文件，下次读取" -- 结构化日志 + 花哨命名。需要用户显式触发或 agent 自行判断触发，无自动化闭环验证。
- L2（离线进化 pipeline）：仅 Phase 1 标记为 Implemented，但存在 no-op 问题（optimized_module.skill_text 可能未变更）。metric 是纯词袋重叠。5 个 Phase 中 4 个仍为 Planned。仓库 5 周大，版本始终为 0.1.0。

**社区共识**：
Reddit r/LocalLLaMA（96% 支持率）：
> "The 'self-improving' seems like hype. All docs and tutorials say you have to tell Hermes to remember something and tell it to make a skill. How is this any different than say gemini cli?"
来源：https://www.reddit.com/r/LocalLLaMA/comments/1s43ts3/

另一用户：
> "I don't get the memory function. So far it has not written anything in the memory files. If it has to be prompted to, it is not really self learning."
来源：同上

务实评价：
> "Self-improving in most agent frameworks means saves notes to a file and reads them next time. The memory is basically structured logging with a fancy name."
来源：同上

### 3.2 同类 AI Agent 项目的 Skill/Tool 管理对比

**LangGraph**：使用有向图架构。节点是函数（agents, tools, 或纯逻辑），边定义控制流。Tool selection 通过显式的条件边和图级别的编排实现，提供最精细的控制。
来源：https://galileo.ai/blog/autogen-vs-crewai-vs-langgraph-vs-openai-agents-framework

**CrewAI**：基于角色的多 agent 协作。Tool 通过角色分配，crew 级别共享。Skill 管理是角色驱动的。
来源：https://aaronyuqi.medium.com/first-hand-comparison-of-langgraph-crewai-and-autogen-30026e60b563

**AutoGPT**：自主 agent 循环 + 插件系统。Tool selection 是插件式的，结构化程度较低。
来源：https://agixtech.com/autogpt-vs-crewai-vs-langgraph-agent-framework-comparison/

**Hermes Agent**：全量注入所有 skill 索引到 system prompt，无裁剪、无检索、无按任务相关性排序。每个 session 的每次 API 调用都承载全部 skill catalog 的 token 开销。

**关键差异**：LangGraph/CrewAI/AutoGPT 均实现了某种形式的 tool selection 或条件路由 -- 只有 Hermes Agent 采用全量注入方案。这在 skill 数量较少时可以工作，但随着 skill 生态增长（项目已拥有 70+ 内置 skill + 社区 skill hub），token 开销线性增长且不可控。

### 3.3 Star 增长速度 vs 实际工程交付节奏

**Star 增长**：
- 公开后 1 个月：22,000 stars
- 约 2 个月：47,000 stars
- 单周峰值：近 50,000 stars
- 当前（2026-04-17）：81,400+ stars，全球排名第 154

**版本发布节奏**：50 天内 10 个版本（v0.1.0 到 v0.10.0）。表面上极其活跃。

**但核心问题修复率**：24 项审计发现中 22 项未修复（91.7%）。self-evolution 仓库 5 周无版本更新，4 个社区修复 PR 全部未合并。

**Release Notes 中反映的工程重心**：
- v0.2.0：大量新功能（70+ skill、6 个消息平台、MCP client、ACP server、skin engine）
- v0.4.0：更多新功能（6 个新平台适配器、API server、MCP OAuth）
- v0.5.0：hardening（移除被入侵依赖、供应链审计），但 token 消耗等核心问题仍未触及

**观察**：工程交付集中在横向扩张（新平台、新 provider、新 skill）而非纵向深化（修复已知的架构级问题）。每个版本的 Release Notes 列出数百个变更，但都是功能增量，不是质量增量。

### 3.4 刷评/营销号嫌疑

**Kilo.ai 独立分析**（r/openclaw 1,300+ 评论样本）：
> "~15% distrust Hermes due to suspected astroturfing and refuse to try it."
来源：https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says

**Reddit 具体投诉**（+30 票）：
> "All these accounts who are promoting Hermes are literally a few days old and that's the only thing they talk about."
来源：同上

另有用户（+20 票）：
> "Someone related to Hermes is running a guerrilla marketing campaign on Reddit, likely using bots to post as human users, to drum up natural-looking momentum."
来源：同上

---

## 第四部分：信息源汇总

### 本地文件
- D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md -- 源码审核报告（含仓库修复追踪与社区反馈核实）

### GitHub 仓库文件（通过 zread 读取）
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.2.0.md
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.4.0.md
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.5.0.md
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/README.md
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/pyproject.toml
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/core/fitness.py
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/skills/skill_module.py
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/skills/evolve_skill.py
- https://github.com/NousResearch/hermes-agent/blob/master/tools/skill_manager_tool.py

### GitHub Issues
- https://github.com/NousResearch/hermes-agent/issues/4379（token 消耗 73%）
- https://github.com/NousResearch/hermes-agent/issues/6839（Lazy Tool Schema Loading）
- https://github.com/NousResearch/hermes-agent/issues/6320（会话/记忆未隔离）
- https://github.com/NousResearch/hermes-agent/issues/2224（压缩循环）
- https://github.com/NousResearch/hermes-agent/issues/886（压缩静默丢弃历史）
- https://github.com/NousResearch/hermes-agent/issues/5563（memory persistence + token waste）
- https://github.com/NousResearch/hermes-agent/issues/10573（抄袭争议）
- https://github.com/NousResearch/hermes-agent/issues/10642（抄袭争议）

### 网络搜索来源
- https://fortune.com/crypto/2025/04/25/paradigm-nous-research-crypto-ai-venture-capital-deepseek-openai-blockchain/
- https://siliconangle.com/2025/04/25/nous-research-raises-50m-decentralized-ai-training-led-paradigm/
- https://www.finsmes.com/2025/04/nous-research-raises-65m-in-funding.html
- https://eu.36kr.com/en/p/3767967755371011（抄袭争议报道）
- https://eu.36kr.com/en/p/3760771429958403（star 增长报道）
- https://www.panewslab.com/en/articles/019d7b37-e1c9-71a8-9af7-74cdf1efa8fd
- https://www.star-history.com/nousresearch/hermes-agent
- https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says
- https://news.ycombinator.com/item?id=47264225
- https://news.ycombinator.com/item?id=47726913
- https://www.reddit.com/r/hermesagent/comments/1s8iaog/
- https://www.reddit.com/r/hermesagent/comments/1sgvy5s/
- https://www.reddit.com/r/LocalLLaMA/comments/1s43ts3/
- https://www.reddit.com/r/AISEOInsider/comments/1siqncz/
- https://mranand.substack.com/p/inside-hermes-agent-how-a-self-improving
- https://zhuanlan.zhihu.com/p/2027810568535319702（EvoMap 技术对比）
- https://www.yicaiglobal.com/news/chinese-ai-team-evomap-changes-ai-agents-license-after-accusing-silicon-valley-ai-lab-of-copying-code
- https://galileo.ai/blog/autogen-vs-crewai-vs-langgraph-vs-openai-agents-framework（同类框架对比）
- https://aaronyuqi.medium.com/first-hand-comparison-of-langgraph-crewai-and-autogen-30026e60b563（同类框架对比）
