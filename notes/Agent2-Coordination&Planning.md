# 🤖 About Agent - 规划、记忆与多智能体协作 (Planning, Memory & Multi-Agent)

Agent 要想处理真实世界中复杂的软件工程，光会调用工具是不够的。它还需要拥有“长短期的记忆力”、“拆解并追踪任务的能力”，以及在遇到海量上下文时“自我隔离和降维”的能力。

---

## 1. 外部工作记忆与动态干预 (Todo & Nagging)

大模型的上下文（Context Window）里塞满了各种报错和代码，它极易忘记自己“最初要干什么”。解决办法是：**在内存中开辟一块干净的地方，强制 AI 把计划写下来。**

### 1.1 严格的状态机约束 (TodoManager)

代码中引入的 `TodoManager` 不是简单的文本记录，而是一个具有强约束的**状态机**。

```python
class TodoManager:
    def update(self, items: list) -> str:
        # 约束 1：限制任务总数，防止 AI 发散思维陷入无底洞
        if len(items) > 20:
            raise ValueError("Max 20 todos allowed")
            
        in_progress_count = 0
        for item in items:
            status = str(item.get("status", "pending")).lower()
            # 约束 2：严格的状态枚举
            if status not in ("pending", "in_progress", "completed"):
                raise ValueError(f"Invalid status '{status}'")
            if status == "in_progress":
                in_progress_count += 1
                
        # 约束 3（极其重要）：强制 AI “单线程”工作，一次只能做一个任务
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress at a time")
            
        self.items = validated
        return self.render()
```

### 1.2 工程技巧：动态提示词注入-问责 (Nag Reminder)

即便有了 `todo` 工具，AI 在连续敲代码修 Bug 时，往往会忘记更新进度。这就需要底层系统像适时问责。

```python
def agent_loop(messages: list):
    rounds_since_todo = 0 # 记录 AI 多少轮没碰过进度表了
    while True:
        # ... 调用大模型 ...
        used_todo = any(b.type == "tool_use" and b.name == "todo" for b in response.content)
        
        rounds_since_todo = 0 if used_todo else rounds_since_todo + 1
        
        # 【关键点】如果连续 3 轮没有更新进度，强行在上下文开头注入提醒！
        if rounds_since_todo >= 3:
            results.insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```
*   **原理解析**：AI 刚刚执行完一条命令，系统在返回报错日志的同时，在它耳边大喊一句 `<reminder>`。AI 看到这句话，下一轮就会乖乖调用 `todo` 工具同步状态。这保证了长线任务**绝对不偏航**。

---

## 2. 上下文隔离与子智能体 (Subagents)

**核心洞察：“Process isolation gives context isolation for free.”（进程隔离免费带来了上下文隔离）**

当 Agent 需要大范围探索一个陌生项目时，不断的试错（ls, cat, 报错）会导致 Token 爆炸并污染主 Agent 的大脑。此时引入boss和员工的模式（Multi-Agent）。

### 2.1 隔离循环 (`run_subagent`)

子智能体（实习生）带着极其干净的上下文去独立工作，干完活只把结论汇报给主智能体（老板）。

```python
def run_subagent(prompt: str) -> str:
    # 1. 开局一张白纸：没有任何主 Agent 的冗长历史
    sub_messages =[{"role": "user", "content": prompt}]  
    
    # 2. 防死循环：实习生最多只能试错 30 次
    for _ in range(30):  
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        # ... 执行子工具 ...
        
        if response.stop_reason != "tool_use":
            break
            
    # 3. 阅后即焚：把包含大量废话的 sub_messages 丢弃，只返回最后一句总结给老板
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```

### 2.2 防御性工具分发

```python
# 实习生的工具包（干脏活累活）
CHILD_TOOLS =[bash, read_file, write_file, edit_file]

# 老板的工具包（多了一个派发任务的权力）
PARENT_TOOLS = CHILD_TOOLS + [task]
```
*   **原理解析**：绝不能给 `CHILD_TOOLS` 赋予 `task` 能力，否则子 Agent 遇到困难会再召唤子 Agent，导致无限产生。

---

