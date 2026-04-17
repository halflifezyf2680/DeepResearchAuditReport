# Hermes Agent 源码核实报告

> 核实方法：通过 read 工具直接读取 GitHub 仓库源码，逐项提取具体代码证据。
> 仓库：NousResearch/hermes-agent (main), NousResearch/hermes-agent-self-evolution (main)
> 核实日期：2026-04-17

---

## 缺陷 1: Skill 索引全量注入

**状态：已核实（部分成立）**

### 源码证据

**1.1 索引构建入口：`build_skills_system_prompt()`**

文件：`agent/prompt_builder.py`
函数：`build_skills_system_prompt(available_tools, available_toolsets)`

该函数遍历所有 skill 目录，为每个 SKILL.md 构建索引条目。关键流程：

```python
# 文件: agent/prompt_builder.py, build_skills_system_prompt() 函数
# 遍历所有 skill 文件（无检索/裁剪）
for skill_file in iter_skill_index_files(skills_dir, "SKILL.md"):
    is_compatible, frontmatter, desc = _parse_skill_file(skill_file)
    # ...
    skills_by_category.setdefault(entry["category"], []).append(
        (skill_name, entry["description"])
    )
```

最终拼接为完整的 system prompt 块：

```python
# 文件: agent/prompt_builder.py, build_skills_system_prompt() 末尾
result = (
    "## Skills (mandatory)\n"
    "Before replying, scan the skills below. If one clearly matches your task, "
    "load it with skill_view(name) and follow its instructions. "
    # ...
    "<available_skills>\n"
    + "\n".join(index_lines) + "\n"
    "</available_skills>\n"
)
```

**1.2 存在的过滤/裁剪机制**

文件：`agent/prompt_builder.py`

存在以下过滤层，但均为**静态/粗粒度过滤**，并非基于语义检索：

- **平台过滤** (`_parse_skill_file`): 根据 frontmatter 中的 `platforms` 字段排除不兼容平台的 skill
- **禁用列表** (`get_disabled_skill_names`): 从 config.yaml 读取 `skills.disabled` 列表
- **条件激活** (`_skill_should_show`): 根据 `requires_tools` / `requires_toolsets` / `fallback_for_toolsets` 等条件过滤
- **去重**：同名 skill 只保留第一个

```python
# 文件: agent/prompt_builder.py, _skill_should_show()
def _skill_should_show(
    conditions: dict,
    available_tools: "set[str] | None",
    available_toolsets: "set[str] | None",
) -> bool:
    # fallback_for: hide when the primary tool/toolset IS available
    for ts in conditions.get("fallback_for_toolsets", []):
        if ts in ats:
            return False
    # requires: hide when a required tool/toolset is NOT available
    for ts in conditions.get("requires_toolsets", []):
        if ts not in ats:
            return False
    return True
```

**1.3 不存在的机制**

- 无语义相似度检索（无 embedding/RAG）
- 无 LLM 判断 relevance 的环节
- 无基于用户当前 task 的动态裁剪
- 索引只包含 name + description，不包含 skill 全文，但 description 被截断为 60 字符（见 `skill_utils.py` 的 `extract_skill_description`）

**1.4 缓存机制**

文件：`agent/prompt_builder.py`

存在两层缓存（进程内 LRU + 磁盘 snapshot），但缓存的是全量索引结果，不是检索结果：

```python
_SKILLS_PROMPT_CACHE_MAX = 8
_SKILLS_PROMPT_CACHE: OrderedDict[tuple, str] = OrderedDict()
```

### 结论

- 全量注入到 system prompt 的方式**已核实**：所有通过平台/禁用/条件过滤后的 skill 索引被完整拼入 `<available_skills>` 块
- **不存在**动态语义检索或按 relevance 裁剪的机制
- 过滤机制是静态的（平台兼容、工具依赖、禁用列表），不做按查询意图的检索
- 缓存是全量结果的缓存，不改变"全量注入"的本质

---

## 缺陷 2: 自修改无确认

**状态：已核实**

### 源码证据

**2.1 System prompt 中的"立即修补"指令**

文件：`agent/prompt_builder.py`

`SKILLS_GUIDANCE` 常量直接注入 system prompt：

```python
# 文件: agent/prompt_builder.py
SKILLS_GUIDANCE = (
    "After completing a complex task (5+ tool calls), fixing a tricky error, "
    "or discovering a non-trivial workflow, save the approach as a "
    "skill with skill_manage so you can reuse it next time.\n"
    "When using a skill and finding it outdated, incomplete, or wrong, "
    "patch it immediately with skill_manage(action='patch') -- don't wait to be asked. "
    "Skills that aren't maintained become liabilities."
)
```

关键短语：`patch it immediately with skill_manage(action='patch') -- don't wait to be asked`

**2.2 skill_manage 工具的 schema 描述中的确认指令**

文件：`tools/skill_manager_tool.py`

schema 的 `description` 字段中包含：

```python
# 文件: tools/skill_manager_tool.py, SKILL_MANAGE_SCHEMA
"description": (
    # ...
    "After difficult/iterative tasks, offer to save as a skill. "
    "Skip for simple one-offs. Confirm with user before creating/deleting.\n\n"
    # ...
)
```

注意：schema 中对 `create` 和 `delete` 要求确认，但 `patch` 和 `edit` 没有确认要求。`SKILLS_GUIDANCE` 中的指令是 "patch it immediately...don't wait to be asked"。

