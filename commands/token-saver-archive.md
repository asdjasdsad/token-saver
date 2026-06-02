---
description: 把最近一轮 Q&A 按概念归档到 qa-cache/concepts/，相同概念追加而非新建
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
---

# /token-saver-archive

只归档"最近这一轮"的问答到概念知识库。同一概念的多次讨论会**累积到同一个文件**，不会按日期产生多份。

适合场景：用户问了一个有价值的问题，得到了好答案，想把这个 Q&A 存入跨会话知识库。

## 执行步骤

1. **解析数据目录**：按 SKILL.md 的 Step 1。

2. **读配置**：`~/.token-saver/config.json`。

3. **提取概念**：
   - 从最近一轮 Q&A 内容中识别 1-3 个概念标签
   - 概念是抽象主题，不是具体场景
   - 例：
     - "Mac docker 构建报 platform 错" → `docker-镜像构建-推送`
     - "K8s Service 类型对比" → `k8s-service-类型与差异`
   - 如果一轮 Q&A 涉及多个概念，每个都做归档

4. **对每个概念，匹配现有文件**：
   - 路径：`<data_dir>/qa-cache/concepts/<concept>.md`
   - 完全匹配 → append 模式
   - 语义高度相似（用判断力）→ append 模式
   - 都不匹配 → create 模式

5. **写概念文件**：

   **新概念（create 模式）**：
   ```markdown
   # <概念名>

   > 概念标签：tag1, tag2, tag3
   > 创建：YYYY-MM-DD
   > 最近更新：YYYY-MM-DD
   > 累计问答次数：1

   ## 核心知识

   ### <稳定子主题>
   [本次得到的稳定结论或方法]

   ### 常见坑（如果讨论中提到）
   - 坑 1
   - 坑 2

   ## 历史问答记录

   ### YYYY-MM-DD — <一句话问题摘要>
   **Q**: <用户原始问题（保留口语化）>
   **A**: <浓缩答案，保留必要的代码/表格/对比>
   ```

   **已存在概念（append 模式）**：
   - 在 "## 历史问答记录" 下追加新条目
   - 如果新内容修正或扩展了 "## 核心知识" 中的内容，更新核心知识区
   - 头部 metadata：
     - "最近更新" 改为今天
     - "累计问答次数" +1

6. **更新 `<data_dir>/qa-cache/index.md`**：

   **新概念**：在文件中追加一段：
   ```markdown
   ## <concept-name>
   - 文件：./concepts/<concept-name>.md
   - 主要内容：<一句话>
   - 累计：1 次  最近：YYYY-MM-DD
   ```

   **已有概念**：找到对应段落，更新累计次数和最近日期。

7. **报告**：
   ```
   ✓ 归档完成：
     - 已追加到 docker-镜像构建-推送.md（累计 3 次）
     - 已新建概念 k8s-service-路由
   ```

## 与 snapshot/export 的区别

| 命令 | 范围 | 输出位置 | 输出形式 |
|------|------|---------|---------|
| /token-saver-snapshot | 全部对话 | notes/ | 三层文件 |
| /token-saver-export | 指定部分 | notes/ | 三层文件 |
| /token-saver-archive | 当前轮 | qa-cache/concepts/ | 概念文件追加 |

## 注意事项

- 归档单位是"当前轮"，不是整个会话。如果想归档多轮，用 /token-saver-export。
- 概念匹配优先 append。新建概念是为新主题留的，避免概念碎片化。
- 如果用户连续 archive 同一概念，一律追加到同一文件，确保概念累积。
- 不删除旧条目，只追加（除非用户用 /token-saver-clean）。
