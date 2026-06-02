---
name: token-saver
version: 0.2.0
description: |
  Use this skill at the start of any long-running Claude Code or CodeBuddy
  session to reduce input token consumption. Activates compact output style,
  monitors conversation length to suggest layered snapshots (detailed +
  distilled), and accumulates concept-organized Q&A across sessions.
  Auto-detects whether running in Claude Code or CodeBuddy and writes data
  to the appropriate product directory. Trigger whenever the user mentions
  "save tokens", "reduce token consumption", "long session", "compact output",
  "snapshot conversation", "concept knowledge base", or starts what seems
  likely to be an extended technical conversation.
license: MIT
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
---

# token-saver: Reduce Input Token Consumption (v0.2.0)

You help the user run long Claude Code / CodeBuddy sessions more economically
through four behaviors: compact output style, automatic layered snapshots,
selective export of conversation segments, and a concept-organized cross-session
knowledge base.

This skill is **advisory** — you cannot intercept input or modify conversation
history. What you can do is shape your own outputs, watch session length, and
maintain a knowledge base on disk that the user can leverage in future sessions.

## Why Each Behavior Matters

Long Claude Code conversations grow expensive because the entire message history
is sent as input on every turn. The user is paying for tokens they may not need:
- Repeated explanations of things already discussed in past sessions.
- Verbose answers with filler phrases and over-cautious hedging.
- Conversations that drift across multiple unrelated topics, dragging old context forward.
- Snapshots that include debugging noise alongside real conclusions, forcing the user to re-load all of it on next session.

You address these directly: write tightly, signal when a topic shift makes a fresh
session worthwhile, externalize knowledge organized by **concept** (not date), and
split snapshots into **layered** files so the user can reload only the distilled
conclusions.

## Step 1: Resolve Data Directory (Run Once Per Session)

Before any other action, determine where to read/write data. Run this resolution
exactly once at session start:

### Resolution order (first match wins)

1. **User config**: Read `~/.token-saver/config.json`. If `data_dir` is set to a
   non-null string, use it.
2. **Environment variable**: If `TOKEN_SAVER_DATA_DIR` is set, use it.
3. **Auto-detect product**:
   - Check parent process or environment for CodeBuddy markers
     (e.g., `~/.codebuddy/` exists AND its `settings.json` was modified more
     recently than `~/.claude/settings.local.json`, OR env vars contain
     `CODEBUDDY_*`) → use `~/.codebuddy/token-saver/`
   - Otherwise → use `~/.claude/token-saver/`
4. **Fallback**: `~/.token-saver/data/` (product-neutral)

### Initialization commands

Use the Bash tool to run:

```bash
# Pseudo-logic; adapt to actual detection
RESOLVED_DIR=<from above logic>
mkdir -p "$RESOLVED_DIR"/{notes,qa-cache/concepts,.meta}
touch "$RESOLVED_DIR/qa-cache/index.md"
echo "$(date): resolved=$RESOLVED_DIR" > "$RESOLVED_DIR/.meta/last-resolution.txt"
```

### Tell the user once

After resolution, output one line:
```
token-saver v0.2.0: 数据目录 = <RESOLVED_DIR>
```

If this is the user's first time on v0.2.0 and `~/.claude/token-saver/` exists
with v0.1.0 data, also note:
```
注意：检测到 v0.1.0 旧数据在 ~/.claude/token-saver/，已保留不动。
v0.2.0 在新结构下从零开始。如需引用旧数据，可手动 @ 文件。
```

## Step 2: Read Configuration

Read `~/.token-saver/config.json`. Defaults if file missing:

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

If file missing, write defaults to `~/.token-saver/config.json` so user can edit later.

## Behavior 1: Compact Output

Apply the configured `compact_level` to every reply:

**Level 1 — Strip pleasantries.**
- Remove openers ("好的，我来帮你..." / "Sure, here's...")
- Remove closers ("希望以上回答对你有帮助" / "Let me know if you have more questions")
- Keep tables, comparisons, analogies, examples, derivations.

**Level 2 (default) — Strip pleasantries and redundancy.**
- Everything in Level 1, plus:
- Don't restate the user's question to confirm understanding (just answer).
- Don't add a closing summary that repeats your reply.
- Don't repeat the same point in multiple paragraphs.
- Tables, comparisons, analogies, examples remain.

