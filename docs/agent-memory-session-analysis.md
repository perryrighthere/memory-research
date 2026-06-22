# Agent 框架 Memory 与 Session 管理调研

本调研面向当前仓库中的三个项目：OpenClaw、OpenHarness/ohmo（用户描述为 OpenHarmony，仓库内对应目录为 `OpenHarness`）和 Hermes Agent。结论基于三个并行 subagent 对源码和文档的只读检查，并在主线程复核关键文件后整理。

## 总览对比

| 项目 | Memory 定位 | Session 定位 | 持久化主体 | 与 agent run 的关系 |
| --- | --- | --- | --- | --- |
| OpenClaw | 插件化长期记忆，典型实现为 bundled `memory-lancedb`；另有文档化的 Markdown/SQLite memory search 体系 | 核心 conversation/runtime 状态，区分 session key、session id、metadata store 和 JSONL transcript | session metadata `sessions.json` + JSONL transcript；memory 默认 LanceDB 或 per-agent SQLite index | memory 在 prompt build 前召回并 prepend context，agent end 后自动捕获；session transcript 保存 conversation tree |
| OpenHarness/ohmo | Markdown project/personal memory，`MEMORY.md` 作索引；支持 relevant memory、auto-extract、auto-dream | 每轮保存 session snapshot；gateway 维持每 chat/thread 一个 runtime bundle | `~/.openharness/data/...` 或 `~/.ohmo/...` 下的 memory/session JSON/Markdown 文件 | runtime system prompt 注入 memory；每轮保存 messages snapshot；session memory checkpoint 辅助 compact |
| Hermes Agent | 内置 `MEMORY.md`/`USER.md` 精选长期记忆，外部 memory provider 可扩展 | SQLite `SessionDB` 是完整 session/message 历史与检索源 | `$HERMES_HOME/memories/*` + `$HERMES_HOME/state.db` | 内置 memory 在 agent 初始化冻结为 system prompt；外部 provider 每轮 prefetch/sync；SessionDB 每轮持久化消息 |

## OpenClaw

### Session 管理

OpenClaw 的 Session 是核心 conversation 状态，不只是聊天路由。它显式区分：

- `sessionKey`：路由/registry key，例如 agent main、channel peer、group/topic 等。
- `sessionId`：某次实际 transcript/run 的 id，可因 reset、freshness、rotation 改变。
- session metadata store：`sessions.json`，保存会话级 metadata、状态、插件扩展字段、工具策略、模型 override、compaction/checkpoint 等。
- JSONL transcript：由 `SessionManager` 维护，是 append-only conversation tree，不是简单线性日志。

`SessionEntry` 中保存 `sessionId`、`updatedAt`、`sessionFile`、父子 session、spawn depth、插件扩展、next-turn injections、模型/推理/工具/usage/compaction/memory flush 等大量会话状态。`SessionManager` 的 transcript entry 带 `id`、`parentId`，当前 LLM 上下文从 leaf 回溯到 root，因此分支/fork 不会改写历史，而是选择当前 branch 作为上下文。

默认 session 持久化路径由 agent id 决定：`<stateDir>/agents/<agentId>/sessions/sessions.json` 与同目录下的 `*.jsonl` transcript。agent run 后，`updateSessionStoreAfterAgentRun()` 将 active `sessionFile`、usage、model/provider、context tokens、compaction 等 run metadata 写回 session store。

### Memory 管理

OpenClaw 的长期 memory 主要是插件化能力。subagent 重点核查了 bundled `extensions/memory-lancedb`：

- `MemoryEntry` 包含 `id`、`text`、`vector`、`importance`、`category`、`createdAt`。
- 分类包括 `preference`、`fact`、`decision`、`entity`、`other`。
- 默认 LanceDB 路径是 `~/.openclaw/memory/lancedb`，也可通过插件 `dbPath` 配置覆盖。
- `MemoryDB` 懒初始化 LanceDB 表，`store()` 写入 vector row，`search()` 走 vector search 并按 score 过滤，`delete()` 先校验 UUID 再删除。
- 工具层提供 `memory_recall`、`memory_store`、`memory_forget`。

Memory 与 session transcript 是分离的：

1. `before_prompt_build`：如果 auto recall 启用，插件用当前 prompt/latest user text 做 embedding search，把相关 memory 作为 prepend context 注入本轮 prompt；这不写入 transcript。
2. `agent_end`：成功 run 后，如果 auto capture 启用，插件从 event messages 中抽取值得持久化的 user facts/preferences/decisions，embedding、查重后写入 LanceDB。
3. `session_end`：清理进程内 auto-capture cursor，不删除长期 memory。