**2.3 skill_manage 的实现代码**

文件：`tools/skill_manager_tool.py`

`skill_manage()` 函数是纯代码执行，没有任何用户确认环节：

```python
# 文件: tools/skill_manager_tool.py, skill_manage()
def skill_manage(
    action: str,
    name: str,
    content: str = None,
    # ...
) -> str:
    """Manage user-created skills. Dispatches to the appropriate action handler."""
    if action == "create":
        # ...
        result = _create_skill(name, content, category)
    elif action == "patch":
        # ...
        result = _patch_skill(name, old_string, new_string, file_path, replace_all)
    elif action == "edit":
        result = _edit_skill(name, content)
    # ... 直接执行，无确认调用
    return json.dumps(result, ensure_ascii=False)
```

`_patch_skill()` 直接修改磁盘文件：

```python
# 文件: tools/skill_manager_tool.py, _patch_skill()
def _patch_skill(name, old_string, new_string, file_path=None, replace_all=False):
    # ...
    _atomic_write_text(target, new_content)  # 直接写入磁盘
    # ...
```

**2.4 安全扫描**

文件：`tools/skill_manager_tool.py`

存在安全扫描机制（`_security_scan_skill`），但这是内容安全检查（注入/恶意代码），不是用户确认机制：

```python
# 文件: tools/skill_manager_tool.py, _patch_skill() 末尾
scan_error = _security_scan_skill(skill_dir)
if scan_error:
    _atomic_write_text(target, original_content)  # rollback
    return {"success": False, "error": scan_error}
```

### 结论

- system prompt 明确指示 agent "patch it immediately...don't wait to be asked" -- **已核实**
- `skill_manage` 工具的实现代码中没有任何用户确认环节 -- **已核实**
- 对 create/delete 有语义层面的"confirm with user"指令（在 schema description 中），但对 patch/edit 没有
- 存在安全扫描作为后防线，但不等同于用户确认

---

## 缺陷 3: 会话/记忆未隔离

**状态：已核实（部分成立 -- profile 间已隔离，profile 内跨用户/跨平台未隔离）**

### 源码证据

**3.1 session_search 的数据库路径**

文件：`hermes_state.py`

```python
# 文件: hermes_state.py
DEFAULT_DB_PATH = get_hermes_home() / "state.db"
```

`get_hermes_home()` 在不同 profile 下返回不同路径，因此不同 profile 的 session 数据物理隔离。

**3.2 session_search 函数签名**

文件：`tools/session_search_tool.py`

```python
# 文件: tools/session_search_tool.py, session_search()
def session_search(
    query: str,
    role_filter: str = None,
    limit: int = 3,
    db=None,
    current_session_id: str = None,
) -> str:
```

参数列表中**没有** `profile`、`user_id`、`platform` 或任何隔离维度。`db` 参数由框架注入，指向当前 profile 的 state.db。

**3.3 search_messages 调用**

文件：`tools/session_search_tool.py`

```python
# 文件: tools/session_search_tool.py, session_search() 内
raw_results = db.search_messages(
    query=query,
    role_filter=role_list,
    exclude_sources=list(_HIDDEN_SESSION_SOURCES),
    limit=50,
    offset=0,
)
```

`db.search_messages()` 的调用中没有传入任何用户/平台过滤条件。`exclude_sources` 仅排除 `"tool"` 来源的会话。

**3.4 _list_recent_sessions 调用**

```python
# 文件: tools/session_search_tool.py, _list_recent_sessions()
sessions = db.list_sessions_rich(limit=limit + 5, exclude_sources=list(_HIDDEN_SESSION_SOURCES))
```

同样没有用户/平台过滤。

**3.5 Memory 的 profile 隔离**

文件：`tools/memory_tool.py`

```python
# 文件: tools/memory_tool.py
def get_memory_dir() -> Path:
    """Return the profile-scoped memories directory."""
    return get_hermes_home() / "memories"
```

Memory 文件位于 `HERMES_HOME/memories/`，因此 profile 间是隔离的。但同一 profile 下的所有会话共享同一份 MEMORY.md 和 USER.md。

### 结论

- **Profile 间隔离**：已通过 `get_hermes_home()` 实现，每个 profile 有独立的 `state.db` 和 `memories/`
- **Profile 内隔离**：不存在。同一 profile 下，CLI 会话、Telegram 会话、Discord 会话等的 session_search 结果混在一起；memory 也是全局共享的
- `session_search` 函数参数中没有 `platform`、`user_id` 等过滤维度
- `AGENTS.md` 文档中 Profiles 章节确认了 profile 是隔离的基本单元，未提及更细粒度的隔离

---

## 缺陷 4: Keyword Overlap Fitness Metric

**状态：已核实**

### 源码证据

**4.1 `skill_fitness_metric` 完整代码**

文件：`evolution/core/fitness.py`