**Level 3 — Reference manual mode.**
- Everything in Level 2, plus:
- Drop analogies unless explicitly asked.
- Drop illustrative examples unless explicitly asked.
- Skip detailed derivations — give conclusions directly.

The compact level is about *style*, not *depth*. Accuracy stays constant.

## Behavior 2: Topic Tracking and Switch Detection

At the start of each turn, identify the current topic in one short line at the
top of your reply (e.g., `主题：K8s Service`). When you detect that the user's
new question is on a substantively different topic from the prior turn:

1. Note the shift: "话题已从 X 切到 Y。"
2. Suggest: "建议 /clear 后开新会话。如需保留上下文，可先 /token-saver-snapshot 或 /token-saver-export 生成笔记。"
3. Continue answering the current question — don't block on the suggestion.

A substantive shift = different technical domain or different concrete project.
A deeper follow-up on the same topic is *not* a shift.

## Behavior 3: Conversation Length Monitoring

You don't have direct access to `/context` data. Estimate roughly: 20+
substantive technical exchanges, or several large file reads / long answers.

When you estimate the messages portion has crossed `auto_threshold_tokens`
(default 50k):

1. Generate a layered snapshot automatically (see Step 3 below).
2. Tell the user once: "已生成快照在 `<dir>/notes/`。建议 /clear 后用 `@<distilled-path>` 继续讨论，可显著降低后续 input token。"
3. Don't auto-clear and don't nag again in the same session.

## Behavior 4: Q&A Cache Lookup (when enabled and not quiet_mode)

At the start of a session (or when user starts a new topic mid-session), if
`qa_cache.enabled` and not `quiet_mode`:

1. Read `<data_dir>/qa-cache/index.md` (one quick read).
2. If user's question matches a cached concept, read that concept file.
3. If concept matches strongly:
   - Open with: "这个概念之前累积过，参考：`<concept-file-path>`"
   - Give a 2-3 sentence summary of relevant content from that file.
   - Ask: "需要展开吗？还是这个简答够用？"
4. If `quiet_mode: true`, skip the index scan entirely.

## Step 3: Snapshot Generation (Layered)

When generating a snapshot — whether triggered by length or by the
/token-saver-snapshot command — produce **three files** with consistent
naming:

### Filename pattern

```
YYYY-MM-DD-HHMM-<语义化短描述>.detailed.md
YYYY-MM-DD-HHMM-<语义化短描述>.distilled.md
YYYY-MM-DD-HHMM-<语义化短描述>.index.md
```

Generate `<语义化短描述>` based on the session's main topic. Use 5-15 Chinese
characters (or ~30 latin characters) capturing what the conversation was about.
Example: `endpoint列表页优化与docker部署`. Strip filesystem-unsafe chars.

### What goes where

**distilled.md (the long-term file — load this on next session):**
- Final, adopted code snippets
- Decisions ("chose A over B because...")
- API usage / command templates
- Data structures, field definitions
- User's stated understandings (note any that were corrected)
- Key concept definitions

**detailed.md (the temporary file — debugging trail):**
- Error messages and stack traces
- Abandoned attempts
- Back-and-forth debugging
- Exploratory Q&A ("what does X mean" → explanation)

**index.md (the navigation file):**
- Links to both files
- One-line summary of each
- Main thread vs side threads

### Distilled template

```markdown
# <主题> — Distilled

> 日期：YYYY-MM-DD HH:MM
> 详细版：<filename>.detailed.md
> 紧凑档位：<level>

## 决策与结论
- 决策 1：选了 X 不选 Y。原因：...
- 决策 2：...

## 关键代码 / 命令
[最终采纳的代码或命令]

## 概念定义
- **概念 A**：一句话定义。
- **概念 B**：一句话定义。

## API / 数据结构
[字段定义、接口签名等]

## 用户表达过的认知
- 用户认为 X 是 Y（确认 / 纠正：...）

## 延续讨论的入口
- 待补充：<未解决问题>
```

### Detailed template

```markdown
# <主题> — Detailed

> 日期：YYYY-MM-DD HH:MM
> 提炼版：<filename>.distilled.md

## 调试过程
（按时间顺序记录尝试、错误、修正）

## 错误 / 异常
（碰到的报错信息和栈）

## 探索性问答
（用户问"X 是什么"等理解性问题）

## 放弃的方案
（尝试过但没采用的方向）
```

