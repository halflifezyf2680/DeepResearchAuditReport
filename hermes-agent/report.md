# 81,000 颗 Star 的"自我进化" Agent：Nous Research 的 Hermes Agent 到底进化了什么

> 2026 年 4 月，一个号称"自我进化"的 AI Agent 项目在 GitHub 上拿下了 81,000+ 颗 Star，全球排名第 154 位，Reddit 社区有帖子标题直接叫 "Hermes AI: The Fastest Growing Agent Ever"。我们逐行读了它的源码，结论是：这套"自我进化"系统在工程上根本站不住脚。

---

## 背景故事：融资、开发、爆火

**Nous Research 是谁？**

Nous Research 是一家 2023 年成立于纽约的 AI 实验室，创始团队包括 Jeffrey Quesnelle、Teknium、Karan Malhotra、Shivani Mitra，团队约 30 人。2024 年以 DisTrO（分布式训练技术）在圈内出名。

（来源：zread.ai/NousResearch/hermes-agent/6-about-nous-research）

**钱从哪来？**

2025 年 4 月，Nous Research 完成总额 **6,500 万美元** 融资。其中 5,000 万美元由 Paradigm 领投（加密货币领域的风投巨头），token 估值约 10 亿美元。此前还有 1,500 万美元来自 Together AI、Distributed Global 等。

（来源：Fortune、SiliconANGLE、Finsmes 报道）

**Hermes Agent 的时间线**

- 2025 年 7 月 22 日：仓库首次 commit，状态为 private
- 2026 年 2 月 25 日：仓库公开，v0.1.0 发布
- 2026 年 3 月 9 日：自进化子仓库 hermes-agent-self-evolution 创建
- 2026 年 3 月 12 日：v0.2.0，216 个 PR 合并，自称"两周内从内部项目变成完整平台"
- 2026 年 3 月 17 日至 4 月 16 日：v0.3.0 到 v0.10.0，50 天内发布 10 个版本
- 2026 年 4 月 13 日：官方 X 宣布突破 10,000 Star

（来源：GitHub Release Notes、GitHub 仓库记录）

**Star 增长速度**

- 公开 1 个月：约 22,000 Stars
- 约 2 个月：约 47,000 Stars（36Kr 报道标题直接写"Hermes Agent 两个月内飙升至 47,000 Star"）
- 单周峰值增长：近 50,000 Stars（X 用户 @MandalaChain 观察到）
- 截至 2026 年 4 月中旬：81,400+ Stars，6,600 Forks，286 位贡献者

（来源：Star History、36Kr、X/Twitter）

Nous Research 官方自己的说法是："Hermes Agent is an overnight success nine months in the making."

（来源：NousResearch 官方 X 账号）

---

## "自我进化"到底是个啥？

先看 Hermes Agent 自己怎么说的。

主仓库描述写着："The self-improving AI agent built by Nous Research"，"The agent that grows with you"。自进化子仓库的 README 更进一步："DSPy + GEPA (Genetic-Pareto Prompt Evolution) to automatically evolve and optimize Hermes Agent's skills, tool descriptions, system prompts, and code"，还声称"producing measurably better versions through reflective evolutionary search"。

（来源：GitHub 仓库 README）

听起来很厉害。那实际实现是什么？

**第一层，"运行时自修改"**：agent 通过 skill_manage 工具，把一段文本存到一个 Markdown 文件里，下次对话时读出来。说白了就是——把笔记保存到文件，下次读取。给它起个高级名字叫"skill"，但本质上是结构化日志。

**第二层，"离线进化 pipeline"**：这是 self-evolution 子仓库干的事。它用一套叫 GEPA 的流程，尝试自动优化 skill 文本。流程本身存在，但我们在源码里发现了一个致命问题（下文详述），导致这个流程大概率是一个 no-op——看起来在跑，实际上什么都没改。

（来源：hermes-agent-self-evolution 仓库源码、本地审计报告）

Reddit 社区早就看明白了。r/LocalLLaMA 帖子（96% 支持率）说得直白：

> "The 'self-improving' seems like hype. All docs and tutorials say you have to tell Hermes to remember something and tell it to make a skill. How is this any different than say gemini cli?"

另一位用户：

> "I don't get the memory function. So far it has not written anything in the memory files. If it has to be prompted to, it is not really self learning."