```python
# 文件: evolution/core/fitness.py
def skill_fitness_metric(example: dspy.Example, prediction: dspy.Prediction, trace=None) -> float:
    """DSPy-compatible metric function for skill optimization.

    This is what gets passed to dspy.GEPA(metric=...).
    Returns a float 0-1 score.
    """
    # The prediction should have an 'output' field with the agent's response
    agent_output = getattr(prediction, "output", "") or ""
    expected = getattr(example, "expected_behavior", "") or ""
    task = getattr(example, "task_input", "") or ""

    if not agent_output.strip():
        return 0.0

    # Quick heuristic scoring (for speed during optimization)
    # Full LLM-as-judge scoring is expensive -- use it selectively
    score = 0.5  # Base score for non-empty output

    # Check if key phrases from expected behavior appear
    expected_lower = expected.lower()
    output_lower = agent_output.lower()

    # Simple keyword overlap as a fast proxy
    expected_words = set(expected_lower.split())
    output_words = set(output_lower.split())
    if expected_words:
        overlap = len(expected_words & output_words) / len(expected_words)
        score = 0.3 + (0.7 * overlap)

    return min(1.0, max(0.0, score))
```

该函数的实际逻辑：
1. 将 `expected_behavior` 和 `agent_output` 分词为 set
2. 计算交集比例（Jaccard-like，但只除以 expected 的大小）
3. 映射为 0.3-1.0 的分数

**4.2 LLMJudge 类的定义**

文件：`evolution/core/fitness.py`

```python
# 文件: evolution/core/fitness.py
class LLMJudge:
    """LLM-as-judge scorer with rubric-based evaluation."""

    class JudgeSignature(dspy.Signature):
        """Evaluate an agent's response against an expected behavior rubric.
        Score the response on three dimensions (0.0 to 1.0 each):
        1. correctness
        2. procedure_following
        3. conciseness
        """
        task_input: str = dspy.InputField(desc="The task the agent was given")
        expected_behavior: str = dspy.InputField(desc="Rubric describing what a good response looks like")
        agent_output: str = dspy.InputField(desc="The agent's actual response")
        skill_text: str = dspy.InputField(desc="The skill/instructions the agent was following")
        correctness: float = dspy.OutputField(desc="Score 0.0-1.0")
        procedure_following: float = dspy.OutputField(desc="Score 0.0-1.0")
        conciseness: float = dspy.OutputField(desc="Score 0.0-1.0")
        feedback: str = dspy.OutputField(desc="Specific, actionable feedback")
```

**4.3 LLMJudge 是否被接入 GEPA 循环**

文件：`evolution/skills/evolve_skill.py`

```python
# 文件: evolution/skills/evolve_skill.py, evolve() 函数
optimizer = dspy.GEPA(
    metric=skill_fitness_metric,   # <-- 传入的是 keyword overlap 版本
    max_steps=iterations,
)

optimized_module = optimizer.compile(
    baseline_module,
    trainset=trainset,
    valset=valset,
)
```

GEPA 优化器的 `metric` 参数接收的是 `skill_fitness_metric`（keyword overlap 版本），而非 `LLMJudge.score()`。

在 `evolve_skill.py` 的 import 中，`LLMJudge` 被导入但仅用于 holdout 评估阶段（第 8 步），不参与 GEPA 优化循环：

```python
# 文件: evolution/skills/evolve_skill.py
from evolution.core.fitness import skill_fitness_metric, LLMJudge, FitnessScore
```

但在第 8 步（holdout 评估）中，代码仍然使用 `skill_fitness_metric` 而非 `LLMJudge`：

```python
# 文件: evolution/skills/evolve_skill.py, 第 8 步
for ex in holdout_examples:
    with dspy.context(lm=lm):
        baseline_pred = baseline_module(task_input=ex.task_input)
        baseline_score = skill_fitness_metric(ex, baseline_pred)  # keyword overlap
        evolved_pred = optimized_module(task_input=ex.task_input)
        evolved_score = skill_fitness_metric(ex, evolved_pred)    # keyword overlap
```

### 结论

- `skill_fitness_metric` 确实使用简单 keyword overlap -- **已核实**
- `LLMJudge` 类已定义，提供了更精细的多维评分（correctness + procedure_following + conciseness）-- **已核实**
- `LLMJudge` 未被接入 GEPA 优化循环，GEPA 使用的是 keyword overlap 版本的 metric -- **已核实**
- `LLMJudge` 在 holdout 评估阶段也未使用，holdout 评估同样使用 keyword overlap
- 代码注释中说 "Full LLM-as-judge scoring is expensive -- use it selectively"，但没有找到任何调用 `LLMJudge.score()` 的代码路径

---

## 缺陷 5: optimized_module.skill_text 疑似 no-op

**状态：已核实**

### 源码证据

**5.1 evolve_skill.py 第 6 步提取 skill_text**

文件：`evolution/skills/evolve_skill.py`

```python
# 文件: evolution/skills/evolve_skill.py, evolve() 函数第 6 步
# ── 6. Extract evolved skill text ───────────────────────────────────
# The optimized module's instructions contain the evolved skill text
evolved_body = optimized_module.skill_text
evolved_full = reassemble_skill(skill["frontmatter"], evolved_body)
```

代码假设 `optimized_module` 上存在 `skill_text` 属性，且该属性包含 GEPA 优化后的 skill 指令文本。

**5.2 SkillModule 类中 skill_text 的定义**

文件：`evolution/skills/skill_module.py`

