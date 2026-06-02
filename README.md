# token-saver

> Reduce input token consumption in long Claude Code sessions.

## 这个插件做什么

在 Claude Code 的长会话中，每次提问都要把整个对话历史作为 input 发给模型。这个插件通过三个机制降低 input token：

1. **紧凑表达**：让 Claude 输出去掉客套话、复述、无效总结
2. **会话快照**：对话过长时自动浓缩成主题分区笔记，方便 /clear 后用 @ 文件继续
3. **问答档案**：跨会话保留关键 Q&A，下次问相似问题时直接引用而非重新回答

## 这个插件不做什么

- 不修改对话历史（skill 没这个能力）
- 不自动 /clear（决策权交给你）
- 不引入"语义向量检索"（按主题文件夹粒度，简单可靠）
- 不跨设备同步（纯本地文件）

## 安装

```bash
# 通过本地 marketplace 安装（如果尚未配置）
claude plugin marketplace add ~/.claude/local-marketplace
claude plugin install token-saver@local-plugins
```

## 使用

### 启用 skill
新会话开始时，可以在 prompt 里说 "启用 token-saver" 或 "开启紧凑模式"。
也可以在 ~/.claude/CLAUDE.md 里写一条偏好让它默认启用。

### 命令

| 命令 | 用途 |
|------|------|
| `/token-saver-snapshot` | 立即生成当前会话的快照笔记 |
| `/token-saver-archive` | 把当前轮的 Q&A 单独归档 |
| `/token-saver-config` | 查看/修改配置 |
| `/token-saver-clean` | 清理过期的 qa-cache 条目 |

### 配置

编辑 `~/.claude/token-saver/config.json`：

```json
{
  "compact_level": 2,
  "snapshot": {
    "auto_threshold_tokens": 50000,
    "enabled": true
  },
  "qa_cache": {
    "enabled": true,
    "quiet_mode": false,
    "max_topics_to_scan": 20
  }
}
```

| 字段 | 默认 | 说明 |
|------|------|------|
| `compact_level` | 2 | 1=只去客套；2=去客套+去重复；3=极简 |
| `snapshot.auto_threshold_tokens` | 50000 | 估算 messages 超过此值时自动快照 |
| `snapshot.enabled` | true | 是否启用自动快照 |
| `qa_cache.enabled` | true | 是否使用问答档案 |
| `qa_cache.quiet_mode` | false | true 时不主动扫描索引 |
| `qa_cache.max_topics_to_scan` | 20 | 扫描时最多检查的主题数 |

## 数据存放位置

```
~/.claude/token-saver/
├── config.json          # 用户配置
├── notes/               # 会话快照
│   └── YYYY-MM-DD-<topic>.md
└── qa-cache/            # 问答档案
    ├── index.md         # 主题清单
    └── <topic>/
        ├── _index.md
        └── YYYY-MM-DD-NNN.md
```

## 典型工作流

### 学习场景（如 K8s 学习）

1. 新会话开始 → "启用 token-saver"
2. 长时间讨论 → 自动触发快照（~50k tokens 时）
3. /clear → `@~/.claude/token-saver/notes/2026-06-02-k8s.md` 继续
4. 后续问相关问题 → Claude 引用 qa-cache 给简答

### 项目设计场景（如 TokenHub）

1. 启用 token-saver
2. 设计讨论结束 → /token-saver-snapshot
3. 跨会话 → /token-saver-archive 归档关键决策
4. 月度清理 → /token-saver-clean

## 限制

- 行为约束是软性的，Claude 在某些场景下可能偏离
- 阈值估算不精确（skill 没法读 /context 的真实数据）
- qa-cache 主题粒度可能过粗，命中后不够精确
- 紧凑档位 3 在概念学习场景下会显著降低体验，谨慎使用

## 设计文档

完整设计见 `~/Documents/work doc/chanchu/specs/2026-06-02-token-saver-design.md`。
