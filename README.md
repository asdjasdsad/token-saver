# token-saver

> Reduce input token consumption in long Claude Code / CodeBuddy sessions.

[![version](https://img.shields.io/badge/version-0.2.0-blue)]()
[![license](https://img.shields.io/badge/license-MIT-green)]()

## 这个插件做什么

在 Claude Code 和 CodeBuddy 的长会话中，每次提问都要把整个对话历史作为 input 发给模型。这个插件通过四个机制降低 input token：

1. **紧凑表达**：让 Claude 输出去掉客套话、复述、无效总结
2. **分层快照**：自动浓缩长对话为 detailed / distilled / index 三层文件，下次会话只载入 distilled
3. **选择性导出**：用户可以只导出对话的特定部分，不影响当前会话
4. **概念累积型知识库**：跨会话 Q&A 按"概念"组织，同一概念多次讨论累积到同一文件

## v0.2.0 新特性

- ✨ **环境感知**：自动识别 Claude Code 还是 CodeBuddy，写入对应产品目录
- ✨ **语义化文件名**：notes 和 concepts 都用可读文件名，不再是 `2026-06-02-001.md`
- ✨ **分层快照**：每次快照产出三文件，distilled 用于跨会话载入，detailed 用于回顾调试
- ✨ **选择性导出 `/token-saver-export`**：只导出会话的指定部分
- ✨ **概念累积**：qa-cache 不再按时间平铺，按概念聚合，同主题多次问答合并到一个文件

## 这个插件不做什么

- 不修改对话历史（skill 没这个能力）
- 不自动 /clear（决策权交给你）
- 不引入"语义向量检索"（按概念文件夹粒度，简单可靠）
- 不跨设备同步（纯本地文件）
- 不自动迁移 v0.1.0 数据（保留原位，可手动 @ 引用）

## 安装

### 从 GitHub 直接安装（最快）

```bash
# 1. 添加 marketplace
claude plugin marketplace add asdjasdsad/token-saver

# 2. 安装插件
claude plugin install token-saver@token-saver-marketplace
```

### 从 community marketplace 安装（审核通过后）

```bash
claude plugin marketplace add anthropics/claude-plugins-community
claude plugin install token-saver@claude-community
```

## 使用

### 启用 skill

新会话开始时，可以在 prompt 里说 "启用 token-saver" 或 "开启紧凑模式"。
也可以在 `~/.claude/CLAUDE.md` 或 `~/.codebuddy/CLAUDE.md` 里写一条偏好让它默认启用。

### 命令

| 命令 | 用途 |
|------|------|
| `/token-saver-snapshot` | 立即生成当前会话的分层快照 |
| `/token-saver-export <部分>` | 选择性导出对话的某部分 |
| `/token-saver-archive` | 把当前轮 Q&A 按概念归档 |
| `/token-saver-config` | 查看/修改配置 |
| `/token-saver-clean` | 清理过期的概念条目 |

### 配置

编辑 `~/.token-saver/config.json`（v0.2 起从 `~/.claude/token-saver/` 移到此处）：

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

| 字段 | 默认 | 说明 |
|------|------|------|
| `data_dir` | null | null 时自动检测产品；显式字符串覆盖 |
| `compact_level` | 2 | 1=只去客套；2=去客套+去重复；3=极简 |
| `snapshot.auto_threshold_tokens` | 50000 | 估算 messages 超过此值时自动快照 |
| `snapshot.enabled` | true | 是否启用自动快照 |
| `snapshot.layered` | true | 是否分层（detailed + distilled + index） |
| `qa_cache.enabled` | true | 是否使用概念知识库 |
| `qa_cache.quiet_mode` | false | true 时不主动扫描 index |
| `qa_cache.max_concepts_to_scan` | 30 | 扫描时最多检查的概念数 |
| `naming.use_chinese` | true | 文件名是否允许中文 |
| `naming.max_filename_length` | 60 | 文件名长度上限 |

## 数据目录解析

```
1. config.json 的 data_dir 字段（显式覆盖）
2. 环境变量 TOKEN_SAVER_DATA_DIR
3. 自动检测产品：
   - 在 CodeBuddy 里 → ~/.codebuddy/token-saver/
   - 在 Claude Code 里 → ~/.claude/token-saver/
4. 兜底：~/.token-saver/data/
```

会话开始时 Claude 会打印实际使用的目录：
```
token-saver v0.2.0: 数据目录 = /Users/you/.codebuddy/token-saver
```

## 数据目录结构

```
<data_dir>/
├── notes/                                          ← 会话快照（分层）
│   ├── 2026-06-02-1547-架构选型与pd分离决策.distilled.md
│   ├── 2026-06-02-1547-架构选型与pd分离决策.detailed.md
│   └── 2026-06-02-1547-架构选型与pd分离决策.index.md
└── qa-cache/
    ├── index.md                                    ← 概念清单
    └── concepts/
        ├── docker-镜像构建-推送.md                  ← 概念累积，多次讨论合并
        ├── k8s-service-类型与差异.md
        └── ...
```

## 典型工作流

### 学习场景（如 K8s 学习）

1. 新会话开始 → "启用 token-saver"
2. 长时间讨论 → 自动触发分层快照（~50k tokens 时）
3. /clear → 用 `@<distilled-path>` 继续
4. 后续问相关概念 → Claude 引用 qa-cache/concepts/ 给简答

### 项目设计场景（如 TokenHub）

1. 启用 token-saver
2. 设计讨论结束 → /token-saver-snapshot
3. 临时归档关键决策 → /token-saver-archive
4. 只导出某次讨论 → /token-saver-export 架构那部分
5. 月度清理 → /token-saver-clean

## v0.1 → v0.2 升级注意

- 配置文件位置变更：`~/.claude/token-saver/config.json` → `~/.token-saver/config.json`
- 数据目录可能变更：取决于你在哪个产品里用（CodeBuddy 用户的数据现在写到 `~/.codebuddy/token-saver/`）
- v0.1 的旧数据保留在 `~/.claude/token-saver/`，不自动迁移
- 如需引用旧数据，可以手动 @ 旧文件
- v0.2 的笔记文件不再是 `2026-06-02-tokenhub.md`，而是 `2026-06-02-1547-tokenhub具体描述.md`（带时间戳和语义描述）
- v0.2 的 qa-cache 由"按主题文件夹+按日期编号"改为"按概念聚合"

## 限制

- 行为约束是软性的，Claude 在某些场景下可能偏离
- 阈值估算不精确（skill 没法读 /context 的真实数据）
- 自动检测产品可能误判（CodeBuddy 没有标识性环境变量），可以用 data_dir 显式覆盖
- 概念归档可能误归（不同主题被归到同一概念），用户可手动编辑或 clean
- 跨设备 data_dir 不一致（v0.2 不解决，v0.3 考虑）

## 设计文档

- v0.1 设计：`docs/specs/2026-06-02-token-saver-design.md`
- v0.2 设计：`docs/specs/2026-06-02-token-saver-v0.2-design.md`

## 反馈与贡献

- Issues: https://github.com/asdjasdsad/token-saver/issues
- 此插件正在持续打磨中。v0.3 计划加入：
  - 调用前预筛（提问时自动检索 qa-cache，命中则短答）
  - 主动 GC（长会话定期主动建议精简）