此外，OpenClaw 文档层还有 Markdown memory 体系：`MEMORY.md`、`memory/YYYY-MM-DD.md`、`DREAMS.md`，以及默认 builtin memory engine 将 `MEMORY.md` 和 `memory/*.md` chunks 索引到 owning agent SQLite DB（`agents/<agentId>/agent/openclaw-agent.sqlite`），用于 keyword/vector/hybrid search。

### 关键理解

OpenClaw 的 session 是“conversation 内容和运行状态的真实账本”；memory 是“跨 session 可召回的长期知识层”。Memory recall 影响 prompt 上下文，但不是 session transcript；memory capture 从 run messages 中提炼事实，但不改变已存在 transcript。

## OpenHarness / ohmo

> 注：用户称 OpenHarmony；仓库中未发现 `OpenHarmony` 目录，相关项目是 `OpenHarness` 与其 personal-agent app `ohmo`。

### Memory 管理

OpenHarness/ohmo 存在三层 memory/session 概念，容易混淆：

1. **Durable Project/Personal Memory**：跨 session 长期记忆，Markdown 文件存储，`MEMORY.md` 是索引入口。OpenHarness 默认以 `~/.openharness/data/memory/<project-name>-<sha1>/` 组织项目记忆；ohmo 则用 `~/.ohmo/memory/` 或显式 `OHMO_WORKSPACE` 下的 `memory/`。
2. **Session Snapshot**：完整 conversation messages、system prompt、model、usage、tool metadata 的 JSON 快照，用于 continue/resume。
3. **File-backed Session Memory**：为 auto-compact/长对话压缩服务的 per-session Markdown checkpoint，包含当前目标、下一步、已验证工作、活跃 artifacts、最近对话摘要；它不是 durable memory。

ohmo 的 memory helper 将每条 personal memory 写成 Markdown 文件并更新 `memory/MEMORY.md` 索引。`add_memory_entry()` 使用 `.memory.lock` 文件锁、计算 signature 去重、写 frontmatter（schema version、id、name、description、type/category/importance/source/signature/time/ttl/disabled 等），重复则更新现有文件。`remove_memory_entry()` 是软删除：设置 `disabled=True` 并从 index 移除行。

runtime system prompt 会把 memory directory、`MEMORY.md` 以及相关 memory 文件内容注入 prompt。OpenHarness 的 generic project memory 还支持 usage index、relevant memory selection、auto-extract（默认关闭）和 auto-dream（默认关闭）。

### Session 管理

OpenHarness generic runtime 用 session snapshots 做 conversation 恢复：保存 `latest.json` 和 `session-<id>.json`，payload 包含 `session_id`、`cwd`、`model`、`system_prompt`、`messages`、`usage`、白名单 `tool_metadata`、`summary`、`message_count` 等。

ohmo 覆盖了 session backend，使用 `.ohmo/sessions`：

- `latest.json`：全局最近 session。
- `latest-<sessionKeyHash>.json`：每个 gateway chat/thread session 的最近 snapshot。
- `session-<sid>.json`：按 session id 保存的快照。

`OhmoSessionRuntimePool` 在 gateway 中维护 `session_key -> RuntimeBundle`：

- 如果同一 `session_key` 且 cwd 未变，复用已有 bundle，仅刷新 system prompt。
- 如果 cwd 变化，关闭旧 bundle 并重建。
- 如果没有 bundle，先按 `session_key` 加载最近 snapshot，恢复 messages 和 tool metadata，再 build/start runtime。
- 每轮处理后 `_save_snapshot()` 将当前 engine messages、usage、system prompt、tool metadata 保存到 `OhmoSessionBackend`。

### 关键理解

OpenHarness/ohmo 的 durable memory 是 Markdown 知识库，session 是 JSON snapshot 恢复机制；session memory checkpoint 是 compact continuity，不是长期 memory。ohmo gateway 还额外引入了 `session_key` 粒度的 runtime pool，使每个 chat/thread 有独立 bundle 和独立最新快照。

## Hermes Agent

### Memory 管理

Hermes 的内置 memory 是“小而精选”的长期记忆：

- `MEMORY.md`：agent personal notes / environment facts / project conventions。
- `USER.md`：用户画像、偏好、沟通风格、工作习惯。
- 路径是 `$HERMES_HOME/memories/`，通过 `get_hermes_home()` 动态解析，因此支持 profiles。
- 文件内 entries 用 `§` delimiter 分隔。

`MemoryStore` 同时维护两套状态：

1. `_system_prompt_snapshot`：`load_from_disk()` 时捕获，进入 system prompt，整个 session 内冻结不变。
2. live state：`memory_entries` / `user_entries`，tool 写入后立即落盘，但不会改变当前 session 的 system prompt。

这正是 Hermes prompt caching 设计的一部分：会话中途不重建系统提示、不改变历史上下文、不破坏 prefix cache。`MemoryStore.load_from_disk()` 会扫描 threat patterns，危险 entry 在 system prompt snapshot 中替换成 `[BLOCKED: ...]` 占位符，但 live state 保留原文，便于用户读取并删除。

