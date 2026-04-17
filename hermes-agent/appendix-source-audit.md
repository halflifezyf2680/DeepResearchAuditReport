# Hermes Agent 源码审计报告（附录）

> 审核对象：NousResearch/hermes-agent + hermes-agent-self-evolution
> 审核方法：通过 GitHub 仓库源码逐文件读取，提取代码证据
> 审核日期：2026-04-17
> 结论：**10 项核心缺陷全部经源码核实，无一修复**

---

## 审计范围

- **主仓库**：NousResearch/hermes-agent（agent/prompt_builder.py、tools/skill_manager_tool.py、tools/session_search_tool.py、hermes_state.py、tools/memory_tool.py）
- **子仓库**：NousResearch/hermes-agent-self-evolution（evolution/core/fitness.py、evolution/skills/skill_module.py、evolution/skills/evolve_skill.py、evolution/core/constraints.py）

---

## 缺陷 1：Skill 索引全量注入

**严重性：高 | 状态：未修复**

### 源码证据

`agent/prompt_builder.py` 的 `build_skills_system_prompt()` 函数遍历所有 skill 目录，将索引完整拼入 system prompt：

```python
# agent/prompt_builder.py
result = (
    "## Skills (mandatory)\n"
    # ...
    "<available_skills>\n"
    + "\n".join(index_lines) + "\n"
    "</available_skills>\n"
)
```

存在的过滤仅为静态过滤（平台兼容、禁用列表、工具依赖条件），不存在语义检索或按任务相关性裁剪的机制。缓存机制是全量结果的缓存，不改变全量注入的本质。

### 实测数据

GitHub Issue #4379 工程师搭建监控仪表板实测：31 个工具定义占 8,759 tokens（46.1%），系统 prompt 占 5,176 tokens（27.2%），固定开销占每次 API 调用的 73%。

（来源：github.com/NousResearch/hermes-agent/issues/4379）

---

## 缺陷 2：自修改无确认

**严重性：高 | 状态：未修复**

### 源码证据

`agent/prompt_builder.py` 中的 `SKILLS_GUIDANCE` 常量直接注入 system prompt：

```python
SKILLS_GUIDANCE = (
    # ...
    "patch it immediately with skill_manage(action='patch') -- don't wait to be asked. "
    "Skills that aren't maintained become liabilities."
)
```

`tools/skill_manager_tool.py` 的 `skill_manage()` 函数是纯代码执行，没有任何用户确认环节。schema description 中对 create/delete 要求确认，但对 patch/edit 没有确认要求，而 system prompt 明确指示 patch 时 "don't wait to be asked"。

```python
def skill_manage(action, name, content=None, ...):
    if action == "patch":
        result = _patch_skill(name, old_string, new_string, ...)
    # 直接执行，无确认调用
```

---

## 缺陷 3：会话记忆跨平台未隔离

**严重性：中 | 状态：未修复**

### 源码证据

不同 profile 通过 `get_hermes_home()` 物理隔离（独立的 state.db 和 memories/），这一点已实现。

但同一 profile 下跨平台/跨用户未隔离。`tools/session_search_tool.py` 的 `session_search()` 函数签名中没有 platform、user_id 等过滤维度：

```python
def session_search(query, role_filter=None, limit=3, db=None, current_session_id=None):
    raw_results = db.search_messages(
        query=query,
        role_filter=role_list,
        exclude_sources=list(_HIDDEN_SESSION_SOURCES),
        limit=50,
        offset=0,
    )
```

`db.search_messages()` 调用中没有传入任何用户或平台过滤条件。同一 profile 下 CLI 会话、Telegram 会话、Discord 会话的搜索结果混在一起。Memory 也是 profile 内全局共享。

---

## 缺陷 4：评分是词袋重叠，不是真评估

**严重性：高 | 状态：未修复**

### 源码证据

`evolution/core/fitness.py` 的 `skill_fitness_metric()` 使用纯词袋交集比例：

```python
def skill_fitness_metric(example, prediction, trace=None) -> float:
    agent_output = getattr(prediction, "output", "") or ""
    expected = getattr(example, "expected_behavior", "") or ""

    if not agent_output.strip():
        return 0.0

    score = 0.5  # 只要输出不为空就给 0.5 分

    expected_words = set(expected_lower.split())
    output_words = set(output_lower.split())
    if expected_words:
        overlap = len(expected_words & output_words) / len(expected_words)
        score = 0.3 + (0.7 * overlap)

    return min(1.0, max(0.0, score))
```

