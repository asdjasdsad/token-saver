---
description: 选择性地导出对话的指定部分为分层快照（detailed + distilled + index）。不影响当前会话上下文。
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
---

# /token-saver-export

选择性导出对话的某个部分，生成与 /token-saver-snapshot 相同结构的三层文件。**当前会话上下文不变**，仅生成外部文件。

## 用法

```
/token-saver-export <部分描述>
/token-saver-export 架构讨论那部分
/token-saver-export 最后 10 轮
/token-saver-export 关于 docker 的所有内容
/token-saver-export                    # 不带参数 → 等同 /token-saver-snapshot 全量
```

## 执行步骤

1. **解析数据目录**：按 SKILL.md 的 Step 1。

2. **读配置**：`~/.token-saver/config.json`。

3. **解析 `<部分描述>`**：
   - 关键词匹配（如 "docker" → 包含此词的对话段）
   - 范围匹配（如 "最后 10 轮" → 截取末尾 N 轮）
   - 主题匹配（如 "架构讨论" → 识别讨论架构的连续段落）
   - 如果无法识别（描述太模糊），询问用户具体指哪些消息序号或主题
   - 若用户没有传参数，等同 /token-saver-snapshot 的全量模式

4. **生成主题描述**（基于选定的部分内容，不是整个会话）：
   - 5-15 字中文，反映选中部分的核心
   - 例：用户说 "导出架构那部分" → `架构选型与pd分离决策`

5. **构造文件路径 + 写三层文件**（同 snapshot 命令）：
   ```
   <data_dir>/notes/<timestamp>-<描述>.distilled.md
   <data_dir>/notes/<timestamp>-<描述>.detailed.md
   <data_dir>/notes/<timestamp>-<描述>.index.md
   ```

6. **概念归档（针对选中部分）**：
   - 从选中的对话片段中识别概念
   - 追加或新建概念文件（同 snapshot 流程）

7. **报告**：
   ```
   ✓ 已导出选定部分：
     - 选中范围：<对哪些消息/主题>
     - 提炼版：<distilled-path>
     - 详细版：<detailed-path>
     - 索引：<index-path>
   
   当前会话上下文未变。如需载入，/clear 后 @<distilled-path>。
   ```

## 与 snapshot/archive 的区别

| 命令 | 范围 | 输出 | 是否分层 |
|------|------|------|---------|
| /token-saver-snapshot | 全部对话 | notes/ 三文件 | 是 |
| /token-saver-export <X> | 用户指定部分 | notes/ 三文件 | 是 |
| /token-saver-archive | 当前轮 Q&A | qa-cache/concepts/ 单文件 | 否（太短） |

## 注意事项

- export 不会修改当前会话的任何内容，仅产出磁盘文件。
- 如果用户描述含糊，先回问具体指哪部分，而不是猜测后导出错误内容。
- 如果选中部分太短（< 2 轮交流），告诉用户"选中内容太少，建议直接用 /token-saver-archive 归档单轮"。