```python
# 文件: evolution/skills/skill_module.py
class SkillModule(dspy.Module):
    """A DSPy module that wraps a skill file for optimization.

    The skill text (body) is the parameter that GEPA optimizes.
    On each forward pass, the module:
    1. Uses the skill text as instructions
    2. Processes the task input
    3. Returns the agent's response
    """

    class TaskWithSkill(dspy.Signature):
        """Complete a task following the provided skill instructions."""
        skill_instructions: str = dspy.InputField(desc="The skill instructions to follow")
        task_input: str = dspy.InputField(desc="The task to complete")
        output: str = dspy.OutputField(desc="Your response following the skill instructions")

    def __init__(self, skill_text: str):
        super().__init__()
        self.skill_text = skill_text
        self.predictor = dspy.ChainOfThought(self.TaskWithSkill)

    def forward(self, task_input: str) -> dspy.Prediction:
        result = self.predictor(
            skill_instructions=self.skill_text,
            task_input=task_input,
        )
        return dspy.Prediction(output=result.output)
```

**关键分析**：

1. `self.skill_text` 是一个普通的 Python 字符串属性，不是 DSPy 可优化参数
2. `skill_instructions` 在 `TaskWithSkill` Signature 中被定义为 `dspy.InputField`（输入字段），不是 `dspy.OutputField` 也不是没有类型标注的可优化字段
3. DSPy 的 GEPA 优化器优化的是 Signature 中的指令（即 Signature class 的 docstring）或未标注的参数，而不是 `InputField`
4. `skill_text` 作为 `self.skill_text` 在 `forward()` 中被传递给 `skill_instructions=InputField`，GEPA 不会修改这个值

**5.3 DSPy GEPA 的优化目标**

DSPy GEPA 优化模块时，修改的是 Signature 的 docstring（指令部分）和 `OutputField` 之外的字段标注。对于 `SkillModule`：

- `TaskWithSkill` 的 docstring 是 GEPA 可以修改的指令
- `skill_instructions: str = dspy.InputField()` 是输入字段，GEPA 不会优化它
- `self.skill_text` 是模块的普通属性，不在 DSPy 的参数追踪范围内

因此 `optimized_module.skill_text` 在 GEPA 优化后仍然是原始的 `skill_text` 值，未被修改。

### 结论

- `optimized_module.skill_text` 在 GEPA 优化后大概率保持原值 -- **已核实**
- `skill_text` 是普通 Python 属性，不是 DSPy 可优化参数 -- **已核实**
- `skill_instructions` 被定义为 `dspy.InputField`，GEPA 不优化 InputField -- **已核实**
- GEPA 实际修改的是 `TaskWithSkill` Signature 的 docstring，而非 `skill_text`
- 因此第 6 步提取的 `evolved_body` 实际上是原始 skill body，不是优化后的版本

---

## 缺陷 6: Prompt 注入漏洞

**状态：已核实**

### 源码证据

**6.1 TaskWithSkill 的 skill_instructions 字段类型**

文件：`evolution/skills/skill_module.py`

```python
# 文件: evolution/skills/skill_module.py, SkillModule.TaskWithSkill
class TaskWithSkill(dspy.Signature):
    """Complete a task following the provided skill instructions."""
    skill_instructions: str = dspy.InputField(desc="The skill instructions to follow")
    task_input: str = dspy.InputField(desc="The task to complete")
    output: str = dspy.OutputField(desc="Your response following the skill instructions")
```

`skill_instructions` 是 `dspy.InputField`，接收任意字符串。skill 文件（SKILL.md body）的内容被原样传入此字段，没有任何 sanitize 或 escape 处理。

**6.2 ConstraintValidator 的检查逻辑**

文件：`evolution/core/constraints.py`

```python
# 文件: evolution/core/constraints.py, ConstraintValidator.validate_all()
def validate_all(self, artifact_text, artifact_type, baseline_text=None):
    results = []
    results.append(self._check_size(artifact_text, artifact_type))
    if baseline_text:
        results.append(self._check_growth(artifact_text, baseline_text, artifact_type))
    results.append(self._check_non_empty(artifact_text))
    if artifact_type == "skill":
        results.append(self._check_skill_structure(artifact_text))
    return results
```

四个检查：

1. `_check_size` -- 字符数限制
2. `_check_growth` -- 相对 baseline 的增长率
3. `_check_non_empty` -- 非空检查
4. `_check_skill_structure` -- 检查是否有 YAML frontmatter（name + description）

```python
# 文件: evolution/core/constraints.py, _check_skill_structure()
def _check_skill_structure(self, text: str) -> ConstraintResult:
    has_frontmatter = text.strip().startswith("---")
    has_name = "name:" in text[:500] if has_frontmatter else False
    has_description = "description:" in text[:500] if has_frontmatter else False
    # ...
```

**缺失的检查**：

- 无 prompt injection 检查（对比主仓库 `prompt_builder.py` 中的 `_CONTEXT_THREAT_PATTERNS` 有 10 个注入模式检测）
- 无内容安全性检查（无恶意指令检测、无 `<system>` 标签检测、无 escape 检查）
- 无 YAML frontmatter 注入检测（攻击者可在 frontmatter 中注入 `metadata` 字段覆盖配置）

### 结论

- `skill_instructions` 是 `InputField`，接收任意字符串 -- **已核实**
- ConstraintValidator 不检查 prompt injection -- **已核实**
- 主仓库（hermes-agent）的 `prompt_builder.py` 有完善的 injection 检查（`_CONTEXT_THREAT_PATTERNS`），但 self-evolution 子仓库完全没有类似机制 -- **已核实**
- 演化出的 skill text 可能包含 prompt injection payload，而这些 payload 会被注入到 hermes-agent 的 system prompt 中

---

## 缺陷 7: Pipeline 可靠性

**状态：已核实**

### 源码证据

**7.1 evolve_skill.py 完整流程**

