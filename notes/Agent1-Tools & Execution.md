# 🤖About Agent - 工具与执行（Tools & Execution）

Agent与传统Chatbot的核心区别，在于其拥有一个**持续运转的Agent Loop**，并且能够通过**Tools**与真实的物理世界（文件系统、终端）进行交互。

**学习总结基于github的project“learn claude code”,总结中使用的所有代码也来自该仓库。**

---

## 1. Agent 的灵魂：核心循环 (The Agent Loop)

普通大模型是“一问一答”的单次生成（One-shot），而 Agent 是一个**“感知 -> 思考 -> 行动 -> 观察” (Observe-Think-Act Loop)** 的循环。

### 1.1 核心代码实现

在不到30行代码就可实现了一个最小可用的Agent核心循环：

```python
def agent_loop(messages: list):
    while True: 
        # 1. 思考：大模型根据当前上下文决定下一步行动
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        
        # 2. 记录：将 AI 的思考和动作指令追加到对话历史中
        messages.append({"role": "assistant", "content": response.content})
        
        # 3. 停止条件：如果 AI 觉得任务完成了，没有调用任何工具，就跳出循环
        if response.stop_reason != "tool_use":
            return
            
        # 4. 行动与观察：提取工具调用指令，执行真实代码，并将结果喂回给大模型
        results =[]
        for block in response.content:
            if block.type == "tool_use":
                print(f"\033[33m$ {block.input['command']}\033[0m")
                output = run_bash(block.input["command"]) # 真实执行
                
                # 按照严格的格式规范，组装执行结果
                results.append({
                    "type": "tool_result", 
                    "tool_use_id": block.id,
                    "content": output
                })
                
        # 5. 闭环：伪装成用户的口吻，把真实世界的执行结果反馈给大模型，继续下一轮 while 循环
        messages.append({"role": "user", "content": results})
```

### 1.2 核心

*   **`stop_reason` (停止原因)**：这是判断Agent是否继续运转的核心标志。只有当大模型返回 `"tool_use"` 时，说明它觉得还需要干活；一旦它返回 `"end_turn"`，说明它认为任务已彻底完成。
*   **标准的格式接力**：大模型的上下文（Context）有严格的规范。
    *   **Assistant 消息**：包含工具调用的名字和参数。
    *   **User 消息**：必须包含 `tool_result` 和对应的 `tool_use_id`。这就好比AI递出一张工单（ID=123），外层代码干完活后，必须拿着带有ID=123的回执单交还给AI，AI才能对得上号。

---

## 2. 工具路由 (Tool Dispatch)

**“Agent 的循环逻辑永远不用变，变强的唯一方式是给它加工具。”**

当工具从1个变成几十个时，则不应该在代码里写满 `if-else`，而应该引入**路由分发机制**。

### 2.1 路由字典的实现

在 `s02_tool_use.py` 中，使用了一个字典（Dictionary）来映射工具：

```python
# -- The dispatch map: {tool_name: handler} --
TOOL_HANDLERS = {
    "bash":       lambda **kw: run_bash(kw["command"]),
    "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
    "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
    "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"], kw["new_text"]),
}
```

### 2.2 执行层的动态派发

```python
results =[]
for block in response.content:
    if block.type == "tool_use":
        # 1. 查表获取对应的函数
        handler = TOOL_HANDLERS.get(block.name)
        
        # 2. **block.input 是 Python 的神级操作，直接将 JSON 对象解包为函数的关键字参数
        output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
        
        results.append({"type": "tool_result", "tool_use_id": block.id, "content": output})
```

### 2.3 核心知识点解析

*   **`lambda **kw` 与参数解包**：大模型生成的工具参数是一个 JSON 对象（在 Python 中被解析为字典 `block.input`）。通过 `**kw` 语法，我们可以在不改变底层调用的情况下，完美适配各种有着不同参数的工具函数。
*   **容错设计**：`if handler else "Unknown tool"`。大模型存在“幻觉”，偶尔会编造不存在的工具。与其让 Python 抛出异常导致整个程序崩溃，不如返回一个报错信息给大模型，大模型看到后会“自我纠错”，重新调用正确的工具。

---

## 3. 定义规则：JSON Schema

工具不仅需要 Python 代码实现，还需要给大模型提供一本“API 接口文档”，让它知道怎么用。

```python
TOOLS =[
    {"name": "bash", "description": "Run a shell command.",
     "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
    
    {"name": "edit_file", "description": "Replace exact text in file.",
     "input_schema": {
         "type": "object", 
         "properties": {
             "path": {"type": "string"}, 
             "old_text": {"type": "string"}, 
             "new_text": {"type": "string"}
         }, 
         "required":["path", "old_text", "new_text"]
     }},
]
```

*   **设计原则**：大模型极度依赖 `description`。描述越清晰，AI 使用工具的时机就越准确。`required` 字段强制约束了 AI 必须提供哪些变量，这大大降低了运行报错的概率。

---

## 4. 工程化细节：沙盒防御与文件流

真实的生产级 Agent 必须具备极强的鲁棒性（Robustness）和安全性。有了 `bash` 为什么还要单独写文件工具？原因在于**大模型的 Bash 能力在处理复杂长文本、特殊转义符时极易翻车**。

### 4.1 电子围栏（Sandboxing）

```python
WORKDIR = Path.cwd()

def safe_path(p: str) -> Path:
    path = (WORKDIR / p).resolve()
    # 核心防御：防止大模型使用 "../../etc/passwd" 路径穿越技术攻击系统
    if not path.is_relative_to(WORKDIR):
        raise ValueError(f"Path escapes workspace: {p}")
    return path
```

### 4.2 文件读写

*   **`read_file` 的防爆显存设计 (Truncation)**
    如果 AI 尝试读取一个几百 MB 的日志文件，大模型的 Token 会瞬间超限。
    *实现逻辑*：加入 `limit` 参数，一旦超过行数，强行截断并提示 `... (XXX more lines)`。
    
*   **`write_file` 的自动建目录机制**
    ```python
    fp.parent.mkdir(parents=True, exist_ok=True)
    fp.write_text(content)
    ```
    *实现逻辑*：当 AI 想要在 `src/utils/math.py` 写代码时，即便目录不存在，Python 也会帮它连带创建父级目录，避免 AI 陷入“找不到目录 -> 报错 -> 放弃”的死循环。

*   **`edit_file` 的精准替换机制 (Search-and-Replace)**
    ```python
    # 极其高明的设计：只替换变动部分
    fp.write_text(content.replace(old_text, new_text, 1))
    ```
    *实现逻辑*：不要求大模型重写整个文件（浪费 Token 且容易丢代码），而是要求它提供“待替换的旧字符串”和“新字符串”。系统使用 `replace` 实现精准无损修改。这是目前诸如 Cursor、Claude Code 等顶尖工具都在使用的底层改代码策略。

---

## 5. Takeaways

1.  **System Prompt 决定工作态度**：`"Act, don't explain."`（少废话多干活）是 Coding Agent 极其重要的提示词，它省去了不必要的寒暄，节省了时间和 Token 成本。
2.  **闭环反馈是智能的来源**：Agent 的聪明不是因为模型本身有多强大，而是因为它能在出错后，看到 `Error Log` 并**自我纠错**。
3.  **大一统架构**：无论是查询天气、操作数据库还是浏览网页，对于现代 Agent 来说，无非就是在 `TOOLS` 数组里多加一个 JSON 配置，在 `TOOL_HANDLERS` 里多加一行路由。核心的 `agent_loop` 稳如磐石。