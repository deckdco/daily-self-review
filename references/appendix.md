# Appendix

## Memory Directory Layout

```
~/.codex/skills/self-improving-agent/
├── SKILL.md                    # 本文件，skill 入口
├── memory/
│   ├── semantic/
│   │   └── patterns.json       # 结构化模式库
│   ├── episodic/
│   │   ├── YYYY-MM-DD-slug.json  # 按日期的情景记录
│   │   └── archive/              # 90 天以上的归档
│   └── improve-log.jsonl       # 追加式审计日志
├── templates/
│   ├── pattern-template.md
│   ├── correction-template.md
│   └── validation-template.md
└── references/
    └── appendix.md             # 本文件
```

## Category 分类

| Category | 说明 | 示例 |
|----------|------|------|
| coding | 代码相关的模式 | 优先用 rg 而非 grep |
| workflow | 工作流模式 | 批量任务拆分多个会话 |
| tool-usage | 工具使用模式 | apply_patch 优于 cat 写文件 |
| communication | 沟通模式 | 先说结论再说细节 |

## Confidence 生命周期

```
0.0 ──── 0.3 ──── 0.5 ──── 0.7 ──── 0.9 ──── 0.99
 │         │         │         │         │         │
deprecated  candidate  新提取    已验证     高可信     极高可信
(清理)    (待确认)  (首次)    (被复用)   (多次验证)  (用户确认)
```

## 与 Codex 记忆系统的集成点

当 /improve 产生高价值模式时 (confidence >= 0.9)，建议写入:
```
~/.codex/memories/extensions/ad_hoc/notes/{timestamp}-pattern-{slug}.md
```

格式:
```markdown
# Pattern: {name}

{pattern_description}
Confidence: {confidence}
Source: self-improving-agent / {episode_id}
```

## 改进日志格式 (improve-log.jsonl)

每行一个 JSON 对象:
```json
{"ts":"YYYY-MM-DDTHH:MM:SS","type":"extract","episodes":["ep-..."],"patterns":{"new":[],"reinforced":["pat-..."],"deprecated":[]},"confidence_changes":{"pat-...":{"before":0.7,"after":0.8}}}
```

## Research Basis

- [SimpleMem: Efficient Lifelong Memory for LLM Agents](https://arxiv.org/html/2601.02553v1) — 模式累积
- [A Survey on the Memory Mechanism of Large Language Model Agents](https://dl.acm.org/doi/10.1145/3748302) — 语义 + 情景双层记忆
- [Lifelong Learning of LLM based Agents](https://arxiv.org/html/2501.07278v1) — 持续学习
- [ClawHub: pskoett/self-improving-agent](https://clawhub.ai/pskoett/self-improving-agent) — 原始灵感来源 (Claude Code 版)