文件：`evolution/skills/evolve_skill.py`

10 步流程：
1. 找到并加载 skill
2. 构建/加载评估数据集
3. 对 baseline 做约束验证
4. 配置 DSPy + GEPA 优化器
5. 运行 GEPA 优化
6. 提取演化后的 skill text
7. 验证演化后的 skill
8. 在 holdout 集上评估
9. 报告结果
10. 保存输出

**7.2 GEPA -> MIPROv2 fallback**

```python
# 文件: evolution/skills/evolve_skill.py, evolve() 函数第 5 步
try:
    optimizer = dspy.GEPA(
        metric=skill_fitness_metric,
        max_steps=iterations,
    )

    optimized_module = optimizer.compile(
        baseline_module,
        trainset=trainset,
        valset=valset,
    )
except Exception as e:
    # Fall back to MIPROv2 if GEPA isn't available in this DSPy version
    console.print(f"[yellow]GEPA not available ({e}), falling back to MIPROv2[/yellow]")
    optimizer = dspy.MIPROv2(
        metric=skill_fitness_metric,
        auto="light",
    )
    optimized_module = optimizer.compile(
        baseline_module,
        trainset=trainset,
    )
```

**分析**：

- fallback 捕获 `Exception`（所有异常），条件是 "GEPA isn't available in this DSPy version"
- fallback 使用 `dspy.MIPROv2(metric=skill_fitness_metric, auto="light")`
- 注意 `auto="light"` 模式，这意味着 MIPROv2 使用较少的 prompt 优化尝试

**7.3 缺失的可靠性机制**

1. **无数据集质量检查**：步骤 2 生成/加载数据集后，直接进入优化，没有数据集质量验证（如：是否有 degenerate examples、label 分布是否合理）
2. **无优化进度监控**：步骤 5 是一次性调用 `optimizer.compile()`，没有中间 checkpoint 或 early stopping
3. **无增量重试**：GEPA 失败后直接切换到 MIPROv2，没有尝试降低参数或换用不同 metric
4. **步骤 6 的 no-op 问题**（见缺陷 5）：提取的 `optimized_module.skill_text` 可能未改变
5. **约束验证失败无回退**：步骤 7 约束验证失败后直接放弃（保存为 `evolved_FAILED.md`），不尝试修复或回退到上一个合法版本
6. **MIPROv2 fallback 无 valset**：GEPA 传入 `valset=valset`，但 MIPROv2 fallback 没有传入 `valset`

### 结论

- GEPA -> MIPROv2 fallback 机制存在 -- **已核实**
- fallback 捕获所有 Exception，条件较宽泛 -- **已核实**
- MIPROv2 fallback 缺少 `valset` 参数 -- **已核实**
- 缺失多项可靠性机制（checkpoint、early stopping、增量重试、数据集质量检查）-- **已核实**
- 结合缺陷 5 的 no-op 问题，整个 pipeline 可能在"成功"路径上产出一个未实际优化的 skill

---

## 缺陷 8: 插件系统零安全检查

**状态：已核实**

### 源码证据

**8.1 插件加载：importlib 直接执行，无安全检查**

文件：`hermes_cli/plugins.py`

插件通过 `_load_directory_module()` 和 `_load_entrypoint_module()` 加载，均直接使用 `importlib` 执行代码，无任何签名验证、沙箱隔离或权限限制。

目录插件的加载路径（`_load_directory_module`）：

```python
# 文件: hermes_cli/plugins.py, PluginManager._load_directory_module()
spec = importlib.util.spec_from_file_location(
    module_name,
    init_file,
    submodule_search_locations=[str(plugin_dir)],
)
# ...
module = importlib.util.module_from_spec(spec)
module.__package__ = module_name
module.__path__ = [str(plugin_dir)]
sys.modules[module_name] = module
spec.loader.exec_module(module)
return module
```

Pip 入口点插件的加载路径（`_load_entrypoint_module`）：

```python
# 文件: hermes_cli/plugins.py, PluginManager._load_entrypoint_module()
for ep in group_eps:
    if ep.name == manifest.name:
        return ep.load()  # 直接加载，无安全检查
```

**8.2 插件可执行的能力范围**

文件：`hermes_cli/plugins.py`, `PluginContext` 类

插件通过 `PluginContext` 可执行以下操作：

- **注册工具** (`register_tool`): 直接注入全局工具注册表，与内置工具同等权限
- **注入消息** (`inject_message`): 可向活跃会话注入任意角色消息（`role="user"` 或其他）
- **注册 CLI 命令** (`register_cli_command`): 注册新的终端子命令
- **注册斜杠命令** (`register_command`): 注册会话内斜杠命令
- **派发工具调用** (`dispatch_tool`): 可调用任意已注册工具
- **替换上下文引擎** (`register_context_engine`): 可替换内置的 ContextCompressor
- **注册生命周期钩子** (`register_hook`): 在 10 个生命周期点注册回调
- **注册技能** (`register_skill`): 注册只读 skill

```python
# 文件: hermes_cli/plugins.py, PluginContext.inject_message()
def inject_message(self, content: str, role: str = "user") -> bool:
    """Inject a message into the active conversation."""
    cli = self._manager._cli_ref
    msg = content if role == "user" else f"[{role}] {content}"
    if getattr(cli, "_agent_running", False):
        cli._interrupt_queue.put(msg)  # Agent 运行中可中断注入
    else:
        cli._pending_input.put(msg)   # Agent 空闲可排队注入
    return True
```

