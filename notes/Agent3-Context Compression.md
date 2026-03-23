# 🤖 About Agent - 上下文压缩与无限记忆循环 (Context Compression)

Agent 在执行复杂任务时，每一次调用 `bash` 打印的日志、每一次 `read_file` 读入的代码，都会消耗大模型的上下文窗口（Context Window）。如果不加以干预，Agent 很快就会变慢、变贵，甚至因为超出 Token 上限而直接崩溃。

**核心洞察：“The agent can forget strategically and keep working forever.”（Agent 可以通过策略性地遗忘，来永远工作下去。）**

---

## 1. 三层压缩管道架构 (Three-Layer Compression Pipeline)

为了实现“无限会话”，引入了一个极其精妙的 **三层防御体系**。这三层压缩机制相互配合，在保留 Agent 思考连贯性的同时，极致地压榨了冗余 Token。

### 1.1 第一层：静默的微压缩 (Layer 1: Micro-Compact)

**触发时机：每次大模型思考前静默执行。**
大模型真正需要关注的，通常只有**最近几次**的工具执行结果。几十步之前长达几千字的报错日志，现在已经毫无用处了。

#### 核心代码实现

```python
KEEP_RECENT = 3

def micro_compact(messages: list) -> list:
    # ... (收集所有的 tool_result 及其对应的工具名称) ...
    
    # 核心逻辑：只保留最近的 KEEP_RECENT（3次）的工具返回结果
    to_clear = tool_results[:-KEEP_RECENT]
    
    for _, _, result in to_clear:
        # 如果历史工具的输出大于 100 个字符，就把它“折叠”起来
        if isinstance(result.get("content"), str) and len(result["content"]) > 100:
            tool_id = result.get("tool_use_id", "")
            tool_name = tool_name_map.get(tool_id, "unknown")
            
            # 【神级替换】把几千字的日志，替换成极其简短的一句话！
            result["content"] = f"[Previous: used {tool_name}]"
            
    return messages
```

*   **原理解析**：为什么不直接删掉历史记录，而是替换为 `"[Previous: used {tool_name}]"`？
    因为我们需要保留 AI 的**“思维轨迹（Trajectory）”**。如果直接删除，AI 会忘记自己曾经尝试过什么。通过留下这个“脚印”，AI 会知道：“哦，我之前调用过 `read_file` 和 `bash`”，但又不需要承担阅读冗长返回结果的 Token 成本。

---

### 1.2 第二层：触顶自动总结 (Layer 2: Auto-Compact)

**触发时机：当全局 Token 估算超过安全阈值（如 50000）时自动触发。**
当第一层的微压缩也救不了场（比如对话实在太长了），系统会直接掀桌子，对整个对话历史进行“降维打击”。

#### 核心代码实现

```python
def auto_compact(messages: list) -> list:
    # 步骤 1：为了防止永久丢失信息，把完整的原始对话写进本地硬盘的 transcript (剧本) 日志里
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        # ... 保存 JSONL ...
        
    # 步骤 2：呼叫大模型自己来总结自己！
    conversation_text = json.dumps(messages, default=str)[:80000]
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity. Include: "
            "1) What was accomplished, 2) Current state, 3) Key decisions made. "
            "Be concise but preserve critical details.\n\n" + conversation_text}],
        max_tokens=2000,
    )
    summary = response.content[0].text
    
    # 步骤 3：极其暴力的上下文替换！将原本可能有 100 条的 messages 瞬间变成 2 条
    return [
        {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
    ]
```

*   **原理解析**：
    *   **落盘备份**：AI 可能会在总结中遗漏细节，所以先把原始记录存入 `.transcripts/`。
    *   **特定 Prompt**：强制要求大模型总结三个核心要素：“完成了什么(Accomplished)、当前状态(Current state)、关键决策(Key decisions)”。
    *   **无缝衔接**：替换后的上下文伪装成了 User 和 Assistant 的一次对话，助手回答 `"Understood..."` 保证了后续生成的连贯性。

---

### 1.3 第三层：赋予 AI 主动遗忘的能力 (Layer 3: Manual-Compact)

**触发时机：AI 觉得自己需要清空大脑时。**
这是最体现 Agent “自主性”的一个设计。系统不仅被动监控 Token，还把压缩能力封装成了一个工具 `compact` 交给 AI 自己掌控。

#### 核心代码实现

```python
TOOLS =[
    # ... bash, read, write 等 ...
    {
        "name": "compact", 
        "description": "Trigger manual conversation compression.",
        "input_schema": {"type": "object", "properties": {"focus": {"type": "string", "description": "What to preserve in the summary"}}}
    },
]

# 在 agent_loop 中拦截：
if manual_compact:
    print("[manual compact]")
    messages[:] = auto_compact(messages) # 触发全局替换
```

*   **原理解析**：当 AI 完成了一个巨大的里程碑（比如“数据库搭建完毕，接下来准备写前端”），聪明的模型会意识到前面的建表 SQL 日志对接下来写前端毫无帮助。此时它会主动调用 `compact` 工具，主动要求系统清空并总结它的上下文，轻装上阵迎接下一个子任务。

---

## 2. 引擎底层的拦截与运转逻辑

这三层逻辑是如何在核心死循环 `agent_loop` 中串联起来的？

```python
def agent_loop(messages: list):
    while True:
        # 1. 每次思考前，先悄悄把历史冗长输出折叠掉 (Layer 1)
        micro_compact(messages)
        
        # 2. 估算 Token，如果快爆了，强制执行总结 (Layer 2)
        if estimate_tokens(messages) > THRESHOLD:
            print("[auto_compact triggered]")
            messages[:] = auto_compact(messages) # 注意这里用 messages[:] 原地修改列表
            
        # 3. 呼叫大模型
        response = client.messages.create(...)
        
        # 4. 执行工具逻辑
        # ... 
        
        # 5. 如果 AI 刚才调用了 compact 工具，在这个回合结束时触发压缩 (Layer 3)
        if manual_compact:
            print("[manual compact]")
            messages[:] = auto_compact(messages)
```

---

## 3. 核心 Takeaways

1. **上下文管理是省钱和提智的核心**：大模型有一种“Lost in the Middle（迷失在中间）”的缺陷。把无用的日志及时清理，不仅能省下巨额的 API Token 费用，还能让 AI 的注意力更加集中，大幅降低犯错率。
2. **保留结构，抹去细节**：在微压缩（Micro-Compact）中，把大段日志替换成 `[Previous: used bash]` 是神来之笔。Agent 不需要记住过去每一步看到了什么，只要记住“我走过这条路”就足够了。
3. **记忆是可以分层的**：
   * **工作记忆 (Working Memory)**：当前的 `messages`。
   * **提炼记忆 (Summary)**：经过压缩后的状态描述。
   * **外部档案 (Transcripts)**：存在硬盘里的原始日志，可供人类回溯审计或未来做 RAG（检索）。
   这种架构突破了 LLM 物理窗口的限制，使得 **“无限运行的永动机 Agent”** 成为可能。