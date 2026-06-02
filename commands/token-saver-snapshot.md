---
description: 立即生成当前对话的分层快照（detailed + distilled + index 三文件），并把代表性问答归档到概念知识库
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
---

# /token-saver-snapshot

立即把当前对话浓缩成三层 markdown 文件：
- `*.distilled.md` — 决策、结论、关键代码（长期保留，下次会话载入这个）
- `*.detailed.md` — 调试过程、错误、放弃的方案（短期参考）
- `*.index.md` — 两个文件的导航 + 主线/支线划分

## 执行步骤

1. **解析数据目录**：按 SKILL.md 的 Step 1 解析 `<data_dir>`（环境感知）。

2. **读配置**：`~/.token-saver/config.json`，取 `naming` 和 `snapshot.layered` 字段。

3. **生成主题描述**：
   - 根据当前对话内容，提炼一个 5-15 字的中文短描述
   - 例：`endpoint列表页优化与docker部署`、`k8s-service路由排障`
   - 文件名安全（去掉 `/ \ : * ? " < > |`）
   - 长度不超过 `naming.max_filename_length`

4. **生成时间戳**：`YYYY-MM-DD-HHMM`（精确到分）

5. **构造文件路径**：
   ```
   <data_dir>/notes/<timestamp>-<描述>.distilled.md
   <data_dir>/notes/<timestamp>-<描述>.detailed.md
   <data_dir>/notes/<timestamp>-<描述>.index.md
   ```

6. **写三个文件**，按 SKILL.md 的"Distilled / Detailed / Index template"。
   - distilled：决策、关键代码、概念定义、API、用户认知、待补充
   - detailed：调试过程、错误、探索性问答、放弃的方案
   - index：链接 + 主线/支线 + 推荐载入

7. **概念归档（同步执行）**：
   - 从对话中识别 1-3 个最重要的概念
   - 对每个概念，按 SKILL.md 的 Step 4 流程，在 `<data_dir>/qa-cache/concepts/<concept>.md` 创建或追加
   - 更新 `<data_dir>/qa-cache/index.md`

8. **报告路径**：
   ```
   ✓ 快照已生成：
     - 提炼版：<distilled-path>
     - 详细版：<detailed-path>
     - 索引：<index-path>
   
   ✓ 概念归档：
     - 已追加到 <concept-A>.md (累计 N 次)
     - 已新建概念 <concept-B>
   
   建议：/clear 后用 @<distilled-path> 继续，可显著降低后续 input token。
   ```

## 注意事项

- 笔记内容必须做浓缩，不是照抄原文。distilled 尤其要紧凑。
- 如果对话内容不足以生成快照（< 3 轮实质性交流），告诉用户"对话内容尚不足以生成快照"，不强行生成。
- 如果同一时间戳已有同名文件（极罕见），追加 `-a3f` 短 hash 避免冲突。
- 配置 `snapshot.layered: false` 时，退化为单文件模式（与 v0.1 兼容）：只生成一个 `<timestamp>-<描述>.md`。
