# Hermes Agent（Nous Research）

"自我进化" Agent。2026 年 2 月公开，50 天 81,000+ Stars，融资 6,500 万美元（Paradigm 领投）。

声称具备自我进化能力。源码审计结论：不存在有效进化机制。

## 一句话结论

整条"进化"链赌的是 LLM 永远正确、零幻觉、在超长上下文中自主掌控注意力——然后把所有工程兜底全拆了。

## 决定性硬伤（10 项，均有源码证据）

- **Skill 全量注入** — 所有 skill 索引塞进 system prompt，无裁剪无检索，固定开销占每次 API 调用的 73%
- **词袋评分** — "进化"评分用的是关键词重叠，不是语义评估；写了 LLMJudge 但没接入，即使接入了也缺乏标准
- **Pipeline no-op** — GEPA 优化后的 skill 文本大概率没变过，流程跑了但输出和输入一样
- **错误叠加链** — 生成测试、执行、打分、突变全靠 LLM，自己出卷自己考自己判
- **自修改无确认** — prompt 直接指示 agent "立即修补不要等确认"，修改后无回归验证
- **注入防护缺失** — 进化产出的 skill 文本无注入检查，主仓库有防护但没集成到进化流程
- **Pipeline 可靠性缺失** — 无 checkpoint、无 early stopping、fallback 粗糙
- **插件系统零安全检查** — 可执行 Python 代码直接 import 进程，无签名无沙箱无权限限制
- **Skill Hub 信任策略不对等** — 自己的进化产出几乎不拦，社区 skill 严格审查
- **API Server 无认证** — 安全系统做了不少，但入口裸奔，无 API key 无速率限制

## 文件索引

- **[report.md](report.md)** — 完整调研报告。时间线全貌、硬伤拆解、同类对比、抄袭争议。**从头读这个。**
- **[short-comment.md](short-comment.md)** — 短评。评论区/社媒发帖用，三分钟看完。
- **[appendix-source-audit.md](appendix-source-audit.md)** — 源码审计。10 项缺陷的代码证据（文件路径 + 行号 + 代码片段）。
- **[appendix-source-verification.md](appendix-source-verification.md)** — 源码核实记录。逐文件读取仓库源码的完整过程。
- **[appendix-findings.md](appendix-findings.md)** — 原始调研数据。所有来源链接、社区反馈、媒体报道汇总。
