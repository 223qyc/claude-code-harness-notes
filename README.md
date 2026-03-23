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

---

## 📚 学习笔记目录

| 序号 | 主题 | 核心内容 | 对应会话 |
|:----:|------|----------|:--------:|
| 1 | [工具与执行](./notes/Agent1-Tools%20&%20Execution.md) | Agent Loop、Tool Dispatch、JSON Schema、沙盒防御 | s01-s02 |
| 2 | [规划、记忆与多智能体协作](./notes/Agent2-Coordination%26Planning.md) | TodoWrite、Subagents、Skill Loading、Persistent Tasks | s03-s07 |
| 3 | [上下文压缩](./notes/Agent3-Context%20Compression.md) | Micro-Compact、Auto-Compact、Manual-Compact | s06 |

### 待完成

- [ ] s08 后台任务 (Background Tasks)
- [ ] s09 智能体团队 (Agent Teams)
- [ ] s10 团队协议 (Team Protocols)
- [ ] s11 自主智能体 (Autonomous Agents)
- [ ] s12 工作树隔离 (Worktree Task Isolation)

---

## 🎯 项目结构

```
claude-code-harness-notes/
├── README.md                           # 项目说明
├── LICENSE                             # MIT 许可
└── notes/                              # 学习笔记
    ├── Agent1-Tools & Execution.md     # 工具与执行
    ├── Agent2-Coordination&Planning.md # 规划、记忆与多智能体
    └── Agent3-Context Compression.md   # 上下文压缩
```

---

## 🚀 后续计划

1. **完成剩余会话学习** - 继续学习 s08-s12 的内容
2. **代码实践** - 基于学到的 harness 模式，实现自己的 agent 项目
3. **代码重写** - 用不同的方式重新实现核心机制，加深理解

---

## 🛠️ 技术栈

- **Python** - Agent harness 实现
- **Anthropic Claude API** - 大模型调用
- **Next.js** - 原项目的 Web 平台

---

## 📦 快速开始

如果你想跟随我的学习路径，可以先克隆原项目：

```bash
git clone https://github.com/shareAI-lab/learn-claude-code
cd learn-claude-code
pip install -r requirements.txt
```

然后阅读本仓库的笔记，对照原项目的 `agents/` 目录中的代码进行学习。

---

## 🔗 相关资源

- [原项目：learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- [Claude Code 官方文档](https://claude.ai/code)
- [Anthropic API 文档](https://docs.anthropic.com/claude/docs)

---

## 📝 许可

本项目采用 [MIT 许可](./LICENSE)。

---

<div align="center">

**Build great harnesses. The agent will do the rest.**

</div>