```python
# 文件: hermes_cli/plugins.py, PluginContext.register_tool()
def register_tool(self, name, toolset, schema, handler, ...):
    from tools.registry import registry
    registry.register(...)  # 直接写入全局注册表
    self._manager._plugin_tool_names.add(name)
```

**8.3 有效的生命周期钩子**

```python
# 文件: hermes_cli/plugins.py
VALID_HOOKS: Set[str] = {
    "pre_tool_call",
    "post_tool_call",
    "pre_llm_call",
    "post_llm_call",
    "pre_api_request",
    "post_api_request",
    "on_session_start",
    "on_session_end",
    "on_session_finalize",
    "on_session_reset",
}
```

**8.4 项目插件通过环境变量启用**

```python
# 文件: hermes_cli/plugins.py, PluginManager.discover_and_load()
# 2. Project plugins (./.hermes/plugins/)
if _env_enabled("HERMES_ENABLE_PROJECT_PLUGINS"):
    project_dir = Path.cwd() / ".hermes" / "plugins"
    manifests.extend(self._scan_directory(project_dir, source="project"))
```

项目插件默认关闭，需显式设置环境变量 `HERMES_ENABLE_PROJECT_PLUGINS`。用户插件（`~/.hermes/plugins/`）和 pip 入口点插件始终启用。

**8.5 唯一存在的安全机制**

唯一的限制是斜杠命令名的冲突检查和内置命令名称验证：

```python
# 文件: hermes_cli/plugins.py, PluginContext.register_command()
from hermes_cli.commands import resolve_command
if resolve_command(clean) is not None:
    logger.warning("Plugin '%s' tried to register command '/%s' which conflicts "
                   "with a built-in command. Skipping.", ...)
    return
```

以及插件可在 `config.yaml` 中被禁用：

```python
# 文件: hermes_cli/plugins.py
def _get_disabled_plugins() -> set:
    from hermes_cli.config import load_config
    config = load_config()
    disabled = config.get("plugins", {}).get("disabled", [])
    return set(disabled)
```

**8.6 不存在的安全机制**

- 无代码签名或完整性验证
- 无沙箱或权限隔离
- 无插件能力声明/审批流程
- 无插件间隔离
- 无 `register()` 函数的参数校验
- hook 回调中的异常被 catch 并忽略（不影响核心循环），但这是可用性保护，不是安全边界

### 结论

- 插件通过 `importlib` 直接执行代码 -- **已核实**
- 无签名验证、无沙箱、无权限隔离 -- **已核实**
- 插件可注册工具、注入消息、替换上下文引擎、注册生命周期钩子 -- **已核实**
- 项目插件需环境变量显式启用，用户/pip 插件始终启用 -- **已核实**
- 唯一限制：斜杠命令名冲突检查 + 配置文件禁用列表 -- **已核实**

---

## 缺陷 9: Skill Hub 信任策略不对等

**状态：已核实**

### 源码证据

**9.1 信任矩阵定义**

文件：`tools/skills_guard.py`

```python
# 文件: tools/skills_guard.py
INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),
    "trusted":       ("allow",  "allow",   "block"),
    "community":     ("allow",  "block",   "block"),
    "agent-created": ("allow",  "allow",   "ask"),
}
```

**9.2 信任来源映射**

```python
# 文件: tools/skills_guard.py
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

def _resolve_trust_level(source: str) -> str:
    if normalized_source == "agent-created":
        return "agent-created"
    if normalized_source.startswith("official/") or normalized_source == "official":
        return "builtin"
    for trusted in TRUSTED_REPOS:
        if normalized_source.startswith(trusted) or normalized_source == trusted:
            return "trusted"
    return "community"
```

信任级别映射逻辑：
- `"agent-created"` -> `agent-created` 信任级别
- `"official/"` 或 `"official"` -> `builtin`
- `"openai/skills"` 或 `"anthropics/skills"` -> `trusted`
- 其他所有来源 -> `community`

**9.3 --force 标志的绕过逻辑**

```python
# 文件: tools/skills_guard.py
def should_allow_install(result: ScanResult, force: bool = False) -> Tuple[bool, str]:
    policy = INSTALL_POLICY.get(result.trust_level, INSTALL_POLICY["community"])
    vi = VERDICT_INDEX.get(result.verdict, 2)
    decision = policy[vi]

    if decision == "allow":
        return True, f"Allowed ({result.trust_level} source, {result.verdict} verdict)"

    if force:
        return True, (
            f"Force-installed despite {result.verdict} verdict "
            f"({len(result.findings)} findings)"
        )

    if decision == "ask":
        return None, (  # None = 需要用户确认
            f"Requires confirmation ({result.trust_level} source + {result.verdict} verdict, "
            f"{len(result.findings)} findings)"
        )

    return False, (
        f"Blocked ({result.trust_level} source + {result.verdict} verdict, "
        f"{len(result.findings)} findings). Use --force to override."
    )
```

`force=True` 的行为：
- 当策略为 `"block"` 时，`force` 直接返回 `(True, ...)` 跳过阻止
- 当策略为 `"ask"` 时，`force` **不会**跳过（因为 `decision == "ask"` 的判断在 `if force:` 之后）
- 但注意：如果 `decision` 是 `"ask"`，代码返回 `None`（非 False），而 `force=True` 时由于 `if force:` 在 `if decision == "ask":` 之前，`force` 实际上**会**绕过 `ask` 策略