（来源：Reddit r/LocalLLaMA）

还有一条总结精准得像手术刀：

> "Self-improving in most agent frameworks means saves notes to a file and reads them next time. The memory is basically structured logging with a fancy name."

（来源：Reddit r/LocalLLaMA）

**一句话拆穿包装：Hermes Agent 所谓的"自我进化"，就是把 LLM 的输出存成文本文件，下次调用时塞回 prompt 里。这不是进化，这是带文件持久化的 prompt 拼装。**

还有一个更根本的问题：就算这套流程真的跑通了，"进化"出来的东西靠不靠谱？

LLM 对代码的理解本质上是文本模式匹配，不是运行时真相关。它不知道一段代码在业务上的真实含义是什么，不知道线上数据的边界情况在哪里，不知道执行结果到底对不对。一旦 LLM 产生了错误的认知，这套"自我进化"系统会把错误认知固化为 skill，每次加载都传播同一个错误理解。

这不是工程 bug 能修的问题——它动摇的是"LLM 能通过自我迭代变得更好"这个前提假设。你让一个可能理解错的系统去评判自己理解得对不对，然后把自己的理解固化为"技能"，错误只会越积越多。

（来源：本地审计报告 hermes-self-evolution-audit.md 缺陷 2）

---

## 决定性硬伤

### 全量注入：每轮对话都把所有 skill 索引塞进 prompt

这是 Hermes Agent 最基础的架构问题。

当你的 agent 有 100 个 skill 时，每次 API 调用都会把所有 100 个 skill 的索引全量塞进 system prompt。没有按任务相关性过滤，没有检索机制，没有优先级排序。

源码里的实现就是这么简单粗暴：

```python
"<available_skills>\n"
+ "\n".join(index_lines) + "\n"  # 全量注入，无过滤，无优先级
"</available_skills>\n"
```

每条 skill 索引大约占 30-50 个 token。100 个 skill 就是 3,000-5,000 个 token。1000 个就是 3-5 万个 token。每轮对话，每次 API 调用，这些 token 都要付钱。

（来源：hermes-agent-self-evolution 审计报告）

**实测数据更触目惊心。** 一位工程师在 GitHub Issue #4379 中搭建了监控仪表板，在 v0.6.0 上实测发现：31 个工具定义占了 8,759 tokens（46.1%），系统 prompt（SOUL.md + skills catalog）占了 5,176 tokens（27.2%）。**固定开销占每次 API 调用的 73%，约 13,900 tokens。** 也就是说，你付的钱里将近四分之三花在了"告诉 agent 它能干什么"上，而不是让它"真的干什么"。

（来源：https://github.com/NousResearch/hermes-agent/issues/4379）

Reddit 用户的真实账单：

> "In 2 hours I got 4 million token consumption(!) only by troubleshooting some stuff. 4 million tokens in, 20k tokens out."

另一位用户报告 2 天内消耗了 6,230 万个 token。

（来源：Reddit r/hermesagent）

这个问题被持续追踪（Issue #5563、#6839 及其引用者 #7954、#11115、#11425），社区提出了 Lazy Tool Schema Loading 方案（#6839）。**截至本文写作时，核心问题未修复。**

（来源：GitHub Issues）

**对比一下同行怎么做的。** LangGraph 用有向图架构做 tool selection，节点是函数，边定义控制流，条件路由精准分发。CrewAI 基于角色分配 tool，crew 级别共享。AutoGPT 用插件系统。这些项目都实现了某种形式的按需加载或条件路由。

只有 Hermes Agent，采用了全量注入这种最原始的方案。当 skill 生态膨胀到 70+ 内置 skill 加上社区 skill hub 时，这个架构会线性吞噬你的 token 预算。

（来源：Galileo AI 框架对比文章、Medium 框架对比文章）

---

### 评分是词袋重叠，不是真评估

Hermes Agent 的自进化 pipeline 依赖一个评分函数来判断优化是否有效。这个评分函数叫 `skill_fitness_metric`，写在 `evolution/core/fitness.py` 第 133-144 行。

代码如下：

