---
name: 吾日三省吾身
description: Codex-native self-improvement skill. Learns from errors, corrections, and successes across sessions. Triggered manually via /improve or automatically when errors/corrections occur. Uses ~/.codex/memories/ as the unified memory backend.
metadata:
  triggers:
    manual:
      - "/improve"
      - "自我进化"
      - "总结经验"
      - "分析教训"
    auto:
      - "command exits non-zero"
      - "user corrects agent output"
      - "task completed after error recovery"
---

# 吾日三省吾身

> 让 Codex 从每次交互中学习，持续进化。

## 六支柱设计

| 支柱 | 本 skill 的实现 |
|------|----------------|
| 01 上下文管理 | 分段读取记忆，只加载相关模式，大文件用 offset+limit |
| 02 工具系统 | 仅使用 Codex 原生工具：exec_command、apply_patch、update_plan |
| 03 执行编排 | 4 阶段流水线，每阶段有明确的完成判定 |
| 04 状态与记忆 | 统一使用 ~/.codex/memories/，不创建独立记忆体系 |
| 05 评估与观测 | 每次改进输出摘要报告，confidence 评分可追踪 |
| 06 约束与恢复 | 只追加不覆盖、操作前 dry-run、回退有日志 |

---

## 01 上下文管理

**原则：只加载当前任务需要的记忆，不全量读入。**

读取记忆时的规则：
- 语义记忆 (`memory/semantic/patterns.json`)：只读取 `category` 和 `pattern` 字段做匹配，不全量展开
- 情景记忆 (`memory/episodic/`)：只读取最近 7 天的文件，用 `ls -t | head -5` 定位
- 任何记忆文件 > 100 行时，先用 `head -30` 取摘要，确认相关后再全文读取
- 上下文中同时存在的记忆文件不超过 3 个

---

## 02 工具系统

**原则：只用 Codex 原生工具，不引入依赖。**

| 操作 | Codex 工具 |
|------|-----------|
| 读取记忆 | `exec_command` (`cat`, `head`, `rg`) |
| 写入记忆 | `apply_patch` 或 `exec_command` (`cat >>`) |
| 搜索记忆 | `exec_command` (`rg`, `jq`) |
| 更新计划 | `update_plan` |
| 用户交互 | 直接回复（commentary / final channel） |

---

## 03 执行编排

### /improve 命令流程

```
阶段 1: 提取经验 (Extract)
  ├── 扫描当前会话中的错误/纠正/成功
  ├── 检查 memory/episodic/ 最近 7 天是否有未处理的情景
  └── 输出: 1-3 条原始经验（未抽象）
  
阶段 2: 抽象模式 (Abstract)
  ├── 对比语义记忆中的现有模式
  ├── 判断: 新模式 / 强化已有模式 / 降级已有模式
  └── 输出: pattern_name + confidence 变化

阶段 3: 更新记忆 (Update)
  ├── 写入情景记忆 → memory/episodic/YYYY-MM-DD-{slug}.json
  ├── 更新语义记忆 → memory/semantic/patterns.json
  └── 追加改进日志 → memory/improve-log.jsonl

阶段 4: 输出报告 (Report)
  └── 向用户输出: 提取了什么、模式如何变化、confidence 多少
```

**每阶段完成判定**：
- 阶段 1：至少提取 1 条经验，或明确告知"本次无可提取内容"
- 阶段 2：每条经验都有判定结果（新增/强化/降级/忽略）
- 阶段 3：写入成功，文件存在且 JSON 合法
- 阶段 4：用户收到可读的摘要

### 自动触发条件

当以下情况发生时，agent 主动建议运行 /improve：
- `exec_command` 返回非零退出码且 agent 自行修复了问题
- 用户说"不对"、"不是这样"、"错了"并给出纠正
- 一个任务经历了 3 次以上的迭代才完成

自动触发只建议，不强制执行。用户说"不用"就跳过。

---

## 04 状态与记忆

**原则：统一使用 ~/.codex/memories/，不创建独立的记忆体系。**

### 记忆架构

```
~/.codex/skills/self-improving-agent/memory/
├── semantic/
│   └── patterns.json          # 累积的模式库
├── episodic/
│   ├── 2026-05-24-context-overflow.json
│   └── ...                    # 按日期的情景记录
└── improve-log.jsonl          # 追加式改进日志
```

### 与 Codex 记忆系统的关系

| 本 skill 记忆 | Codex 记忆 | 关系 |
|--------------|-----------|------|
| `semantic/patterns.json` | `MEMORY.md` | 互补：patterns 是结构化模式，MEMORY.md 是自由文本 |
| `episodic/*.json` | `rollout_summaries/` | 互补：episodic 是精选经验，rollout 是完整记录 |
| `improve-log.jsonl` | 无对应 | 本 skill 独有 |