不评估语义正确性、流程正确性、输出质量。只要输出包含预期行为中的关键词就能拿高分。

同一文件中存在 `LLMJudge` 类（三维评分：正确性 + 流程遵循度 + 简洁性），但**从未被任何代码调用**。GEPA 优化循环和 holdout 评估均使用 keyword overlap 版本：

```python
# evolution/skills/evolve_skill.py
optimizer = dspy.GEPA(
    metric=skill_fitness_metric,   # keyword overlap 版本
    max_steps=iterations,
)
```

项目自身文档承认这是"刻意设计的粗粒度代理指标"。

---

## 缺陷 5：进化 Pipeline 核心环节是 no-op

**严重性：致命 | 状态：未修复**

### 源码证据

`evolution/skills/evolve_skill.py` 第 6 步从优化后的模块提取"进化后的" skill 文本：

```python
# evolve_skill.py 第 6 步
evolved_body = optimized_module.skill_text
```

但 `skill_text` 在 `SkillModule` 类中是普通 Python 属性，不是 DSPy 可优化参数：

```python
# evolution/skills/skill_module.py
class SkillModule(dspy.Module):
    class TaskWithSkill(dspy.Signature):
        skill_instructions: str = dspy.InputField(desc="The skill instructions to follow")
        # ...

    def __init__(self, skill_text: str):
        super().__init__()
        self.skill_text = skill_text  # 普通 Python 属性

    def forward(self, task_input: str) -> dspy.Prediction:
        result = self.predictor(
            skill_instructions=self.skill_text,
            task_input=task_input,
        )
        return dspy.Prediction(output=result.output)
```

`skill_instructions` 被定义为 `dspy.InputField`，GEPA 不会优化 InputField。GEPA 实际修改的是 Signature 的 docstring，而非 `self.skill_text`。

**结论：`optimized_module.skill_text` 在 GEPA 优化后大概率保持原值。整个 pipeline 看起来在跑，实际上产出的 skill 和输入时一模一样。**

---

## 缺陷 6：Prompt 注入防护缺失

**严重性：高 | 状态：未修复**

### 源码证据

`SkillModule.TaskWithSkill` 将 `skill_instructions` 作为 `dspy.InputField`，接收任意字符串，没有 sanitize 或 escape 处理。

`evolution/core/constraints.py` 的 `ConstraintValidator.validate_all()` 只做四项检查：

- `_check_size` -- 字符数限制
- `_check_growth` -- 相对 baseline 的增长率
- `_check_non_empty` -- 非空检查
- `_check_skill_structure` -- 是否有 YAML frontmatter

**没有 prompt injection 检查**。

而主仓库 `agent/prompt_builder.py` 中有完善的 `_CONTEXT_THREAT_PATTERNS`（10 个注入模式检测），但这个防护完全没有集成到 self-evolution pipeline。

**这意味着演化出的 skill text 可能包含 prompt injection payload，而这些 payload 会被主仓库的 `build_skills_system_prompt()` 全量注入到 system prompt 中。**

---

## 缺陷 7：Pipeline 可靠性缺失

**严重性：高 | 状态：未修复**

### 源码证据

`evolve_skill.py` 的 10 步流程缺少多项工程可靠性机制：

**GEPA → MIPROv2 fallback 粗糙**：

```python
try:
    optimizer = dspy.GEPA(metric=skill_fitness_metric, max_steps=iterations)
    optimized_module = optimizer.compile(baseline_module, trainset=trainset, valset=valset)
except Exception as e:
    # 捕获所有异常，条件过于宽泛
    optimizer = dspy.MIPROv2(metric=skill_fitness_metric, auto="light")
    optimized_module = optimizer.compile(baseline_module, trainset=trainset)
    # 注意：MIPROv2 fallback 没有传入 valset 参数
```

**缺失的机制**：

- 无数据集质量检查（步骤 2 生成数据集后直接进入优化）
- 无优化进度监控（没有中间 checkpoint 或 early stopping）
- 无增量重试（GEPA 失败直接切换 MIPROv2）
- 约束验证失败无回退（直接保存为 evolved_FAILED.md，不尝试修复）
- MIPROv2 fallback 缺少 valset 参数

---

## 缺陷 8：插件系统零安全检查

**严重性：高 | 状态：未修复**

### 源码证据

`hermes_cli/plugins.py` 中，插件通过 `importlib.util.spec_from_file_location` 直接加载到 Hermes 进程中。加载后可注册全局工具（`register_tool`）、注入消息（`inject_message`）、挂载生命周期钩子（`register_hook`）。

