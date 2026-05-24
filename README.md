# 吾日三省吾身 (Daily Self-Review)

> Codex 原生的自我改进 skill。从错误、纠正和成功中学习，跨会话持续进化。

灵感来自 [ClawHub: pskoett/self-improving-agent](https://clawhub.ai/pskoett/self-improving-agent)（Claude Code 版），基于 harness 工程六支柱重新设计，完全适配 Codex CLI / Codex Desktop。

## 为什么需要它？

LLM agent 每次会话都是一张白纸。你纠正过它的错误、教过它的工作流，下次全忘了。

**吾日三省吾身** 解决这个问题：

- 遇到错误 → 记录下来，下次不犯
- 被纠正 → 提炼模式，自动复用
- 做对了 → 强化信心，持续可靠

## 六支柱设计

| 支柱 | 本 skill 的实现 |
|------|----------------|
| **01 上下文管理** | 分段读取记忆，只加载相关模式，大文件用 offset+limit |
| **02 工具系统** | 仅使用 Codex 原生工具：`exec_command`、`apply_patch`、`update_plan` |
| **03 执行编排** | 4 阶段流水线（Extract → Abstract → Update → Report），每阶段有完成判定 |
| **04 状态与记忆** | 语义记忆（模式）+ 情景记忆（经验）双层架构 |
| **05 评估与观测** | confidence 评分体系、改进报告、JSON 自检 |
| **06 约束与恢复** | 只追加不覆盖、操作前验证、审计日志可回溯 |

## 快速安装

```bash
# 1. Clone 到 Codex skills 目录
git clone https://github.com/deckdco/daily-self-review.git \
  ~/.codex/skills/吾日三省吾身

# 2. 初始化记忆文件（空库）
echo '{"patterns":{}}' > ~/.codex/skills/吾日三省吾身/memory/semantic/patterns.json
touch ~/.codex/skills/吾日三省吾身/memory/improve-log.jsonl
```

## 使用方式

### 手动触发

在 Codex 对话中输入：

```
/improve
总结经验
自我进化
```

### 自动触发

Agent 在以下情况会主动建议运行：
- 命令执行失败且 agent 自行修复后
- 用户纠正了 agent 的输出
- 一个任务经历了 3 次以上迭代才完成

### 输出示例

```
## 改进报告 2026-05-24

**提取经验**: 2 条
- 批量处理 128 个视频时上下文溢出
- webReader MCP 反复失败

**模式变化**:
- 🆕 新增: batch-session-split (confidence: 0.8)
- 🆕 新增: fail-fast-switch-method (confidence: 0.75)

**记忆统计**: 7 条情景 | 5 条模式 | 5 条活跃模式
```

## 记忆架构

```
memory/
├── semantic/
│   └── patterns.json       # 累积的模式库（分类、confidence、应用次数）
├── episodic/
│   └── YYYY-MM-DD-slug.json  # 按日期的情景记录
└── improve-log.jsonl        # 追加式审计日志
```

### 语义模式示例

```json
{
  "native-encoding-not-postfix": {
    "id": "pat-2026-05-24-003",
    "name": "Native Encoding Not Postfix",
    "category": "coding",
    "pattern": "中文标点在代码中直接使用，永远不对整个文件做全局替换",
    "confidence": 0.85,
    "applications": 3,
    "status": "active"
  }
}
```

### 情景记忆示例

```json
{
  "id": "ep-2026-05-24-001",
  "situation": "Word 导出时全局替换标点破坏了 Python 代码",
  "root_cause": "事后全局替换无法区分代码引号和内容引号",
  "solution": "在 python-docx 脚本中直接使用中文标点",
  "lesson": "中文标点应在代码中直接使用，不做全局替换"
}
```

## Confidence 评分

```
0.0 ──── 0.3 ──── 0.5 ──── 0.7 ──── 0.9 ──── 0.99
 │         │         │         │         │         │
deprecated  candidate  新提取    已验证     高可信     极高可信
(清理)    (待确认)  (首次)    (被复用)   (多次验证)  (用户确认)
```

| 事件 | 变化 |
|------|------|
| 模式首次提取 | 0.5 |
| 模式被再次验证 | +0.1 |
| 用户确认有效 | +0.2 |
| 用户说不对 | -0.3 |
| 模式导致失败 | -0.15 |

## 安全约束

- 情景记忆和日志**只追加**，不修改已有条目
- confidence 变化 > 0.2 时向用户说明原因
- 不修改任何 AGENTS.md 或其他 skill
- 每次写入后验证 JSON 合法性，失败立即回退
- 单次提取上限 5 条经验，防止上下文溢出

## 目录结构

```
吾日三省吾身/
├── SKILL.md                    # 核心指令，agent 读取的入口
├── README.md                   # 本文件
├── .gitignore                  # 排除个人记忆数据
├── memory/
│   ├── semantic/
│   │   └── patterns.json       # 模式库（不上传）
│   ├── episodic/               # 情景记录（不上传）
│   └── improve-log.jsonl       # 审计日志（不上传）
├── templates/
│   ├── pattern-template.md     # 新模式模板
│   ├── correction-template.md  # 纠正模板
│   └── validation-template.md  # 自检模板
└── references/
    └── appendix.md             # 分类定义、生命周期、集成说明
```

## 与 ClawHub 版的区别

| | ClawHub 版 (pskoett) | 本版 |
|---|---|---|
| 目标平台 | Claude Code | Codex CLI / Desktop |
| 触发机制 | Hooks 自动触发 | 手动 `/improve` + 自动建议 |
| 工具系统 | Read/Write/Edit/Bash/Grep | exec_command/apply_patch/update_plan |
| 记忆路径 | ~/.claude/memory/ | ~/.codex/skills/吾日三省吾身/memory/ |
| 约束系统 | 隐含在 skill 逻辑中 | 六支柱显式设计 |
| 理论基础 | 2025 终身学习论文 | harness 工程六支柱 + 同批论文 |

## 致谢

- [pskoett/self-improving-agent](https://clawhub.ai/pskoett/self-improving-agent) — 原始灵感来源
- [SimpleMem](https://arxiv.org/html/2601.02553v1) — 高效终身记忆
- [Multi-Memory Survey](https://dl.acm.org/doi/10.1145/3748302) — 语义 + 情景双层记忆
- [Lifelong Learning of LLM Agents](https://arxiv.org/html/2501.07278v1) — 持续学习框架

## License

MIT
