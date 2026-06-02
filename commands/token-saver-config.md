---
description: 查看或修改 token-saver 配置（数据目录、紧凑档位、阈值、qa-cache 开关、命名规则等）
allowed-tools:
  - Read
  - Write
  - Edit
  - AskUserQuestion
---

# /token-saver-config

查看或修改 `~/.token-saver/config.json`。

> 注意：v0.2.0 起配置文件路径从 `~/.claude/token-saver/config.json` 改为 `~/.token-saver/config.json`（产品中立路径）。

## 执行步骤

1. **读配置**：读 `~/.token-saver/config.json`。如不存在则用默认值并写入文件。

   默认配置：
   ```json
   {
     "data_dir": null,
     "compact_level": 2,
     "snapshot": {
       "auto_threshold_tokens": 50000,
       "enabled": true,
       "layered": true
     },
     "qa_cache": {
       "enabled": true,
       "quiet_mode": false,
       "max_concepts_to_scan": 30
     },
     "naming": {
       "use_chinese": true,
       "max_filename_length": 60
     }
   }
   ```

2. **解析当前生效的 data_dir**：调用 SKILL.md Step 1 的解析逻辑，告诉用户实际数据目录。

3. **展示当前配置 + 实际生效路径**：
   ```
   配置文件：~/.token-saver/config.json
   实际数据目录（解析后）：<resolved-path>
   
   当前配置：
     data_dir：null（自动检测）
     compact_level：2 (去客套+去重复)
     snapshot.auto_threshold_tokens：50000
     snapshot.enabled：true
     snapshot.layered：true（detailed + distilled 三层输出）
     qa_cache.enabled：true
     qa_cache.quiet_mode：false
     qa_cache.max_concepts_to_scan：30
     naming.use_chinese：true
     naming.max_filename_length：60
   ```

4. **询问是否修改**：用 AskUserQuestion 让用户选择：
   - 修改紧凑档位（1/2/3）
   - 设置/清空 data_dir（覆盖自动检测）
   - 修改自动快照阈值
   - 启用/禁用自动快照
   - 启用/禁用分层快照
   - 启用/禁用 qa-cache
   - 启用/禁用 quiet_mode
   - 修改命名规则（中文 / 文件名长度）

5. **写回配置**：用户确认后，更新 `~/.token-saver/config.json`。

## 配置项说明

| 字段 | 默认 | 含义 |
|------|------|------|
| `data_dir` | null | null 时自动检测；填字符串时强制使用该路径 |
| `compact_level` | 2 | 1=只去客套；2=去客套+去重复（推荐）；3=极简 |
| `snapshot.auto_threshold_tokens` | 50000 | 估算 messages 超过该值时自动快照 |
| `snapshot.enabled` | true | 是否启用自动快照 |
| `snapshot.layered` | true | 是否分层（detailed + distilled + index 三文件） |
| `qa_cache.enabled` | true | 是否使用问答档案 |
| `qa_cache.quiet_mode` | false | true 时不主动扫描 index |
| `qa_cache.max_concepts_to_scan` | 30 | 扫描时最多检查的概念数 |
| `naming.use_chinese` | true | 文件名是否允许中文 |
| `naming.max_filename_length` | 60 | 文件名长度上限 |

## 数据目录解析顺序

1. `data_dir` 字段（本配置文件）
2. `TOKEN_SAVER_DATA_DIR` 环境变量
3. 自动检测产品（Claude Code → `~/.claude/token-saver/`，CodeBuddy → `~/.codebuddy/token-saver/`）
4. 兜底：`~/.token-saver/data/`