## 3. 按需加载私有知识库 (Skill Loading)

**核心洞察：“Don't put everything in the system prompt. Load on demand.”**

如果你把各种内部 API 文档、前端开发规范全都塞进 System Prompt，既费钱又会让 AI “变笨”（Lost in the Middle 效应）。优雅的解法是：**双层注入架构**。

### 3.1 第一层：低成本 Meta 注入

扫描本地 `skills/` 目录下的 Markdown 文件，只提取 YAML 前言（Frontmatter）部分，塞入系统提示词。

```python
# 最终塞入大模型的 System Prompt 极度精简：
SYSTEM = f"""You are a coding agent at {WORKDIR}.
Skills available:
  - pdf: Process PDF files...[python, pdfminer]
  - code-review: Internal code review guidelines
"""
```

### 3.2 第二层：态按需加载

当大模型意识到需要处理 PDF 时，它主动调用 `load_skill("pdf")` 工具，此时底层代码才把几千字的正文传给它。

```python
class SkillLoader:
    def get_content(self, name: str) -> str:
        skill = self.skills.get(name)
        # 将几千字的完整教程包裹在 XML 标签中作为 tool_result 喂给模型
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
```
*   **原理解析**：相当一种极度轻量级的 RAG（检索增强生成）。把“全知全能的臃肿大脑”变成了“知道去哪查字典的聪明大脑”。

---

## 4. 持久化记忆与依赖图 (Persistent Tasks)

**核心洞察：“State that survives compression -- because it's outside the conversation.”**

前面的 `todo` 是活在内存里的，一旦关机重启就清零。为了处理跨越多天的超大型项目，Agent 必须将记忆外置到磁盘，并理解任务的先后顺序。

### 4.1 把状态存入文件系统 (JSON 持久化)

不再使用内存数组，而是为每一个子任务生成独立的 `.json` 文件存储在隐藏的 `.tasks/` 目录中。

```python
class TaskManager:
    def _save(self, task: dict):
        path = self.dir / f"task_{task['id']}.json"
        path.write_text(json.dumps(task, indent=2))
        
    def create(self, subject: str, description: str = "") -> str:
        task = {
            "id": self._next_id, "subject": subject, 
            "status": "pending", 
            "blockedBy": [], "blocks":[] # 核心设计：依赖图字段
        }
        self._save(task)
```

### 4.2 高级项目管理：依赖解析树 (Dependency Graph)

给任务加上 `blockedBy`（被阻塞）属性后，AI 就能进行逻辑严密的统筹规划（比如“必须先写好数据库，才能写前端 API”）。

更强大的是，当一个前置任务完成时，底层系统会自动“解锁”后续任务：

```python
    def _clear_dependency(self, completed_id: int):
        """当一个任务完成时，自动将其从其他所有任务的 blockedBy 列表中移除"""
        for f in self.dir.glob("task_*.json"):
            task = json.loads(f.read_text())
            if completed_id in task.get("blockedBy", []):
                task["blockedBy"].remove(completed_id) # 自动解除阻塞
                self._save(task)
```
*   **原理解析**：这极大地减轻了大模型的思考负担。AI 只要专注于眼下没有被 Block（阻塞）的任务，一旦标记完成，环境会自动刷新出新的可用任务，实现了真正的**自动化流水线**。

---

## 5. 核心 Takeaways

1. **Agent 的记忆本质上是状态管理 (State Management)**：不管是短期的 Todo 还是长期的 JSON Tasks，跳出 LLM 本身的上下文限制，把“状态”交给传统的确定性代码（Python）去管理，是让 Agent 变得可靠的唯一途径。
2. **好的 Prompt 应该在运行中动态注入 (Dynamic Prompting)**：不要只在一开始写一大段 System Prompt。根据 AI 的实时行为（如忘记写进度），在会话流中偷偷插入 `<reminder>`，效果远好于一开始的严厉警告。
3. **隔离产生效率 (Isolation)**：用 Subagent 去吸收脏日志，用按需加载去替代全文 Prompt。时刻保持主 Agent (Parent) 的上下文（Context Window）极度清爽，是降低成本、防止 AI 智商降级的秘密武器。