**9.4 "ask" 返回 None 的问题**

```python
# 文件: tools/skills_guard.py
def should_allow_install(result: ScanResult, force: bool = False) -> Tuple[bool, str]:
    # ...
    if decision == "ask":
        return None, (...)  # 返回 None 而非 (False, ...)
```

返回值类型注解为 `Tuple[bool, str]`，但 `"ask"` 分支返回 `(None, str)`。调用方需要处理三态：`True`/`False`/`None`。

**9.5 agent-created 与 community 的策略对比**

| 场景 | agent-created (dangerous 扫描) | community (dangerous 扫描) |
|------|------|------|
| 无 force | `None` (ask 确认) | `False` (blocked) |
| 有 force | `True` (allow) | `True` (allow) |

agent-created 的 `dangerous` 扫描结果只要求确认（ask），community 的 `dangerous` 结果直接阻止。这意味着 agent 自创的恶意 skill 比外部 community skill 更容易通过（只需一次点击确认 vs 完全阻止）。

而 `caution` 级别下：
- agent-created: `"allow"` -- 直接放行
- community: `"block"` -- 直接阻止
- trusted: `"allow"` -- 直接放行

### 结论

- 信任矩阵定义确认 builtin 全放、community 全拦、agent-created 有 ask 确认 -- **已核实**
- `--force` 绕过 block 和 ask 策略 -- **已核实**
- `should_allow_install` 对 ask 返回 None，与类型注解 `Tuple[bool, str]` 不一致 -- **已核实**
- agent-created 在 caution 级别直接 allow，而 community 同级别 block -- **已核实**

---

## 缺陷 10: API Server 无认证

**状态：已核实（部分成立）**

### 源码证据

**10.1 默认绑定地址**

文件：`gateway/platforms/api_server.py`

```python
# 文件: gateway/platforms/api_server.py
DEFAULT_HOST = "127.0.0.1"
DEFAULT_PORT = 8642
```

默认绑定 `127.0.0.1`（仅本地），不是 `0.0.0.0`。可通过配置或环境变量覆盖：

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter.__init__()
self._host: str = extra.get("host", os.getenv("API_SERVER_HOST", DEFAULT_HOST))
self._port: int = int(extra.get("port", os.getenv("API_SERVER_PORT", str(DEFAULT_PORT))))
self._api_key: str = extra.get("key", os.getenv("API_SERVER_KEY", ""))
```

**10.2 认证机制**

文件：`gateway/platforms/api_server.py`

认证是**可选的**，通过 `API_SERVER_KEY` 环境变量或 `platforms.api_server.key` 配置设置。当未配置 key 时，所有请求被放行：

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter._check_auth()
def _check_auth(self, request: "web.Request") -> Optional["web.Response"]:
    """
    Validate Bearer token from Authorization header.

    Returns None if auth is OK, or a 401 web.Response on failure.
    If no API key is configured, all requests are allowed (only when API
    server is local).
    """
    if not self._api_key:
        return None  # No key configured -- allow all (local-only use)

    auth_header = request.headers.get("Authorization", "")
    if auth_header.startswith("Bearer "):
        token = auth_header[7:].strip()
        if hmac.compare_digest(token, self._api_key):
            return None  # Auth OK

    return web.json_response(
        {"error": {"message": "Invalid API key", "type": "invalid_request_error",
                    "code": "invalid_api_key"}},
        status=401,
    )
```

**10.3 网络暴露保护**

文件：`gateway/platforms/api_server.py`

当绑定地址为非本地地址时，**强制要求**配置 API key：

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter.connect()
# Refuse to start network-accessible without authentication
if is_network_accessible(self._host) and not self._api_key:
    logger.error(
        "[%s] Refusing to start: binding to %s requires API_SERVER_KEY. "
        "Set API_SERVER_KEY or use the default 127.0.0.1.",
        self.name, self._host,
    )
    return False
```

同时检查 key 是否为占位符值：

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter.connect()
if is_network_accessible(self._host) and self._api_key:
    from hermes_cli.auth import has_usable_secret
    if not has_usable_secret(self._api_key, min_length=8):
        logger.error(
            "[%s] Refusing to start: API_SERVER_KEY is set to a "
            "placeholder value. Generate a real secret ...",
            self.name, self._host,
        )
        return False
```

本地模式下（`127.0.0.1`）启动时，仅打印警告，不拒绝启动：

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter.connect()
if not self._api_key:
    logger.warning(
        "[%s] No API key configured (API_SERVER_KEY / platforms.api_server.key). "
        "All requests will be accepted without authentication. "
        "Set an API key for production deployments ...",
        self.name,
    )
```

**10.4 速率限制**

文件：`gateway/platforms/api_server.py`

API Server **不存在请求级别的速率限制中间件**。唯一的限流是 `/v1/runs` 端点的并发限制：

```python
# 文件: gateway/platforms/api_server.py, _handle_runs()
if len(self._run_streams) >= self._MAX_CONCURRENT_RUNS:
    return web.json_response(
        _openai_error(f"Too many concurrent runs (max {self._MAX_CONCURRENT_RUNS})",
                       code="rate_limit_exceeded"),
        status=429,
    )
```

已有的中间件仅有：
- `cors_middleware` -- CORS 处理
- `body_limit_middleware` -- 请求体大小限制
- `security_headers_middleware` -- 响应头安全标志

```python
# 文件: gateway/platforms/api_server.py, APIServerAdapter.connect()
mws = [mw for mw in (cors_middleware, body_limit_middleware, security_headers_middleware)
       if mw is not None]
