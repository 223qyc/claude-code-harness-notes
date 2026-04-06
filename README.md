# Claude Code Harness Notes

> 🤖 学习记录：Claude Code agent harness 工程。从核心循环到多智能体协作，深入理解「模型是 agent，代码是 harness」的设计哲学。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/Claude-Code-orange.svg)](https://claude.ai/code)

---

## 📖 关于本项目

本仓库是对 GitHub 项目 **[shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)** 的深入学习记录。

**learn-claude-code** 是一个教学型项目，通过 12 个渐进式会话 (s01-s12) 演示如何构建一个完整的 Agent Harness。本仓库记录了我对该项目的学习总结、代码分析和实践心得。

### 核心理念

> **The model is the agent. The code is the harness.**
>
> 模型是智能体，代码是载体。

Agent 的智能来自于训练好的模型（Claude、GPT 等），而我们工程师的工作是构建 **Harness**——让模型能够感知环境、调用工具、管理记忆、执行任务的载体系统。

---

## 📚 学习笔记总览

本仓库包含 5 篇学习笔记，覆盖了 Claude Code 的核心架构：

| 笔记 | 主题 | 对应会话 |
|:----:|------|:--------:|
| [Agent 1](./notes/Agent1-Tools%20&%20Execution.md) | 工具与执行 | s01-s02 |
| [Agent 2](./notes/Agent2-Coordination%26Planning.md) | 规划、记忆与多智能体 | s03-s07 |
| [Agent 3](./notes/Agent3-Context%20Compression.md) | 上下文压缩 | s06 |
| [Agent 4](./notes/Agent4-Concurrency%20%26%20Asynchronous%20Tasks.md) | 并发与异步执行 | s08 |
| [Agent 5](./notes/Agent5-Collaboration.md) | 多智能体协作与工作流隔离 | s09-s12 |

---

## 📝 笔记内容详解

### [Agent 1 - 工具与执行](./notes/Agent1-Tools%20&%20Execution.md)

**核心内容**：Agent 与传统 Chatbot 的根本区别——持续运转的循环与真实世界的交互。

| 章节 | 核心概念 |
|------|----------|
| Agent Loop | `while + stop_reason` 的核心循环，实现感知-思考-行动闭环 |
| Tool Dispatch | 路由字典 `TOOL_HANDLERS`，优雅扩展工具而不修改循环逻辑 |
| JSON Schema | 工具的 API 文档，`description` 和 `required` 的设计原则 |
| 沙盒防御 | `safe_path()` 路径穿越防护，防止 Agent 逃逸工作目录 |
| 文件流 | `edit_file` 的精准替换机制，Cursor/Claude Code 的底层改代码策略 |

**核心代码模式**：
```python
def agent_loop(messages):
    while True:
        response = client.messages.create(model, system, messages, tools)
        if response.stop_reason != "tool_use":
            return
        results = [execute(tool) for tool in response.content]
        messages.append({"role": "user", "content": results})
```

---

### [Agent 2 - 规划、记忆与多智能体](./notes/Agent2-Coordination%26Planning.md)

**核心内容**：Agent 如何拥有"记忆力"、拆解任务、以及在复杂上下文中保持清醒。

| 章节 | 核心概念 |
|------|----------|
| TodoWrite | `TodoManager` 状态机，强制单线程工作，防止 AI 发散思维 |
| Nag Reminder | 动态提示词注入，连续 3 轮不更新进度时强行问责 |
| Subagents | 子 Agent 隔离循环，"阅后即焚"，只返回总结给主 Agent |
| Skill Loading | 双层注入架构，Meta 注入 + 按需加载，轻量级 RAG |
| Persistent Tasks | JSON 持久化 + 依赖图 `blockedBy`，自动化流水线 |

**关键洞察**：
> 隔离产生效率。用 Subagent 吸收脏日志，用按需加载替代全文 Prompt，时刻保持主 Agent 上下文清爽。

---

### [Agent 3 - 上下文压缩](./notes/Agent3-Context%20Compression.md)

**核心内容**：突破 LLM 物理窗口限制，实现"无限运行的永动机 Agent"。

| 层级 | 机制 | 触发时机 |
|------|------|----------|
| Layer 1 | Micro-Compact | 每次思考前静默执行，折叠历史工具输出 |
| Layer 2 | Auto-Compact | Token 超阈值时自动触发，大模型总结自己 |
| Layer 3 | Manual-Compact | AI 主动调用 `compact` 工具，清空大脑迎接新任务 |

**记忆分层架构**：
```
工作记忆 (Working Memory)  → 当前 messages[]
提炼记忆 (Summary)         → 压缩后的状态描述
外部档案 (Transcripts)     → 硬盘里的原始日志，可供回溯
```

---

### [Agent 4 - 并发与异步执行](./notes/Agent4-Concurrency%20%26%20Asynchronous%20Tasks.md)

**核心内容**：从串行到并行，Agent 在耗时任务时不再阻塞死等。

| 章节 | 核心概念 |
|------|----------|
| BackgroundManager | 后台线程执行耗时命令，瞬间返回 task_id |
| Notification Queue | 线程安全的信箱机制，跨线程通信 |
| Drain & Inject | 查信箱 + 伪装 User 消息，动态注入后台结果 |

**异步打断示意**：
```
AI 改文件1 → npm install 后台启动 → AI 改文件2 → npm 完成，插播通知 → AI 改文件3
```

---

### [Agent 5 - 多智能体协作与工作流隔离](./notes/Agent5-Collaboration.md)

**核心内容**：构建"AI 团队"，多个大模型持久化生存、异步交流、自主认领任务。

| 章节 | 核心概念 |
|------|----------|
| Agent Teams | JSONL 文件信箱，追加写入实现异步消息总线 |
| Team Protocols | `request_id` 关联模式，关机/计划审批的握手协议 |
| Autonomous Agents | IDLE 轮询抢单，拉取模式（Pull Model）自己找活干 |
| Worktree Isolation | Git worktree 物理沙盒，控制面与执行面绑定 |

**控制面 vs 执行面**：
```
控制面 (Control Plane)  → .tasks/ 目录，管理逻辑任务
执行面 (Execution Plane) → .worktrees/ 目录，管理物理代码沙盒
```

---

## 🎯 项目结构

```
claude-code-harness-notes/
├── README.md                                    # 项目说明（本文件）
├── LICENSE                                      # MIT 许可
└── notes/                                       # 学习笔记
    ├── Agent1-Tools & Execution.md              # 工具与执行
    ├── Agent2-Coordination&Planning.md          # 规划、记忆与多智能体
    ├── Agent3-Context Compression.md            # 上下文压缩
    ├── Agent4-Concurrency & Asynchronous Tasks.md # 并发与异步执行
    └── Agent5-Collaboration.md                  # 多智能体协作与工作流隔离
```

---

## 📊 学习进度

| 会话 | 主题 | 笔记覆盖 | 状态 |
|:----:|------|:--------:|:----:|
| s01 | The Agent Loop | Agent 1 | ✅ |
| s02 | Tool Use | Agent 1 | ✅ |
| s03 | TodoWrite | Agent 2 | ✅ |
| s04 | Subagents | Agent 2 | ✅ |
| s05 | Skill Loading | Agent 2 | ✅ |
| s06 | Context Compact | Agent 3 | ✅ |
| s07 | Task System | Agent 2 | ✅ |
| s08 | Background Tasks | Agent 4 | ✅ |
| s09 | Agent Teams | Agent 5 | ✅ |
| s10 | Team Protocols | Agent 5 | ✅ |
| s11 | Autonomous Agents | Agent 5 | ✅ |
| s12 | Worktree Isolation | Agent 5 | ✅ |

**12 个会话已全部学习完成！**

---

## 🚀 后续计划

1. **代码实践** - 基于学到的 harness 模式，实现自己的 agent 项目
2. **代码重写** - 用不同的方式重新实现核心机制，加深理解
3. **扩展阅读** - 探索其他 agent 框架（LangChain、CrewAI）并对比设计差异

---

## 🛠️ 技术栈

| 技术 | 用途 |
|------|------|
| Python | Agent harness 核心实现 |
| Anthropic Claude API | 大模型调用 |
| threading | 后台任务并发执行 |
| Git Worktree | 多 Agent 代码沙盒隔离 |
| JSONL | 消息总线持久化 |

---

## 📦 快速开始

如果你想跟随我的学习路径：

```bash
# 1. 克隆原项目
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code

# 2. 安装依赖
pip install -r requirements.txt

# 3. 配置环境变量
cp .env.example .env  # 设置 ANTHROPIC_API_KEY

# 4. 从最简单的循环开始
python agents/s01_agent_loop.py

# 5. 对照本仓库的笔记学习
```

---

## 🔗 相关资源

| 资源 | 链接 |
|------|------|
| 原项目 | [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) |
| Claude Code 官方 | [claude.ai/code](https://claude.ai/code) |
| Anthropic API 文档 | [docs.anthropic.com](https://docs.anthropic.com/claude/docs) |
| Kode Agent CLI | [shareAI-lab/Kode-cli](https://github.com/shareAI-lab/Kode-cli) |

---

## 📝 许可

本项目采用 [MIT 许可](./LICENSE)。

笔记中引用的代码片段均来自 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 项目，遵循其 MIT 许可。

---

<div align="center">

**The model is the agent. The code is the harness.**
**Build great harnesses. The agent will do the rest.**

</div>