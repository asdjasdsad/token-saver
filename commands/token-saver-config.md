---
description: 查看或修改 token-saver 配置（紧凑档位、阈值、qa-cache 开关等）
allowed-tools:
  - Read
  - Write
  - Edit
  - AskUserQuestion
---

# /token-saver-config

查看或修改 `~/.claude/token-saver/config.json`。

## 执行步骤

1. **读配置**：读 `~/.claude/token-saver/config.json`，如不存在则用默认值并写入文件。

   默认配置：
   ```json
   {
     "compact_level": 2,
     "snapshot": {
       "auto_threshold_tokens": 50000,
       "enabled": true
     },
     "qa_cache": {
       "enabled": true,
       "quiet_mode": false,
       "max_topics_to_scan": 20
     }
   }
   ```

2. **展示当前配置**：以易读的方式显示当前各项配置和含义。

3. **询问是否修改**：用 AskUserQuestion 让用户选择要修改的项：
   - 紧凑档位（1/2/3）
   - 自动快照阈值
   - 是否启用自动快照
   - 是否启用 qa-cache 检索
   - 是否进入 qa-cache 安静模式（手动 @ 引用而非自动扫描）

4. **写回配置**：用户确认后，更新 `config.json`。

## 配置项说明

| 字段 | 默认 | 含义 |
|------|------|------|
| `compact_level` | 2 | 1=只去客套；2=去客套+去重复（推荐）；3=极简，去类比和举例 |
| `snapshot.auto_threshold_tokens` | 50000 | 估算 messages 超过该值时自动生成快照 |
| `snapshot.enabled` | true | 是否启用自动快照（false 时只能用 /token-saver-snapshot 手动触发）|
| `qa_cache.enabled` | true | 是否使用问答档案 |
| `qa_cache.quiet_mode` | false | true 时不主动扫描 index，只在用户 @ 文件时才用缓存 |
| `qa_cache.max_topics_to_scan` | 20 | 扫描索引时最多检查的主题数（防止缓存膨胀后变慢）|
