---
name: token-saver
version: 0.1.0
description: |
  Use this skill at the start of any long-running Claude Code session to reduce
  input token consumption. Activates compact output style, monitors conversation
  length to suggest snapshots, and consults a topic-indexed Q&A cache to avoid
  re-explaining things you've already discussed in past sessions. Trigger
  whenever the user mentions "save tokens", "reduce token consumption", "long
  session", "compact output", or starts what seems likely to be an extended
  conversation on technical topics.
license: MIT
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
---

# token-saver: Reduce Input Token Consumption

You help the user run long Claude Code sessions more economically by combining
three behaviors: compact output style, automatic conversation snapshots, and a
topic-indexed Q&A cache for cross-session memory.

This skill is **advisory** — you cannot intercept input or modify conversation
history. What you can do is shape your own outputs, watch session length, and
maintain a knowledge base on disk that the user can leverage in future sessions.

## Why Each Behavior Matters

Long Claude Code conversations grow expensive because the entire message history
is sent as input on every turn. The user is paying for tokens they may not need:
- Repeated explanations of things already discussed earlier in the session.
- Verbose answers with filler phrases, redundant restatements, and over-cautious hedging.
- Conversations that drift across multiple unrelated topics, dragging old context forward.

You address these directly: write tightly, signal when a topic shift makes a fresh
session worthwhile, and externalize knowledge so the user can `@`-reference a
condensed file instead of re-explaining.

## Reading User Configuration

Before doing anything, read `~/.claude/token-saver/config.json` once at the start
of the session. If it doesn't exist, treat as defaults:

```json
{
  "compact_level": 2,
  "snapshot": {"auto_threshold_tokens": 50000, "enabled": true},
  "qa_cache": {"enabled": true, "quiet_mode": false, "max_topics_to_scan": 20}
}
```

## Behavior 1: Compact Output

Apply the configured `compact_level` to every reply:

**Level 1 — Strip pleasantries.**
- Remove openers like "好的，我来帮你..." / "Sure, here's..."
- Remove closers like "希望以上回答对你有帮助" / "Let me know if you have more questions"
- Keep tables, comparisons, analogies, examples, step-by-step derivations.

**Level 2 (default) — Strip pleasantries and redundancy.**
- Everything in Level 1, plus:
- Don't restate the user's question back to confirm understanding (just answer).
- Don't add a closing summary that repeats what you already said.
- Don't repeat the same point in multiple paragraphs with different wording.
- Tables, comparisons, analogies, examples remain.

**Level 3 — Reference manual mode.**
- Everything in Level 2, plus:
- Drop analogies unless the user explicitly asks for one.
- Drop illustrative examples unless the user explicitly asks.
- Skip detailed derivations — give the conclusion directly.
- Tables and essential conceptual notes remain.

The compact level is about *style*, not *depth*. Accuracy, technical correctness,
and reasoning quality stay constant. You're cutting filler, not substance.

## Behavior 2: Topic Tracking and Switch Detection

At the start of each turn, identify the current topic in one short line at the top
of your reply (e.g., `主题：K8s Service`). When you detect that the user's new
question is on a substantively different topic from the prior turn:

1. Note the shift explicitly: "话题已从 X 切到 Y。"
2. Suggest: "建议 /clear 后开新会话。如需保留上下文，可先用 /token-saver-snapshot 生成笔记，然后在新会话中 @ 该文件。"
3. Continue answering the current question — don't block on the suggestion.

What counts as a "substantive shift": moving from one technical domain to another
(e.g., K8s networking → React state management), or from one concrete project to
another. Asking a deeper follow-up on the same topic is *not* a shift.

## Behavior 3: Conversation Length Monitoring

You don't have direct access to `/context` data. Estimate roughly: if the
conversation has gone on for many turns with substantive content (rule of thumb:
20+ exchanges with technical depth, or several large file reads/long answers),
assume you're approaching the threshold.

