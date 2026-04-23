# Hermes 记忆系统整改报告
**日期：** 2026-04-23  
**范围：** AgentMemory MCP 功能盘点 + Hermes 记忆系统整改前后对比

---

## 一、AgentMemory MCP 功能盘点

### 接入时间
2026-04-21（昨天），通过 `node .../cli.mjs mcp` 启动，作为 mcp_servers 接入 Hermes config.yaml。

### 可用工具（8个）

| 工具 | 功能 | 状态 |
|------|------|------|
| `memory_recall` | 语义检索历史 session/observations | ✅ 正常 |
| `memory_save` | 保存新记忆/observation | ✅ 正常 |
| `memory_smart_search` | 混合语义+关键词搜索 | ✅ 正常 |
| `memory_sessions` | 查询会话记录列表 | ✅ 正常 |
| `prompts/list` | 列出可用 prompts | ❌ MCP 端点未实现 |
| `get_prompt` | 获取指定 prompt | ❌ MCP 端点未实现 |
| `resources/list` | 列出可用 resources | ❌ MCP 端点未实现 |
| `read_resource` | 读取指定 resource | ❌ MCP 端点未实现 |

### AgentMemory 新增能力（相对原来 Hermes 记忆系统）

| 能力 | 原来 Hermes | AgentMemory MCP |
|------|------------|-----------------|
| 存储后端 | MEMORY.md 纯文本 | SQLite 向量存储 |
| 检索方式 | 关键词搜索（grep） | 语义向量检索 |
| 跨 session 持久化 | 仅 MEMORY.md 参与 session 恢复 | 独立运行，跨 session 存续 |
| 搜索粒度 | 整文件级别 | observation 级别 |
| Session 历史 | 无原生支持 | `memory_sessions` 查询 |

### 关键限制
- 存储在 SQLite，不可人类直接读取
- 无版本控制、无质量层、无变更日志
- `update_memory_type` 通过 MCP 不 work（需绕道 WAL 过滤）
- MCP 协议实现不完整（prompts/resources 未实现），但核心 `memory_*` 工具正常

---

## 二、整改前 Hermes 记忆系统架构

```
MEMORY.md ←---------------→ AgentMemory MCP
  (参与 session 恢复)         (独立运行，不参与 session 恢复)
      ↑                               ↑
  trim_memory.py              8个 MCP tools
  (外部压缩脚本)                   
  纯时间评分                      
  无事务安全                      
  无 WAL                          
  无 DAG                          
```

### 组件清单（整改前）

| 组件 | 位置 | 作用 |
|------|------|------|
| MEMORY.md | ~/.hermes/memories/ | 主要记忆存储，参与 session 恢复 |
| trim_memory.py | ~/.hermes/scripts/ | 外部压缩脚本，阈值 2100/目标 1900 |
| library.md | ~/.hermes/memories/ | MEMORY.md 条目归档库 |
| AgentMemory MCP | 连接状态，8个工具 | 独立运行的向量存储 |
| WAL | ❌ 不存在 | — |
| DAG | ❌ 不存在 | — |

### 核心痛点（整改前）
1. **短期失忆**：进程崩溃或压缩失败时，AI 丢上下文
2. **两套系统脱节**：AgentMemory 和 MEMORY.md 完全不互通
3. **无事务安全**：压缩失败直接丢数据，无回滚
4. **评分单一**：仅按时间（7天/14天/30天），可能误伤有意义的条目
5. **压缩粒度粗**：整文件级别压缩，无法保留语义结构

---

## 三、整改后 Hermes 记忆系统架构

```
MEMORY.md ←--------→ AgentMemory MCP
  (参与 session 恢复)     (DAG节点存储)
          ↑                     ↑
          │              agentmemory_store.py
          │                     ↑
      Hermes          dag_context.py (DAG管理器)
     MemoryStore            ↑
          ↑                wal.py (WAL日志)
          ↑                 ↑
    session 恢复 ←←←←←←←←←←←┘
          ↑
    WAL replay（崩溃恢复）
```

### 今日新增文件

