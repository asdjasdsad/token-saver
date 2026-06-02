---
description: 清理 qa-cache 中的过期或低价值概念条目，保持知识库精简
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - AskUserQuestion
---

# /token-saver-clean

清理 `<data_dir>/qa-cache/concepts/` 中过期或低价值的概念条目。

## 执行步骤

1. **解析数据目录**：按 SKILL.md 的 Step 1。

2. **询问保留期 / 清理策略**：用 AskUserQuestion 让用户选：
   - 30 天（默认）：删除最近 30 天没更新过的概念
   - 90 天：保留更久
   - 1 年
   - 按累计次数：累计 < N 次的概念视为低价值
   - 不按时间清理，只看清单手动选

3. **扫描所有概念文件**：
   - 列出 `<data_dir>/qa-cache/concepts/*.md`
   - 读取每个文件的 metadata（创建日期、最近更新、累计次数）
   - 按用户选择的策略标记待清理候选

4. **展示清理清单**：
   ```
   待清理（X 个概念）：
     - docker-镜像构建-推送.md（最近更新 2026-04-01，累计 1 次）
     - 旧实验性概念.md（最近更新 2026-03-15，累计 2 次）
     ...
   保留：Y 个
   ```

5. **二次确认**：用 AskUserQuestion 选：
   - 全部清理
   - 选择性保留（用户列出要保留的概念名）
   - 取消

6. **执行清理**：
   - 删除选定的概念文件
   - 更新 `<data_dir>/qa-cache/index.md`：移除被删条目段落
   - 不删除 `notes/` 下的快照笔记（笔记保留意义大）
   - 记录到 `<data_dir>/.meta/clean-log.md`：清理时间、清理列表

7. **报告结果**：
   ```
   ✓ 清理完成：
     删除概念：X 个
     保留概念：Y 个
     释放索引开销：约 Z tokens
   ```

## 注意事项

- 不动 `notes/` 下的快照（detailed/distilled/index 三件套），它们保留长期价值
- 不动 `concepts/` 下高频概念（即使日期老）—— 累计次数高说明仍在用
- 删除前必须二次确认
- clean-log 留存便于追溯，万一误删可手动恢复
- 如果某主题清理后导致 index.md 段落过期，自动同步清理

## 与其他命令的关系

- `clean` 只处理 `qa-cache/concepts/`，不动 `notes/`
- 想清理 `notes/`：手动删除文件
- 想合并相似概念：v0.2 不提供，手动编辑 concepts/ 文件