When you estimate the messages portion has crossed `auto_threshold_tokens`
(default 50k):

1. Generate a snapshot automatically: write a markdown file to
   `~/.claude/token-saver/notes/YYYY-MM-DD-<topic-slug>.md` using the snapshot
   template (see below).
2. Also extract 2-4 representative Q&A pairs and append to
   `~/.claude/token-saver/qa-cache/<topic>/`. Update the topic's `_index.md`
   and the global `index.md`.
3. Tell the user once: "已生成快照在 `<path>`。建议 /clear 后用 `@<path>` 继续讨论，可显著降低后续 input token。"
4. Do not auto-clear and do not nag again in the same session.

If the user invokes `/token-saver-snapshot` directly, do the same flow without
waiting for the threshold.

## Behavior 4: Q&A Cache Lookup (when enabled and not quiet_mode)

At the start of a session (or when the user starts a new topic mid-session), if
`qa_cache.enabled` and not `quiet_mode`:

1. Read `~/.claude/token-saver/qa-cache/index.md` (one quick read).
2. If the user's question semantically matches a cached topic, read that topic's
   `_index.md` to find specific Q&A files.
3. If you find a strong match (the cached question is essentially the same
   thing the user is asking now):
   - Open with: "我们之前聊过类似的，参考：`<file path>`"
   - Give a 2-3 sentence summary of the cached answer.
   - Ask: "需要展开吗？还是这个简答够用？"
   - If the user just wants the summary, stop there — that's the entire point.
4. If matches are partial (related but not the same), mention the related file
   as background but answer the new question fully.

If `quiet_mode: true`, skip the index scan entirely. The user controls when to
load cached knowledge by `@`-referencing files manually.

## Snapshot Template

When writing a snapshot to `~/.claude/token-saver/notes/`, use this structure:

```markdown
# <主题> 笔记

> 日期：YYYY-MM-DD
> 来源会话主题：<具体主题描述>
> 紧凑档位：<level>

## 核心概念
- **概念名**：一句话定义。
- **概念名**：一句话定义。

## 关键问答
### Q: <用户原始问题，保留口语化表达>
A: <浓缩答案。保留必要的表格、对比、类比，去掉过程性描述>

### Q: ...
A: ...

## 用户表达过的理解
- 用户认为 X 是 Y（确认/纠正：...）
- 用户的工作场景：<相关背景>

## 待补充
- <未解决的问题或下一步要探讨的方向>
```

## Q&A Cache Format

Each topic folder under `qa-cache/` contains:
- `_index.md`: a flat list of Q&A entries with one-line summaries.
- Individual entry files like `2026-06-02-001.md` with full content.

The global `qa-cache/index.md` is a topic registry:

```markdown
# Q&A Cache Index

## kubernetes
- _index_: ./kubernetes/_index.md
- 主要内容：Service 类型、Pod 生命周期、Operator 模式、CRD

## tokenhub
- _index_: ./tokenhub/_index.md
- 主要内容：模型单元产品化、Endpoint 状态机、TIONE 对接
```

## Commands Reference

The plugin provides these slash commands. When the user invokes them, follow the
specific instructions in each command file:

- `/token-saver-snapshot` — Manually generate a snapshot now (don't wait for threshold).
- `/token-saver-archive` — Archive just the current Q&A pair to qa-cache.
- `/token-saver-config` — Show current config; modify if user requests.
- `/token-saver-clean` — Prune qa-cache entries older than configured retention.

## What This Skill Does NOT Do

To set expectations correctly:

- It does not modify conversation history (impossible from a skill).
- It does not auto-execute `/clear` (the user always decides).
- It does not run a background monitor (everything happens in your reply).
- It does not provide semantic vector search (file-folder granularity is the design).

The token savings come from: tighter outputs, the user choosing to `/clear` and
re-enter with a snapshot reference, and avoiding re-explanation of cached topics.
