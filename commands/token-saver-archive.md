---
description: 把当前轮（最近一个用户问题 + 你的回答）单独归档到 qa-cache，不生成完整笔记
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
---

# /token-saver-archive

只归档"最近这一轮"的问答到 qa-cache，不生成完整会话快照。

适合场景：用户问了一个有价值的问题，得到了好答案，想把这个 Q&A 存下来供将来引用。

## 执行步骤

1. **读配置**：读 `~/.claude/token-saver/config.json`。
2. **识别主题**：根据本轮 Q&A 内容，确定主题 slug。
3. **写 Q&A 文件**：
   - 路径：`~/.claude/token-saver/qa-cache/<topic>/<YYYY-MM-DD>-<NNN>.md`，NNN 是当天该主题下的递增序号。
   - 格式：
     ```markdown
     # <一句话问题摘要>

     > 日期：YYYY-MM-DD
     > 主题：<topic>

     ## Q
     <用户原始问题>

     ## A
     <浓缩后的答案。保留表格、对比、关键代码示例。去掉过程性描述。>
     ```
4. **更新索引**：
   - 在 `<topic>/_index.md` 追加一行：`[NNN] 一句话摘要 → ./<filename>`
   - 如果该主题是新增的，在全局 `qa-cache/index.md` 登记。
5. **报告**：告诉用户文件路径，确认归档完成。

## 与 snapshot 的区别

- `/token-saver-snapshot`：浓缩**整个会话**，输出多文件（笔记 + 多条 Q&A）。
- `/token-saver-archive`：只归档**本轮 Q&A**，输出单条记录。