**同步规则**：当 /improve 提取到高价值模式时（confidence >= 0.9），主动建议用户写入 `~/.codex/memories/extensions/ad_hoc/notes/`。

### 记忆文件格式

**情景记忆** (`memory/episodic/YYYY-MM-DD-{slug}.json`):
```json
{
  "id": "ep-YYYY-MM-DD-NNN",
  "timestamp": "YYYY-MM-DDTHH:MM:SS+08:00",
  "situation": "一句话描述发生了什么",
  "root_cause": "根本原因",
  "solution": "解决方案",
  "lesson": "一句话教训",
  "pattern_match": "匹配的语义模式 ID，或 null",
  "confidence_delta": "+0.05 或 -0.1",
  "source": "manual | auto-error | auto-correction | auto-success"
}
```

**语义模式** (`memory/semantic/patterns.json`):
```json
{
  "patterns": {
    "pattern_slug": {
      "id": "pat-YYYY-MM-DD-NNN",
      "name": "人类可读名称",
      "category": "coding | workflow | tool-usage | communication",
      "pattern": "一句话描述模式",
      "problem": "解决什么问题",
      "solution": "一句话方案",
      "confidence": 0.0-1.0,
      "applications": 0,
      "last_applied": "YYYY-MM-DD",
      "created": "YYYY-MM-DD",
      "status": "active | deprecated | candidate"
    }
  }
}
```

### 记忆清理规则

- 情景记忆保留 90 天，超期归档到 `memory/episodic/archive/`
- 语义模式 confidence < 0.3 且 30 天未应用 → 标记 `deprecated`
- `deprecated` 模式 60 天未应用 → 从 patterns.json 移除，记录到 improve-log
- 清理操作仅在 `/improve` 时执行，不自动触发

---

## 05 评估与观测

### Confidence 评分规则

| 事件 | confidence 变化 |
|------|----------------|
| 模式首次提取 | 0.5 (candidate) |
| 模式被再次验证 | +0.1 |
| 模式导致成功结果 | +0.05 |
| 模式导致失败/纠正 | -0.15 |
| 用户明确确认模式有效 | +0.2 |
| 用户明确说模式不对 | -0.3 |
| 上限 | 0.99 |
| 下限（触发 deprecated） | 0.3 |

### 改进报告格式

每次 /improve 执行后，输出：

```
## 改进报告 YYYY-MM-DD

**提取经验**: N 条
- [经验1摘要]
- [经验2摘要]

**模式变化**:
- 🆕 新增: pattern-name (confidence: 0.5)
- ⬆️ 强化: pattern-name (0.7 → 0.8)
- ⬇️ 降级: pattern-name (0.6 → 0.45)
- ➡️ 不变: pattern-name (0.85)

**记忆统计**: N 条情景 | M 条模式 | K 条活跃模式
**建议**: [如果有高价值模式，建议写入 Codex memories]
```

### 自检

当 `/improve` 执行时，顺便检查：
1. `patterns.json` 是否合法 JSON
2. 是否有情景文件无法解析
3. 是否有模式超过 30 天未应用
异常项列入报告的"警告"部分。

---

## 06 约束与恢复

### 约束

1. **只追加不覆盖**: 情景记忆和 improve-log 只追加，不修改已有条目
2. **语义模式修改需确认**: confidence 变化 > 0.2 时，向用户说明变化原因
3. **不修改 AGENTS.md**: 本 skill 不修改任何 AGENTS.md 文件
4. **不修改其他 skill**: 只更新自身 memory/ 目录下的文件
5. **单次提取上限**: 每次 /improve 最多提取 5 条经验，防止上下文溢出
6. **大文件保护**: patterns.json > 200 行时提醒用户考虑清理

### 恢复

1. **improve-log 是审计日志**: 每次操作记录 before/after，可用于回退
2. **情景记忆不可变**: 已写入的情景文件不被修改，可回溯
3. **语义模式有 status**: deprecated 不删除，可恢复为 active
4. **写入前验证**: patterns.json 写入后立即用 `python3 -c "import json; json.load(open(...))"` 验证合法性，失败则回退

---

## 快速上手

### 手动触发
```
用户: /improve
或:   总结一下这次的经验
或:   自我进化
```

### 自动触发
Agent 在修复错误、接受用户纠正、或多次迭代后主动建议：
```
"刚才修复了一个问题，要运行 /improve 记录一下吗？"
```

### 首次使用
1. 确认 `memory/semantic/patterns.json` 存在（空 `{"patterns": {}}` 即可）
2. 确认 `memory/episodic/` 目录存在
3. 确认 `memory/improve-log.jsonl` 存在（空文件即可）
4. 开始正常工作，/improve 会自动积累记忆
