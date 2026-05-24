## Self-Validation Report: {date}

### Checks
1. **patterns.json 合法性**: ✅ / ❌
2. **情景文件可解析**: ✅ / ❌ (失败: {failed_files})
3. **过期模式检测**: ✅ 无过期 / ⚠️ {count} 个模式超过 30 天未应用
4. **记忆文件大小**: ✅ / ⚠️ patterns.json 已达 {lines} 行

### Pattern Health
| Pattern | Confidence | Last Applied | Status |
|---------|-----------|--------------|--------|
| {name} | {conf} | {date} | {status} |

### Recommended Actions
- {action_1}
- {action_2}

### Stats
- 情景记忆: {episodic_count} 条
- 语义模式: {pattern_count} 条 (active: {active_count}, deprecated: {deprecated_count})
- 改进日志: {log_entries} 条