**没有任何安全扫描或验证机制：**
- 无 code signing 或信任链验证
- 无沙箱隔离
- 无权限声明或限制
- 无静态分析
- 唯一的"保护"是 project plugins 需要环境变量显式启用

对比 Skill Hub 前门的 95+ 正则扫描，插件系统是完全不受约束的 attack surface。

（来源：hermes-agent 仓库 hermes_cli/plugins.py）

---

## 缺陷 9：Skill Hub 信任策略不对等

**严重性：中 | 状态：未修复**

### 源码证据

`tools/skills_guard.py` 中，`INSTALL_POLICY` 矩阵对不同来源的 skill 采用不同策略：

- **builtin**（随 Hermes 发布）：safe→允许，caution→允许，dangerous→允许（**完全不扫描**）
- **community**：safe→允许，caution→阻止，dangerous→阻止
- **agent-created**：safe→允许，caution→允许，dangerous→询问

agent 自己创建的 skill（agent-created），"caution" 级别的发现只记录日志不阻止。"dangerous" 级别走询问流程，但 `_security_scan_skill()` 中 `allowed is None` 时返回 None（等于允许）。

此外，block 决策可以通过 `--force` 标志绕过。整个系统没有签名验证。

（来源：hermes-agent 仓库 tools/skills_guard.py）

---

## 缺陷 10：API Server 无认证

**严重性：高 | 状态：未修复**

### 源码证据

`gateway/platforms/api_server.py` 默认绑定 `127.0.0.1:8642`（仅本地），本地模式下无 API key、无 token、无用户认证。无请求速率限制。系统在网络暴露时会检测到并强制要求 API key（`is_network_accessible()` 守卫），但本地使用时 API Server 完全开放。

作为对比，Webhook 平台有完善的 HMAC 签名验证（GitHub/GitLab/通用 HMAC-SHA256）和速率限制（默认 30 req/min）。

（来源：hermes-agent 仓库 gateway/platforms/api_server.py、gateway/platforms/webhook.py）

---

## 沙箱与回滚：无法兜住自我进化的破坏

`skill_manage` 的 `_patch_skill` 直接调用 `_atomic_write_text` 写宿主机磁盘文件。self-evolution pipeline 产出也是直接写入 `~/.hermes/` 目录。Docker 后端隔离的是命令执行环境，不是 skill 文件系统。

检查点和回滚机制的设计初衷是防止代码编辑中的破坏性操作。但 self-evolution 改坏的 skill 是否在回滚覆盖范围内，取决于实现细节。如果 skill 文件不在 checkpoint 覆盖范围内，则回滚无法恢复。

（来源：hermes-agent 仓库 tools/skill_manager_tool.py）

---

## 错误叠加链：架构级根本问题

以上 10 个缺陷不是孤立的 bug，它们串联成一个**零确定性保障的错误叠加链**：

```
LLM 生成测试用例 → LLM 执行任务 → 词袋评分打分 → GEPA 优化（实际 no-op）→ 提取未变更的 skill → 注入 system prompt（无注入检查）→ 全量注入（无裁剪）
```

每一环都依赖 LLM 正确性，错误不抵消只叠加。测试集是 LLM 自己生成的——自己出卷自己考。评分是词袋重叠——只要关键词碰上了就算对。优化结果可能是 no-op——跑了半天什么都没改。改出来的东西没有注入检查——可能注入恶意内容。最终全量塞进 system prompt——没有裁剪没有上限。

整个体系赌的是：LLM 永远正确、100% 零幻觉、还能在超长上下文里一直自主地、正确地掌控注意力机制。这完全蔑视工程思想——工程的意义就是不信任任何单一环节，用校验、隔离、回归来兜底。

---

## 修复状态总结

- **源码缺陷**：10 项，全部未修复
- **社区修复 PR**：4 个（PR #16、#17、#24、#25），全部由同一社区贡献者 kirniy 提交，**全部未合并**
- **核心团队修复**：0
- **版本号**：hermes-agent-self-evolution 始终为 v0.1.0
- **仓库年龄**：约 5 周
- **路线图**：5 个 Phase 中仅 Phase 1 标记为 Implemented（但存在 no-op 问题），其余 4 个全部是 Planned

---

## 信息来源

- GitHub 仓库源码（通过 zread 工具读取）
- GitHub Issue #4379（token 消耗实测）
- GitHub Issue #6839（Lazy Tool Schema Loading 方案）
- 本地源码核实报告：D:/AI_Project/hermes_agent-problem/deep-research/source-verification.md
