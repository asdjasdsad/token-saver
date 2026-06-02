---
description: 立即生成当前对话的快照笔记，写入 ~/.claude/token-saver/notes/，并提取关键 Q&A 入档到 qa-cache
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
---

# /token-saver-snapshot

立即把当前对话浓缩成一份主题分区的 markdown 笔记。

## 执行步骤

1. **读配置**：读 `~/.claude/token-saver/config.json`，如不存在用默认值。
2. **识别主题**：根据当前对话上下文，确定一个主题 slug（kebab-case，如 `k8s-service`、`tokenhub-mu`）。如果对话跨越多个主题，主题取最主要的一个，并在笔记开头注明涉及的次要主题。
3. **生成笔记**：写到 `~/.claude/token-saver/notes/<YYYY-MM-DD>-<topic-slug>.md`，使用 SKILL.md 中定义的快照模板：
   - 核心概念（按用户实际讨论过的）
   - 关键问答（保留用户原始问题的措辞，浓缩答案保留必要的表格/对比/类比）
   - 用户表达过的理解（哪些是确认正确的、哪些是被纠正过的）
   - 待补充
4. **入档关键 Q&A**：
   - 在 `~/.claude/token-saver/qa-cache/<topic>/` 创建 2-4 个代表性 Q&A 文件（命名 `<YYYY-MM-DD>-NNN.md`）。
   - 更新或创建 `<topic>/_index.md`：每条一行 `[NNN] 一句话问题摘要`。
   - 更新或创建全局 `qa-cache/index.md`：登记本次主题（如未登记过）+ 一行主要内容描述。
5. **报告路径**：告诉用户：
   - 笔记文件路径
   - 涉及的主题
   - 入档的 Q&A 数量
   - 建议下一步：`/clear` 后在新会话中 `@<note-path>` 继续

## 注意

- 笔记内容不要照抄对话原文，必须做浓缩（去客套、去复述、保留核心）。
- 如果当前对话还没有可总结的实质内容（比如刚开始 1-2 轮闲聊），告诉用户"对话内容尚不足以生成快照"，不要强行生成。
- 已存在同名笔记文件时，追加而非覆盖，并在文件头部加分隔线和新日期。