Hermes 还有外部 memory provider 架构：

- `MemoryProvider` 定义 `initialize()`、`system_prompt_block()`、`prefetch()`、`sync_turn()`、tools、`shutdown()` 等生命周期。
- `MemoryManager` 是统一编排点：只允许一个 external provider，负责 tool schema 注入、每轮 prefetch、每轮结束 sync，以及后台单 worker 顺序写入。
- provider 初始化会拿到 `hermes_home`、`session_id`、platform、agent identity、gateway session key、profile 等上下文。

### Session 管理

Hermes 的核心 session store 是 SQLite `SessionDB`，默认路径 `$HERMES_HOME/state.db`。schema 包括：

- `sessions`：session metadata、source、user_id、model/config、system_prompt、parent_session_id、started/ended、end_reason、message/tool/token/cost counters、cwd、title、handoff、rewind/archive 等。
- `messages`：session_id、role、content、tool calls、tool name、timestamp、token、reasoning/codex fields、platform message id、observed、active 等。
- `messages_fts` 与 trigram FTS5：全文检索和 CJK/substring search。
- `compression_locks`：防止多个 agent 对同一 session 同时 compression 造成 split race。

`AIAgent.run_conversation()` 是每条用户消息一轮的入口。典型流程：

1. caller（CLI/gateway）持有 `conversation_history` 和 `session_id`。
2. turn context 复制 history，append 当前 user message。
3. 若 cached system prompt 不存在，构建一次；内置 memory snapshot 在这里进入 prompt。
4. 在模型调用前先 `_persist_session()`，使入站 user turn crash-resilient。
5. 外部 memory provider `prefetch_all()` 的召回上下文被拼到当前 API user message；它不 mutate `messages`，不会进入 session persistence。
6. turn finalizer 结束时再次 `_persist_session()`，把完整 turn 写 JSON log 与 SQLite，并返回更新后的 `messages` 和 `session_id` 给 caller。
7. 正常 turn 后 `_sync_external_memory_for_turn()` 异步同步当前 turn 到 external providers；interrupted turn 跳过，避免污染 memory。

生命周期边界很重要：`run_conversation()` 每条消息都会调用，所以 `on_session_end()` / `shutdown_all()` 不在每 turn 调用；真正 session boundary（CLI 退出、`/new`、`/reset`、gateway session expire、session id rotate 等）才调用 `shutdown_memory_provider()` 或 `commit_memory_session()`。

### 关键理解

Hermes 把 memory 和 session 分得很清楚：memory 是精选、冻结注入的长期上下文；session 是完整消息历史和搜索/恢复账本。外部 provider 的 prefetch 是 API-call-time 临时注入，不改变持久 transcript；sync turn 是每轮结束后的异步长期记忆更新。

## 横向结论

1. **三者都把 Memory 与 Session 分层。** Memory 面向跨 session 的 durable knowledge；Session 面向当前/历史 conversation 和运行状态。
2. **Memory 注入时机差异很大。** OpenClaw 在 plugin `before_prompt_build` prepend；OpenHarness 在 runtime system prompt 中加载 project/personal memory；Hermes 内置 memory 在 agent 初始化时冻结为 system prompt，外部 provider 则 per-turn API-only 注入。
3. **Session 持久化模型不同。** OpenClaw 是 metadata store + JSONL conversation tree；OpenHarness/ohmo 是 JSON snapshot；Hermes 是 SQLite sessions/messages + FTS，另有 JSON log。
4. **压缩/长上下文处理都与 session 相关，但不等同于 memory。** OpenHarness 的 session memory checkpoint 是 compact continuity；Hermes compression 会 split session 并使用 locks；OpenClaw session entry 有 compaction checkpoints 和 transcript tree 分支。
5. **自动提取 memory 都谨慎。** OpenClaw memory-lancedb 有 autoCapture cursor 和过滤/查重；OpenHarness auto-extract/auto-dream 默认关闭；Hermes external providers 每轮 sync 但内置 memory 仍需 tool 明确写入且不改变当前 prompt snapshot。

## 调研命令记录

- `pwd && find .. -name AGENTS.md -print`
- `git status --short`
- `rg -n "memory|session|Memory|Session|conversation|thread" openclaw/src openclaw/docs OpenHarness/ohmo OpenHarness/src/openharness hermes-agent/agent hermes-agent/tools hermes-agent/gateway hermes-agent/hermes_cli`
- `rg --files openclaw/src | rg 'session|memory|state|store|transcript'`
- `sed -n` / `nl -ba` 查看 OpenClaw、OpenHarness/ohmo、Hermes Agent 的关键源码范围。
