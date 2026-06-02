---
description: 清理 qa-cache 中的过期条目（默认 30 天前），保持索引精简
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - AskUserQuestion
---

# /token-saver-clean

清理 `~/.claude/token-saver/qa-cache/` 中过期或低价值的条目。

## 执行步骤

1. **询问保留期**：用 AskUserQuestion 让用户选：
   - 30 天（默认）
   - 90 天
   - 1 年
   - 自定义天数
   - 不按时间清理，只看清单手动选

2. **扫描所有 Q&A 文件**：
   - 列出 `qa-cache/*/` 下所有 .md 文件
   - 读取每个文件的日期（从文件名 `YYYY-MM-DD-NNN.md` 解析）
   - 标记超过保留期的为待清理候选

3. **展示清理清单**：
   ```
   待清理（X 条）：
     - kubernetes/2026-04-01-001.md  「Pod 怎么生命周期管理」
     - kubernetes/2026-04-15-003.md  「Service 类型对比」
     ...
   保留：Y 条
   ```

4. **二次确认**：让用户用 AskUserQuestion 选：
   - 全部清理
   - 选择性保留（用户列出要保留的文件序号）
   - 取消

5. **执行清理**：
   - 删除选定的 Q&A 文件
   - 更新对应主题的 `_index.md`（移除被删条目的行）
   - 如果某主题清空了，删除主题文件夹并从全局 `index.md` 移除登记
   - 记录到 `~/.claude/token-saver/.clean-log.md`：清理时间、清理数量

6. **报告结果**：告诉用户清理了几个文件、剩余多少、释放了多少 tokens 的索引开销。

## 注意

- 不清理 `notes/` 下的会话快照（笔记保留意义大）
- 删除前要二次确认
- 清理 log 留存便于追溯