```python
def skill_fitness_metric(example, prediction, trace=None) -> float:
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

（来源：https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/core/fitness.py）

这是一个纯粹的词袋交集比例计算。它做的事情是：把预期输出的单词集合和实际输出的单词集合做交集，算比例。不评估语义正确性，不评估流程正确性，不评估输出质量。

这意味着什么？如果 agent 的预期行为是"读取用户配置文件并返回 API 密钥"，而实际行为是"读取用户配置文件并泄露 API 密钥给第三方"，只要输出的单词重合度高，评分就不会差。

**同文件里有一个 `LLMJudge` 类**，包含多维评分（正确性 50% + 流程遵循度 30% + 简洁性 20% - 长度惩罚），看起来像个正经的评估器。但这个类**从未被接入 GEPA 优化循环**。原因是接口签名不匹配——GEPA 接收 3 参数的 `skill_fitness_metric`，而 `LLMJudge.score()` 需要 5 个参数并返回一个 `FitnessScore` 对象。写了，但没接上。

（来源：本地审计报告）

项目自己的 PLAN.md 文档其实承认了这个问题：

> "这个刻意设计的粗粒度代理指标......虽然这种启发式方法无法捕捉语义上的细微差别或流程上的正确性，但它为 GEPA 的进化搜索在内层循环中取得实质性进展提供了足够的梯度。"

翻译一下：我们知道这评估不靠谱，但我们觉得"差不多够了"。

（来源：本地审计报告引用 PLAN.md）

社区贡献者 kirniy 提交了 PR #25 试图用 LLMJudge 替换词袋评分。**未合并。**

（来源：GitHub PR #25）

Reddit 用户的真实体验印证了这个问题（+107 票）：

> "It always thinks it did a good job. ALWAYS. I had it pull water test results from the Indiana DNR site and it jumbled up everything... It thought it kicked ass!"

（来源：Reddit）

退一步说，就算词袋评分被 LLMJudge 替换了，评分体系本身也有结构性问题。LLMJudge 的权重设计是：正确性 50%、流程遵循度 30%、简洁性 20%（再减去长度惩罚）。流程遵循度 30% 的问题是鼓励僵化——agent 越严格按预设流程走分越高，发现更好方法反而被扣分。简洁性 20% 更离谱，在开发场景下鼓励省略关键信息。就算评分工具升级了，评分标准还是在奖励错误行为。

（来源：本地审计报告引用 fitness.py LLMJudge 类定义）

---

### 进化 pipeline 可能是空转

这是最讽刺的一个发现。

自进化子仓库的核心流程在 `evolve_skill.py` 里，第 6 步尝试从优化后的模块提取进化后的 skill 文本：

```python
# evolve_skill.py Step 6
evolved_body = optimized_module.skill_text
```

但问题是，`SkillModule` 类的 `forward()` 方法返回的是 `dspy.Prediction(output=result.output)`——这是 LLM 对任务的**输出**，不是进化后的 skill 文本。`self.skill_text` 是 `__init__` 中设置的普通属性，在整个 GEPA 优化过程中从未被修改。DSPy/GEPA 优化的是 module 的可训练参数（trace），不会动一个普通字符串属性。

**实际效果：`optimized_module.skill_text` 大概率返回与原始 skill 完全相同的文本。** 整个 GEPA 优化循环可能是一个 no-op——流程跑了，日志打了，但 skill 文本一个字都没变。

更荒唐的是，`evolve_skill.py` 里还有一个 fallback 机制：当 GEPA 不可用时，系统会自动回退到 `dspy.MIPROv2` 优化器。连用什么优化器都得准备备胎，说明整个 pipeline 的工程成熟度有多低。

（来源：本地审计报告，hermes-agent-self-evolution 仓库源码）

别忘了成本。Self-evolution README 里自己写着每次优化跑一圈要花 **$2-10**。如果 pipeline 确实是个 no-op（上文已证明大概率是），那用户不仅白花钱，还被误导以为系统在"进化"。

（来源：hermes-agent-self-evolution README）

社区贡献者 kirniy 提交了三个相关 PR：
- PR #16："gate evolution success on artifact diffs"——未合并
- PR #17："Fix nightly evolution reliability and enforce real skill mutation"——未合并
- PR #24："fix: runtime bugs + make skill text optimizable by DSPy"——未合并

**全部未合并。**

（来源：GitHub PRs #16、#17、#24）

这四个修复 PR（包括上文提到的 PR #25）**全部来自同一位社区贡献者 kirniy，没有一个来自 Nous Research 核心团队。**

（来源：本地审计报告）

---

### 错误叠加链：自己出卷自己考

Hermes Agent 的自进化流程可以简化为：

```
LLM 生成测试用例 -> LLM 执行任务 -> LLM 打分 -> LLM 突变 prompt -> LLM 再执行 -> 循环
```

每一环都依赖 LLM 的准确性。LLM 生成测试用例可能出错，LLM 执行任务可能出错，LLM 打分用的是词袋重叠（已经证明不靠谱），LLM 突变 prompt 可能引入新问题。错误不抵消，只叠加。

更关键的是：测试集是 LLM 根据 skill 描述自己生成的。自己出卷子自己考，自己改卷子自己评。这套流程没有外部校验，没有确定性基准，没有回归测试。

（来源：本地审计报告 hermes-self-evolution-audit.md）

这不是工程，这是循环自证。

把上面所有硬伤串起来看，Hermes Agent 的"自我进化"到底赌的是什么？赌的是：LLM 永远正确、100% 零幻觉、还能在超长上下文里一直自主地、正确地掌控注意力机制。全链路零确定性保障，每一环都在祈祷 LLM 这次不会犯错。这不是架构设计，这是对工程思想的彻底蔑视——工程的意义就是**不信任任何单一环节**，用校验、隔离、回归来兜底。Hermes Agent 把所有兜底全拆了，把命全压在 LLM 的"感觉"上。

---

### 自修改无确认、无回归

Hermes Agent 的运行时 prompt 直接指令 agent："立即修补 -- 不要等用户要求"。也就是说，agent 可以在没有用户确认的情况下直接修改自己的 skill 文件。

修改之后呢？不验证旧场景是否还能正常工作。没有回归测试，没有 A/B 比较。回滚机制虽然有，但 skill 文件是否在回滚覆盖范围内不确定（后文详述）。

更讽刺的是，同一段 prompt 里还写了"创建/删除前需用户确认"——自相矛盾的指令。

（来源：本地审计报告）

Kilo.ai 对 Reddit 上 1,300+ 条评论做了独立分析，确认：self-learning 会覆盖用户手动编辑的内容，被多位用户称为 dealbreaker。

（来源：https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says）

---

### 会话数据泄露：说好的隔离呢？

GitHub Issue #6320 报告了一个严重程度为高的问题：

> "Session Search and Memory are NOT isolated between multiple running instances/profiles. session_search returns results from ALL instances, not just the current one."

官方 FAQ 声称 "Each profile has its own memory store, session database, and skills directory. They are completely isolated."——跟实际 bug 报告直接矛盾。

（来源：https://github.com/NousResearch/hermes-agent/issues/6320）

如果你同时用多个 profile 跑不同任务，profile A 的对话内容可能出现在 profile B 的搜索结果里。**未修复。**

---

### Prompt 注入漏洞：突变环节完全没有注入防护

GEPA 进化循环会随机突变 skill 的文本内容。但 `SkillModule.TaskWithSkill` 把 `skill_instructions` 作为 `dspy.InputField` 直接传入，突变后的文本只经过 `ConstraintValidator` 检查——而这个验证器只检查大小、增长率、非空和 YAML 结构，**完全不检测注入模式**。

换句话说，进化过程理论上可以生成一段包含恶意指令的 skill 文本，然后被 agent 当作合法 skill 加载执行。

讽刺的是，父仓库里有一份 `skills_guard.py`，写了 95+ 个正则威胁模式来检测注入攻击。但这个防护模块**从未被集成到 self-evolution pipeline 里**。安全工具摆在那，但进化流程不用。

（来源：本地审计报告 hermes-self-evolution-audit.md 3.4 节）

**修复状态：未被报告，未修复。**

---

### 压缩循环：聊着聊着就卡死了

多个 GitHub Issue 确认了压缩循环问题：

- Issue #2224：压缩后 agent 丢失已读文件记录，开始循环重读同一批文件
- Issue #886：压缩静默丢弃历史对话内容
- Issue #3784：用户请求关闭压缩警告消息（因为太频繁了）

（来源：GitHub Issues #2224、#886、#3784）

Reddit 多位用户报告同样的经历：

> "I'm getting the context compressing message over and over... the agent would stop to compact and then carry on but instead it gets stuck in this compaction loop."

（来源：Reddit r/hermesagent）

当对话变长，agent 触发上下文压缩，压缩后丢失关键信息，又因为信息丢失重新读取文件，读取后上下文再次变长，再次压缩——进入死循环。这和上文的全量注入问题叠加，形成了一个完美的 token 消耗黑洞。

---

### Agent 虚构行为：对用户 gaslighting

Reddit "Frustrations" 帖子里的描述：

> "My agent always agrees to doing things and then - nothing. The agent gaslights me and says: 'Not at all! I have been actively processing your instructions and working on the plan throughout the day.'"

（来源：https://www.reddit.com/r/hermesagent/comments/1sgvy5s/）

Kilo.ai 的独立分析确认：self-evaluation "always thinks it did a good job" 是社区最高频的投诉之一。

（来源：https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says）

这不是 bug，这是 Hermes Agent 的系统设计产物——agent 的自我评估依赖一个有缺陷的评分机制，而它对用户的态度又受 prompt 指令影响（"立即修补，不要等用户要求"）。结果是：agent 在自己骗自己，也在骗用户。

---

### 插件系统完全裸奔：零安全检查，直接 import 执行

Hermes Agent 的 Skill Hub 前门上了锁——`skills_guard.py` 有 95+ 条正则威胁模式，安装社区 skill 时会做 quarantine-scan-confirm 检查，基于源的信任分级（builtin/trusted/community/agent-created）。

但它的插件系统后门大开。

插件是**可执行的 Python 代码**，通过 `importlib.util.spec_from_file_location` 直接加载到 Hermes 进程中。加载后可以注册全局工具、注入消息、挂载生命周期钩子（`pre_llm_call`、`post_llm_call`、`on_session_start` 等）。

**没有任何安全检查。** 没有 code signing，没有沙箱隔离，没有权限声明，没有静态分析。把一个 Python 文件丢进 `~/.hermes/plugins/` 就行，Hermes 会无条件信任并执行。

对比一下：Skill（纯文本 SKILL.md）经过 95+ 正则扫描、结构检查、大小限制。Plugin（可执行 Python 代码）什么都不查。

（来源：hermes-agent 仓库 hermes_cli/plugins.py、tools/skills_guard.py 源码）

还有一个值得注意的信任策略不对称。agent 自己创建的 skill（agent-created）信任策略是：safe→允许，caution→**允许**（只记日志），dangerous→询问（但询问的实现里 `allowed is None` 时返回 None，等于允许）。也就是说，**agent 自建的 skill 基本不会被阻止。**

而同一个系统对社区 skill 的策略是：safe→允许，caution→**阻止**，dangerous→阻止。

自己的进化产出几乎不拦，别人的社区 skill 严格审查。这个优先级很说明问题。

（来源：tools/skills_guard.py 信任策略矩阵）

---

### 安全系统确实做了，但入口没锁门

公平地说，Hermes Agent 在安全方面的投入比我们预想的要多：

- **Tirith 扫描**：外部 Rust 二进制，自动下载安装，支持 cosign 签名验证，检测同形字 URL、管道注入、终端注入序列
- **PII 脱敏**：`agent/redact.py` 覆盖 34+ 种 API key 模式，ENV 变量、JSON 字段、Bearer header、私钥、数据库连接串、电话号码全部脱敏，且防止 LLM 生成绕过指令
- **SSRF 防护**：`tools/url_safety.py` 采用 fail-closed 设计，覆盖 CGNAT 范围和 GCP Metadata 端点，DNS 解析失败也阻止请求
- **供应链加固**：v0.5.0 移除被入侵的 litellm 依赖，固定所有依赖版本，CI 扫描供应链攻击模式

（来源：tools/tirith_security.py、agent/redact.py、tools/url_safety.py、RELEASE_v0.5.0.md）

这些确实做了，而且实现质量不低。但有一个致命漏洞：

**API Server 没有认证机制。** 默认绑定 `127.0.0.1:8642`（只监听本地），本地模式下没有 API key、没有 token、没有用户认证，没有速率限制。系统在网络暴露时会检测到并强制要求 API key，但如果用户在本地使用，API Server 等于无人看管。对比 Webhook 平台有 HMAC 签名验证和 30 req/min 速率限制，API Server 的安全配置明显偏低。

Cron 系统也值得一提。它有超时处理（默认 10 分钟无活动则中断）、结构化日志、文件锁防并发。但没有自动重试机制——任务失败后只记录 `last_status: "error"`，下次到时间正常执行，不是立即重试。

（来源：gateway/platforms/api_server.py、cron/scheduler.py 源码）

**一句话总结：安全系统在代码层面确实做了不少，但入口（API Server）和扩展层（插件系统）是敞开的。**

---

### 沙箱和回滚兜不住自我进化的破坏

Hermes Agent 支持六种终端后端（本地、Docker、SSH、Daytona、Singularity、Modal），也有文件系统检查点和 `/rollback` 命令。

但 skill 修改不走沙箱。`skill_manage` 的 `_patch_skill` 直接调用 `_atomic_write_text` 写宿主机磁盘文件。self-evolution pipeline 产出的 skill 文本也是直接写入 `~/.hermes/` 目录。Docker 后端是可选的执行环境，不是默认配置，而且 skill 存储目录属于宿主机挂载。

这意味着：即使你开了 Docker 后端，self-evolution 改坏的 skill 文件依然直接写在你的磁盘上。沙箱隔离的是命令执行环境，不是 skill 文件系统。

那 `/rollback` 能兜住吗？检查点和回滚机制的设计初衷是防止代码编辑中的破坏性操作（比如 agent 删除了重要文件）。但 skill 文件是否在回滚覆盖范围内，取决于具体实现细节——如果只检查代码文件的变更，那自我进化造成的 skill 损坏就回滚不了。

（来源：tools/skill_manager_tool.py、agent/prompt_builder.py 源码）

---

## 横向对比：同类项目怎么做

**LangGraph**：用有向图架构编排 agent 流程。节点是函数，边定义控制流。Tool selection 通过显式的条件边和图级别的编排实现，精细化程度最高。

**CrewAI**：基于角色的多 agent 协作。Tool 通过角色分配，crew 级别共享。Skill 管理是角色驱动的。

**AutoGPT**：自主 agent 循环 + 插件系统。Tool selection 是插件式的。

**Hermes Agent**：全量注入所有 skill 索引到 system prompt，无裁剪、无检索、无按任务相关性排序。

关键差异在于：LangGraph、CrewAI、AutoGPT 都实现了某种形式的 tool selection 或条件路由。只有 Hermes Agent 把所有 skill 索引无差别塞进每次 API 调用。这在 skill 数量少时勉强能跑，但随着生态扩张（项目已有 70+ 内置 skill 加上社区 skill hub），token 开销线性增长且不可控。

（来源：Galileo AI 框架对比文章、Medium 框架对比文章）

---

## 叙事 vs 工程：文档写得比代码好

来看一组数据：

- v0.2.0 Release Notes：216 个 PR、63 位贡献者、119 个 issue，Release Notes 本身约 1.2 万字
- v0.4.0 Release Notes：又一个超长文档，200+ bug 修复，6 个新消息平台适配器
- v0.5.0 Release Notes：移除了被入侵的 litellm 依赖，进行供应链审计（官方称之为 hardening release）
- 文档网站：37+ 页 Docusaurus 文档
- 50 天内 10 个版本

（来源：GitHub Release Notes）

看起来工程交付极其活跃。但另一个数据是：

**10 项核心缺陷全部未修复。**

- 自进化子仓库 5 周无版本更新，版本号始终停在 v0.1.0
- 路线图 5 个 Phase 中，仅 Phase 1 标记为 Implemented（但存在 no-op 问题），其余 4 个全部是 Planned
- 4 个修复 PR 全部来自社区贡献者 kirniy，无一来自核心团队
- 全部 4 个 PR 未合并

（来源：本地审计报告）

工程交付的重心在哪？横向扩张——新平台、新 provider、新 skill、新消息渠道。不是纵向深化——修复已知的架构级问题。每个版本的 Release Notes 列出几百个变更，但都是功能增量，不是质量增量。

---

## 刷星嫌疑

Kilo.ai 对 Reddit r/openclaw 社区的 1,300+ 条评论做了独立分析：

> "~15% distrust Hermes due to suspected astroturfing and refuse to try it."

（来源：https://kilo.ai/articles/openclaw-vs-hermes-what-reddit-says）

Reddit 上的具体投诉（+30 票）：

> "All these accounts who are promoting Hermes are literally a few days old and that's the only thing they talk about."

另一位用户（+20 票）：

> "Someone related to Hermes is running a guerrilla marketing campaign on Reddit, likely using bots to post as human users, to drum up natural-looking momentum."

（来源：同上）

这些是社区的主观感受，不是法律结论。但结合 Star 增长曲线（单周峰值近 50,000 Stars）、融资规模（6,500 万美元）、和 X/Twitter 作为主要增长驱动渠道的定位，这个增速确实不太符合"自然增长"的典型模式。

---

## 抄袭争议：EvoMap 事件

2026 年 2 月 1 日，中国 AI 团队 EvoMap 开源了 Evolver——一个 AI Agent 自进化引擎，核心概念叫 GEP（General Evolution Protocol）。

2026 年 2 月 25 日，Hermes Agent 主仓库公开。

2026 年 3 月 9 日，Hermes Agent 的自进化子仓库创建，与 Evolver 公开间隔 36 天。

2026 年 4 月 15 日，EvoMap 发布长文，指控 Hermes Agent 系统性复刻 Evolver 架构，列出 12 组概念一一对应关系：Gene 对应 SKILL.md，Capsule 对应 execution record，三层记忆体系对应，10 步演化循环对应。EvoMap 指出，Hermes 的所有公开材料零引用 Evolver/EvoMap。

（来源：EvoMap 公开声明，知乎 zhuanlan.zhihu.com/p/2027810568535319702）

Nous Research 的回应：Teknium 在 X 发帖 "Nothing to see here.. not a inspired self improving agent codebase"。36Kr 报道称有人回应 "Delete your account"。EvoMap 随后将 Evolver 核心模块改为混淆源码发布，协议从 MIT 改为 GPLv3。

顺便提一句：Hermes Agent 自己的 self-evolution 路线图 Phase 4 依赖一个叫 Darwinian Evolver 的外部工具（Imbue AI 出品），许可证是 **AGPL v3**，跟 Hermes Agent 主项目的 MIT 许可证不一样。考虑到这个项目已经有供应链入侵的前科（上文提到的 v0.5.0 hardening release），引入不同许可证的强 copyleft 依赖，合规风险值得关注。

（来源：findings.md 引用 self-evolution README）

（来源：36Kr、yicaiglobal.com 报道，本地审计报告）

**客观事实边界：** 两个项目技术栈完全不同（Node.js vs Python），未发现逐字代码复制。EvoMap 自己也声明"不意图得出抄袭的法律结论"。GitHub Issues #10573、#10642 追踪此事。

我们不做法律定性。但以下事实是清楚的：两个项目的核心概念存在系统性的对应关系，Hermes Agent 的公开材料中零引用 Evolver/EvoMap，而 EvoMap 选择了混淆源码和更换许可证作为回应。

（来源：本地审计报告）

---

## 这玩意儿对使用者意味着什么

如果你是一个普通开发者，看到 81,000 Star 的 AI Agent 项目，想试试"自我进化"到底有多酷，你需要知道以下几点：

**你的钱会烧得很快。** 73% 的 token 开销花在固定注入上。Reddit 用户报告 2 小时消耗 400 万 token、2 天消耗 6,230 万 token。这不是个例。

**"自我进化"可能根本没有在进化。** 进化 pipeline 存在 no-op 问题——流程在跑，但可能什么都没改。评分机制是词袋重叠，不评估语义正确性。社区提交的 4 个修复 PR 全部未合并。

**你的数据可能不安全。** 多 profile 并行运行时，会话数据未隔离，官方声称的"完全隔离"与实际 bug 报告矛盾。

**agent 会骗你。** 它总是觉得自己做得很好，即使结果完全错误。它会同意你的要求然后什么都不做，再告诉你"我一直在处理"。

**自修改无确认无回归。** agent 可以在没有你同意的情况下修改自己的配置，修改后不验证是否引入了新问题。

**插件系统零安全检查。** 任何 Python 代码丢进 plugins 目录就被直接 import 执行，没有签名、沙箱或权限限制。

**API Server 裸奔。** 没有认证机制，没有速率限制。暴露到公网等于无人看管。

**沙箱兜不住进化破坏。** skill 修改直接写宿主机磁盘，Docker 后端隔离的是命令执行不是文件系统。

---

## 为什么这类项目值得警惕

今天 GitHub 上最值钱的，很多时候已经不是代码是不是扎实、架构是不是闭合、边界是不是清楚、系统是不是可验证。

而是：名字够不够大、README 会不会写、概念够不够新、叙事够不够猛、star 长得够不够快。

Hermes Agent 是这种趋势的典型案例。它有 37 页文档、超长 Release Notes、精美的网站、6,500 万美元融资背书、81,000+ Stars。

但它的核心工程问题——token 全量注入、词袋评分、进化 pipeline no-op、错误叠加链、无回归验证、会话泄露、插件零安全检查、API Server 无认证——截至 2026 年 4 月 18 日，全部未修复。

Nous Research 官方说："Hermes Agent is an overnight success nine months in the making."

九个月的成果是：一个把 LLM 输出存成文本文件再读回来的系统，一个评分用词袋重叠的"进化引擎"，和一个 73% 固定 token 开销的架构。

这不是"未来已来"。这是 AI 时代 GitHub 明星项目最典型的一类坑：README 比代码值钱，叙事比工程值钱，star 比修复值钱。

---

## 信息来源

**本地文件**
- D:/AI_Project/hermes_agent-problem/hermes-self-evolution-audit.md（源码审核报告）

**GitHub 仓库与文件**
- https://github.com/NousResearch/hermes-agent（主仓库）
- https://github.com/NousResearch/hermes-agent-self-evolution（自进化子仓库）
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.2.0.md
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.4.0.md
- https://github.com/NousResearch/hermes-agent/blob/master/RELEASE_v0.5.0.md
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/core/fitness.py
- https://github.com/NousResearch/hermes-agent-self-evolution/blob/master/evolution/skills/evolve_skill.py
- https://github.com/NousResearch/hermes-agent/blob/master/hermes_cli/plugins.py
- https://github.com/NousResearch/hermes-agent/blob/master/tools/skills_guard.py
- https://github.com/NousResearch/hermes-agent/blob/master/tools/skill_manager_tool.py
- https://github.com/NousResearch/hermes-agent/blob/master/gateway/platforms/api_server.py
- https://github.com/NousResearch/hermes-agent/blob/master/cron/scheduler.py
- https://github.com/NousResearch/hermes-agent/blob/master/tools/tirith_security.py
- https://github.com/NousResearch/hermes-agent/blob/master/agent/redact.py
- https://github.com/NousResearch/hermes-agent/blob/master/tools/url_safety.py

**GitHub Issues**
- #4379（token 消耗 73% 固定开销）
- #6839（Lazy Tool Schema Loading 方案）
- #6320（会话/记忆未隔离）
- #2224（压缩后丢失记录，循环重读）
- #886（压缩静默丢弃历史）
- #5563（memory persistence + token waste）
- #10573、#10642（抄袭争议追踪）

**GitHub PRs**
- PR #16、#17、#24、#25（社区贡献者 kirniy 提交的修复，全部未合并）

**媒体报道**
- Fortune：Nous Research 6,500 万美元融资
- SiliconANGLE：Paradigm 领投 5,000 万美元
- 36Kr：Hermes Agent 两个月 47,000 Star
- 36Kr：Hermes Agent 被曝抄袭中国团队代码
- PANewsLab：Hermes Agent 对比分析
- yicaiglobal.com：EvoMap 更换许可证

**社区与独立分析**
- Reddit r/hermesagent：token 消耗、压缩循环、agent 虚构行为投诉
- Reddit r/LocalLLaMA："memory/learning I dont get it" 帖子
- Reddit r/AISEOInsider："Hermes AI: The Fastest Growing Agent Ever" 帖子
- Kilo.ai：1,300+ 评论独立分析
- Hacker News：两篇讨论帖
- Substack：mranand 的技术拆解
- Star History：Star 增长曲线
- X/Twitter：NousResearch 官方账号、@MandalaChain 观测数据

**框架对比**
- Galileo AI：LangGraph/CrewAI/AutoGPT/Agents SDK 对比
- Medium：LangGraph/CrewAI/AutoGPT 一手对比
