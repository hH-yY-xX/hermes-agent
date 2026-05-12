# Hermes Agent 记忆系统总结

Hermes Agent 拥有一个多层架构的记忆系统，主要包含以下模块：

---

## 1. 内置 Curated 记忆（MEMORY.md / USER.md）

**核心文件：** `tools/memory_tool.py`

- 使用两个 Markdown 文件存储：
  - **MEMORY.md** — Agent 个人笔记（环境事实、项目规范、工具经验、教训）
  - **USER.md** — 用户画像（偏好、沟通风格、工作习惯）
- 存储路径：`~/.hermes/memories/`，条目用 `\n§\n` 分隔
- 字符上限：MEMORY.md 约 2200 字符，USER.md 约 1375 字符
- **MemoryStore 类** 维护双状态：
  - `_system_prompt_snapshot`：会话开始时冻结，注入系统提示（保持前缀缓存稳定）
  - `memory_entries` / `user_entries`：实时可变状态，工具调用后立即持久化
- 操作：`add`、`replace`、`remove`
- 使用文件锁（`fcntl`）+ 原子写入（temp file + `os.replace`）保证并发安全
- 包含内容扫描，阻止提示注入和机密外泄模式

---

## 2. 可插拔 Memory Provider 系统

**核心文件：** `agent/memory_provider.py`、`agent/memory_manager.py`

- **MemoryProvider** 抽象基类定义生命周期：`initialize()`、`prefetch()`、`sync_turn()`、`handle_tool_call()`、`shutdown()` 等
- **MemoryManager** 编排器同时只允许 **一个外部 provider**（加上内置的共两个）
- 内置插件（8 个）：
  - Honcho、Mem0、Hindsight、SuperMemory、Holographic、ByteRover、OpenViking、RetainDB
- 插件发现机制：扫描 `plugins/memory/<name>/`（内置）和 `$HERMES_HOME/plugins/<name>/`（用户安装）

---

## 3. SQLite 会话状态存储（对话历史）

**核心文件：** `hermes_state.py`

- SQLite 数据库位于 `~/.hermes/state.db`，WAL 模式
- 表结构：
  - **sessions**：会话元数据（ID、模型、token 计数、成本、标题等）
  - **messages**：完整消息历史（角色、内容、工具调用、token 计数等）
  - **state_meta**：键值对元数据
  - **messages_fts** + **messages_fts_trigram**：FTS5 全文搜索（支持 CJK 三字词搜索）
- 支持会话分支（`parent_session_id`）
- 写竞争处理：BEGIN IMMEDIATE + 随机退避重试（15 次）

---

## 4. 上下文引擎（上下文窗口压缩）

**核心文件：** `agent/context_compressor.py`、`agent/context_engine.py`

- **ContextCompressor** 算法：
  1. 裁剪旧的工具结果（去重、摘要、截断）
  2. 保护头部消息（系统提示 + 前 N 轮对话）
  3. 按 token 预算保护尾部消息（最近上下文）
  4. 用辅助 LLM 对中间轮次进行结构化摘要
  5. 后续压缩时迭代更新摘要
- 摘要包含：活跃任务、目标、约束、已完成操作、关键决策、待处理问题等
- 防抖动：如果最近 2 次压缩节省 <10% 则跳过
- 摘要失败冷却期（无 provider 600s，临时错误 30-60s）

---

## 5. Checkpoint Manager（文件系统快照）

**核心文件：** `tools/checkpoint_manager.py`

- 基于 git 的裸仓库快照系统，存储在 `~/.hermes/checkpoints/store/`
- 每项目隔离（git refs + 独立索引）
- 在文件写操作前自动快照
- 每轮去重（每目录每轮一次快照）
- 自动维护：清理孤立项目、容量上限（默认 500MB）、保留期（默认 7 天）

---

## 6. 记忆与 Agent 生命周期的交互

| 生命周期节点 | 记忆操作 |
|---|---|
| Agent 初始化 | 加载 MemoryStore；初始化外部 provider |
| 构建系统提示 | 注入冻结的记忆快照 + provider 的 memory block |
| 每轮开始 | `on_turn_start()`；预取所有记忆 |
| 每轮结束 | `sync_all()` + 排队下次预取 |
| 记忆工具调用 | 写入 MEMORY.md/USER.md，通知外部 provider |
| 上下文压缩 | `on_pre_compress()` 保留洞察 |
| 会话切换 | `on_session_switch()` 旋转 session_id |
| 会话结束 | `on_session_end()` + `shutdown_all()` |

---

## 关键文件速查

| 文件 | 职责 |
|------|------|
| `tools/memory_tool.py` | 内置 MEMORY.md/USER.md 管理 |
| `agent/memory_provider.py` | 外部 Memory Provider 抽象基类 |
| `agent/memory_manager.py` | Memory Manager 编排器 |
| `agent/context_compressor.py` | 上下文窗口压缩 |
| `agent/context_references.py` | `@file`/`@folder`/`@git`/`@url` 引用展开 |
| `hermes_state.py` | SQLite 会话存储 + 全文搜索 |
| `tools/checkpoint_manager.py` | Git 文件系统快照 |
| `plugins/memory/*/` | 8 个外部记忆 provider 插件 |