self._app = web.Application(middlewares=mws)
```

**10.5 与 Webhook 平台的对比**

文件：`gateway/platforms/webhook.py`

Webhook 平台默认绑定 `0.0.0.0:8644`，但实现了完整的安全层：

```python
# 文件: gateway/platforms/webhook.py
DEFAULT_HOST = "0.0.0.0"
DEFAULT_PORT = 8644
```

Webhook 的安全机制：

- HMAC 签名验证（GitHub/GitLab/通用 SHA-256）：
```python
# 文件: gateway/platforms/webhook.py, WebhookAdapter._validate_signature()
def _validate_signature(self, request, body, secret) -> bool:
    gh_sig = request.headers.get("X-Hub-Signature-256", "")
    if gh_sig:
        expected = "sha256=" + hmac.new(
            secret.encode(), body, hashlib.sha256
        ).hexdigest()
        return hmac.compare_digest(gh_sig, expected)
    # GitLab, Generic HMAC-SHA256...
```

- 每路由 HMAC secret 必须配置（启动时校验）：
```python
# 文件: gateway/platforms/webhook.py, WebhookAdapter.connect()
for name, route in self._routes.items():
    secret = route.get("secret", self._global_secret)
    if not secret:
        raise ValueError(
            f"[webhook] Route '{name}' has no HMAC secret. "
            f"Set 'secret' on the route or globally. "
            f"For testing without auth, set secret to '{_INSECURE_NO_AUTH}'."
        )
```

- 每路由速率限制（固定窗口，默认 30 次/分钟）：
```python
# 文件: gateway/platforms/webhook.py, WebhookAdapter.__init__()
self._rate_limit: int = int(config.extra.get("rate_limit", 30))  # per minute
```

- 请求体大小限制（auth-before-body 模式，默认 1MB）：
```python
# 文件: gateway/platforms/webhook.py, WebhookAdapter.__init__()
self._max_body_bytes: int = int(config.extra.get("max_body_bytes", 1_048_576))
```

- 幂等性缓存（TTL 1小时，防止重复投递）：
```python
# 文件: gateway/platforms/webhook.py, WebhookAdapter.__init__()
self._idempotency_ttl: int = 3600  # 1 hour
```

**10.6 会话延续的安全门控**

文件：`gateway/platforms/api_server.py`

通过 `X-Hermes-Session-Id` 头的会话延续在无 API key 时被拒绝：

```python
# 文件: gateway/platforms/api_server.py, _handle_chat_completions()
provided_session_id = request.headers.get("X-Hermes-Session-Id", "").strip()
if provided_session_id:
    if not self._api_key:
        logger.warning(
            "Session continuation via X-Hermes-Session-Id rejected: "
            "no API key configured.  Set API_SERVER_KEY to enable "
            "session continuity."
        )
        return web.json_response(
            _openai_error("Session continuation requires API key authentication. "
                          "Configure API_SERVER_KEY to enable this feature."),
            status=403,
        )
```

### 结论

- 默认绑定 `127.0.0.1`（本地），非 `0.0.0.0` -- **已核实**
- 认证为可选的，默认无认证（仅打印警告） -- **已核实**
- 网络暴露时强制要求 API key（包括占位符检测） -- **已核实**
- 无请求级速率限制中间件，仅有 `/v1/runs` 的并发限制 -- **已核实**
- Webhook 平台有完整的 HMAC 验证、速率限制、幂等性保护 -- **已核实**
- API Server 与 Webhook 的安全机制差距显著 -- **已核实**
- 会话延续在无 API key 时被正确拒绝 -- **已核实**

---

## 总结

| 缺陷 | 核实状态 | 核心发现 |
|------|----------|----------|
| 1. Skill 索引全量注入 | 已核实（部分成立） | 全量注入确认，存在静态过滤（平台/禁用/条件），无语义检索 |
| 2. 自修改无确认 | 已核实 | system prompt 指示"immediately patch"，skill_manage 代码无确认环节 |
| 3. 会话/记忆未隔离 | 已核实（部分成立） | profile 间已隔离（通过 HERMES_HOME），profile 内跨平台/跨用户未隔离 |
| 4. Keyword overlap fitness | 已核实 | skill_fitness_metric 使用 word set overlap，LLMJudge 已定义但未被任何代码调用 |
| 5. skill_text no-op | 已核实 | skill_text 是普通属性（非 DSPy 可优化参数），GEPA 不会修改它 |
| 6. Prompt 注入漏洞 | 已核实 | ConstraintValidator 无 injection 检查，演化 skill text 可包含恶意内容 |
| 7. Pipeline 可靠性 | 已核实 | GEPA->MIPROv2 fallback 存在但粗糙，缺少 checkpoint/early stopping 等机制 |
| 8. 插件系统零安全检查 | 已核实 | importlib 直接执行，无签名/沙箱/权限隔离，可注册工具/注入消息/替换上下文引擎 |
| 9. Skill Hub 信任策略不对等 | 已核实 | agent-created 在 caution 级别直接 allow，community 同级别 block；--force 绕过 block/ask |
| 10. API Server 无认证 | 已核实（部分成立） | 默认 127.0.0.1 无认证仅警告，网络暴露强制要求 key，无请求级速率限制 |