| 文件 | 位置 | 作用 |
|------|------|------|
| `wal.py` | ~/.hermes/ | WAL 核心：append/commit/replay/checkpoint/rollback |
| `dag_context.py` | ~/.hermes/ | DAG + WAL 集成：节点管理、压缩、上下文构建 |
| `agentmemory_store.py` | ~/.hermes/ | MCP store 包装：stdio JSON-RPC，线程安全 |
| `dag/` | ~/.hermes/dag/ | DAG 数据目录（checkpoints） |
| `wal/` | ~/.hermes/wal/ | WAL 日志目录 |

### 新增能力清单

| 能力 | 实现 | 说明 |
|------|------|------|
| WAL 事务日志 | wal.py | pending→committed→归档 三状态机 |
| 崩溃恢复 | wal replay | 重启后从 MCP 恢复 committed，pending 重试 |
| DAG 节点管理 | dag_context.py | D0（原始消息）/D1（摘要）/D2（全局摘要）分层 |
| 节点级压缩 | D0 叶子可压缩 | 语义叶子合并为 D1，保护头部 |
| 压缩回滚 | WAL rollback marker | 压缩失败时恢复到压缩前状态 |
| MCP 集成 | agentmemory_store.py | 所有 DAG 节点存 AgentMemory MCP |
| 上下文窗口构建 | build_context_window() | WAL 过滤已归档 D0，保留 D1+尾巴 |
| 混合评分归档 | trim_memory.py | 时间评分 + 关联度 boost |
| 自动归档 Cronjob | 每日 07:00/19:00 | score<100 候选归档，保留 identity/protected |

### 保留的原有组件

| 组件 | 变化 |
|------|------|
| MEMORY.md | 不变，仍参与 session 恢复（但不再被 DAG 直接操作） |
| trim_memory.py | 升级了评分算法（+关联度 boost），仍每日运行 |
| library.md | 归档库，包含今日归档的第一条（Hippocampus-her） |
| AgentMemory MCP 连接 | 保留，作为 DAG 节点存储后端 |

---

## 四、关键设计决策

### 1. WAL 先行，压缩后行
```
append → WAL(pending) → MCP(committed) → WAL(mark)
compress → WAL(rollback) → compress → WAL(commit)
```

### 2. MCP store 绕道 `update_memory_type`
MCP 的 `update_memory_type` 不 work，所以：
- D0 节点压缩后 → WAL 标记为 `archived_d0`
- `build_context_window()` 查 WAL 过滤已归档节点，不查 MCP type

### 3. `_generate_summary` 占位符
D0→D1 压缩时调用 LLM 生成摘要，目前是 `not implemented` 状态。
下一步接入 LLM provider（确认用哪个：openrouter-free 或其他）。

### 4. MEMORY.md 和 DAG 双轨并存
- MEMORY.md：Hermes 核心记忆（identity、偏好、行为准则）
- DAG：会话上下文管理（消息、D0-D2 节点、WAL）

两者通过 `session 恢复` 机制汇合：DAG 从 MCP 重建，MEMORY.md 参与 session 恢复。

---

## 五、待完成事项

| 事项 | 状态 | 说明 |
|------|------|------|
| `_generate_summary` LLM 接入 | ⏳ 待做 | 需确认 provider（建议用 openrouter-free） |
| `memory_dag_sync.py` 整合 | ⏳ 待做 | 目前是独立脚本，未与 DAG 联动 |
| D2 全局摘要层 | 📋 规划 | D0→D1→D2 三层，当前只有 D0/D1 |
| AgentMemory MCP `update_memory_type` | ❌ 不 work | 等待上游修复或自行 patch |

---

## 六、总结

**AgentMemory MCP 的贡献：**
- 向量语义搜索、跨 session 持久化、独立存储后端

**今日整改的贡献：**
- WAL 事务安全 + 崩溃恢复
- DAG 节点级压缩（保护头部，放开尾部）
- MCP 与 Hermes MemoryStore 连通（原来完全脱节）
- 混合评分归档（时间 + 关联度）

**一句话总结：**
> 原来 AgentMemory 和 MEMORY.md 是两个各干各的，今天让它们通过 WAL+DAG 连成了同一套记忆系统。