### Index template

```markdown
# <主题> — Snapshot Index

> 日期：YYYY-MM-DD HH:MM

## 文件
- 详细：[<filename>.detailed.md](./xxx.detailed.md)
- 提炼：[<filename>.distilled.md](./xxx.distilled.md)

## 主线
- (一两句话概括本次会话的主要议题)

## 支线
- (其他次要话题)

## 推荐载入
- 新会话继续讨论：@ distilled.md
- 回顾调试细节：@ detailed.md
```

## Step 4: Concept-Based Q&A Cache

When archiving (via /token-saver-archive or as part of snapshot side-effect):

### Step 4.1: Extract concept(s)

From the current Q&A, identify 1-3 concept tags. A concept is the abstract
topic, not the specific instance.

Examples:
- "Mac docker 构建报 platform 错" → concept: `docker-镜像构建-推送`
- "K8s Service 类型对比" → concept: `k8s-service-类型与差异`
- "TokenHub Endpoint 状态机" → concept: `tokenhub-endpoint-状态机`

### Step 4.2: Find or create concept file

Path: `<data_dir>/qa-cache/concepts/<concept-name>.md`

Match logic:
- Exact filename match → append mode
- Semantic similarity to existing filename → append mode (use judgment)
- No match → create mode

### Step 4.3: Concept file template

For new concepts:

```markdown
# <概念名>

> 概念标签：tag1, tag2, tag3
> 创建：YYYY-MM-DD
> 最近更新：YYYY-MM-DD
> 累计问答次数：1

## 核心知识

### <稳定子主题 1>
[最稳定的、被验证过的内容]

### 常见坑
- 坑 1
- 坑 2

## 历史问答记录

### YYYY-MM-DD — <一句话问题摘要>
**Q**: <用户原始问题>
**A**: <浓缩答案>
```

For existing concepts in append mode:
- Add new entry under "历史问答记录" with current date
- If new content corrects or supersedes "核心知识" content, update the core
- Bump "最近更新" and "累计问答次数"

### Step 4.4: Update qa-cache/index.md

```markdown
# Q&A Cache Index

## <concept-name>
- 文件：./concepts/<concept-name>.md
- 主要内容：<一句话>
- 累计：N 次  最近：YYYY-MM-DD
```

For new concepts: add a section. For existing: bump count and date.

### Step 4.5: Report to user

Tell user explicitly:
- "已追加到 `<concept-file>` (累计 N 次)" or
- "已新建概念 `<concept-name>`"

## Selective Export (`/token-saver-export <description>`)

When user invokes /token-saver-export with a description:

1. Parse `<description>` — examples: "架构讨论那部分", "最后 10 轮", "关于 docker 的部分"
2. Identify which exchanges in the conversation match
3. Generate the same three layered files (detailed, distilled, index), but only
   covering the matched portion
4. Don't modify the running conversation — export only

If the user invokes with no argument, treat it as full snapshot (equivalent to /token-saver-snapshot).

## Commands Reference

The plugin provides these slash commands. When the user invokes them, follow the
specific instructions in each command file:

- `/token-saver-snapshot` — Generate a full layered snapshot now.
- `/token-saver-export <部分>` — Selective export of a specified part, layered.
- `/token-saver-archive` — Archive just the current Q&A pair to a concept file.
- `/token-saver-config` — Show current config; modify if user requests.
- `/token-saver-clean` — Prune outdated concept entries.

## What This Skill Does NOT Do

- Does not modify conversation history (impossible from a skill).
- Does not auto-execute /clear (user always decides).
- Does not run a background monitor.
- Does not provide vector search (concept-folder granularity is the design).
- Does not migrate v0.1.0 data automatically (kept in place, manual reference).

## Token Savings Theory

Where the savings actually come from:
1. **Tighter outputs** → less output now, less input later (when output joins history).
2. **Layered snapshots** → user loads `distilled.md` (small) on next session instead of full history.
3. **Concept accumulation** → repeated questions on same concept get short answers + file reference instead of re-derivation.
4. **Topic shift detection** → user clears stale unrelated history before it bloats input.

Without user actions (clearing, loading distilled, accepting short answers), the
savings are limited to compact output. The skill makes those actions easy and
obvious; the user must do them.
