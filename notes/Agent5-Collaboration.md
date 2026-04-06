# 🤖 About Agent - 多智能体协作与工作流隔离 (Multi-Agent Collaboration)

单独的 Agent 无论模型多强大，都会受限于单一的上下文窗口和串行执行的速度。当遇到需要开发全栈项目时，真正的生产级方案是构建**“AI队列（Agent Teams）”**。

最更高效的 Agent 框架（如 Devin、CrewAI 高级版）的底层是：如何让多个大模型在同一个电脑上**持久化生存、异步交流、遵循审批纪律、自主认领任务，并在物理隔离的沙盒中并行写代码而不产生冲突。**

**核心洞察：“Agent engineering is harness engineering.”（Agent 工程的本质，是为大模型打造一辆极其坚固、解耦的“载具”和“工作台”。）**

---

## 1. 团队组建与异步消息总线 (s09: Agent Teams)

在早期的 `s04` 中，子智能体（Subagent）是“阅后即焚”的，它们没有记忆、没有名字。而在团队协作中，我们需要“常驻后台”的虚拟员工（如“前端工程师 Alice”、“测试工程师 Bob”）。

### 1.1 核心机制：JSONL 基于文件的解耦信箱 (Inbox)

多个 Python 线程（甚至多台机器）同时运行多个大模型，如果直接在内存里传递消息，极易产生并发灾难（Race Condition）。系统采用了巧妙的**“文件追加写入（Append-only）”**机制来实现高吞吐量的消息总线（MessageBus）。

#### 核心代码实现与数据流向：
```python
class MessageBus:
    def send(self, sender: str, to: str, content: str, msg_type: str = "message"):
        msg = {
            "type": msg_type,
            "from": sender,
            "content": content,
            "timestamp": time.time(),
        }
        # 为每个 AI 特工在 .team/inbox/ 目录下分配一个专属的 JSONL 文件
        inbox_path = self.dir / f"{to}.jsonl"
        
        # 追加写入，互不阻塞。Alice 正在敲代码时，Bob 依然可以往她的文件里塞信件。
        with open(inbox_path, "a") as f:
            f.write(json.dumps(msg) + "\n")
```

### 1.2 循环注入：读信与无缝唤醒
当 Alice 的大模型结束上一轮动作，准备思考下一步前，底层的 Python 循环会**强制**帮她去“查信箱”。
```python
# 核心死循环拦截：Drain (排空) 机制
inbox = BUS.read_inbox(name) 
# read_inbox 会读取所有行，然后立刻清空该 JSONL 文件（避免重复读取）

for msg in inbox:
    # 将信件伪装成 User 消息，硬塞入大模型的大脑中
    messages.append({"role": "user", "content": json.dumps(msg)})
```
*   **深度解析**：这种设计实现了**异步协作**。大模型不需要任何特殊的 API 接口，只要 Python 层面将文件里的信件追加到 `messages` 列表里，AI 就会在下一次响应时自然而然地看到：“哦，老板刚才给我发了一条新指令。”

---

## 2. 团队纪律与结构化握手协议 (s10: Team Protocols)

**核心洞察：“Same request_id correlation pattern, two domains.”（用相同的 request_id 关联模式，处理不同的业务协议。）**

如果团队里只有简单的文本聊天，管理会彻底失控。比如老板发了一句“你关机吧”，AI 可能没理解，或者正在执行其他代码而忽略了。在高级协作中，必须引入**状态机（FSM）**和**带凭证的握手协议**。

### 2.1 关机与计划审批协议 (Shutdown & Plan Approval)

这里引入了微服务架构中极度经典的 **Correlation ID（关联 ID）** 设计。任何一项涉及多步骤的协作，都必须带上 `request_id`。

#### 核心流程：以“计划审批”为例
1. **申请**：Alice 调用 `plan_approval(plan="重构数据库")` 工具。
2. **生成凭证**：底层系统生成 `req_id = "abc1234"`，将其记录在内存的 `plan_requests` 字典中，状态标记为 `pending`，并将带 ID 的消息发给老板（Lead）。
3. **审批**：老板查信箱看到了 `req_id="abc1234"` 的计划，调用审批工具。
```python
def handle_plan_review(request_id: str, approve: bool, feedback: str = "") -> str:
    # 加锁：防止并发冲突
    with _tracker_lock:
        req = plan_requests.get(request_id)
        # 状态机流转：pending -> approved / rejected
        req["status"] = "approved" if approve else "rejected"
        
    # 拿着原本的 request_id 发送正式的回执单 (Response)
    BUS.send(
        "lead", req["from"], feedback, "plan_approval_response",
        {"request_id": request_id, "approve": approve, "feedback": feedback},
    )
```

*   **深度解析**：通过这套协议，实现了**“人类在环（Human-in-the-loop）”**的绝对控制。AI 哪怕写出了毁灭系统的代码计划，只要老板不调用 `handle_plan_review` 给它 `approve: true`，这个小弟就会一直挂起（Pending），绝对无法执行下一步。

---

## 3. 自主驱动与空闲轮询机制 (s11: Autonomous Agents)

**核心洞察：“The agent finds work itself.”（自己找活干。）**

前两节的团队依然是“指令驱动”的。为了解放领导层，必须将架构升级为**“拉取模式（Pull Model）”**——建立公共任务看板（Task Board），让闲置的 AI 自己去抢单。

### 3.1 双态生命周期：工作 (WORK) 与闲置 (IDLE)

系统将 Agent 的底层循环打破，一分为二：
*   **WORK 阶段**：处理具体任务，满负荷运行，调用大模型。
*   **IDLE 阶段**：当任务完成，调用 `idle` 工具，进入极低成本的 Python `while/sleep` 轮询。

#### 核心代码实现：轮询抢单逻辑
```python
# 进入 IDLE 阶段
for _ in range(polls):
    time.sleep(POLL_INTERVAL) # 每 5 秒低功耗唤醒一次
    
    # 1. 扫描共享的 .tasks/ 文件夹，寻找 pending 状态且无主人的任务
    unclaimed = scan_unclaimed_tasks()
    
    if unclaimed:
        task = unclaimed[0]
        # 2. 修改任务 JSON，将 owner 改为自己（抢单成功）
        claim_task(task["id"], name)
        
        # 3. 将任务内容封装成 Prompt，塞入自己的上下文中
        task_prompt = f"<auto-claimed>Task #{task['id']}: {task['subject']} ... </auto-claimed>"
        messages.append({"role": "user", "content": task_prompt})
        
        # 4. 退出 IDLE 轮询，自动恢复到 WORK 阶段开始干活
        resume = True
        break 
```

### 3.2 记忆压缩后的人设重塑 (Identity Re-injection)
当 Agent 连续运行几小时后，上下文会被系统的压缩机制（s06）彻底折叠，AI 会处于一种“断片”的失忆状态。当它在 IDLE 状态下抢到新任务醒来时，它甚至不知道自己是前端还是后端。

```python
# 在接单前，检测当前消息列表长度。如果 <=3 说明刚被压缩过记忆
if len(messages) <= 3:
    # 像《记忆碎片》一样，在脑门上强行刻字：
    messages.insert(0, make_identity_block(name, role, team_name))
    # 伪造一条助手的确认回复
    messages.insert(1, {"role": "assistant", "content": f"I am {name}. Continuing."})
```
*   **深度解析**：这种“人设重注”机制彻底解决了长线运行下大模型角色漂移（Role Drift）的问题，让“永远在线、无限接单的赛博牛马”成为现实。

---

## 4. Git 工作树与代码沙盒 (s12: Worktree Isolation)

**核心洞察：“Isolate by directory, coordinate by task ID.”（用物理目录进行隔离，用任务 ID 进行逻辑协同。）**

这是目前开源界与 Devin 级商业产品最大的分水岭。
如果让 3 个 AI 自动抢单并在同一个项目目录下修改代码，必然导致灾难性的冲突（文件被互删、端口被占用、git 提交树锁死）。

### 4.1 核心机制：Git Worktree 的执行平面
利用 Git 原生的 `worktree` ，系统可以为同一个仓库，瞬间克隆出无数个完全独立的物理文件夹（沙盒），它们互不干扰，但共享同一个 `.git` 底层。

#### 核心代码实现：控制面与执行面的完美绑定
在这个架构中，一切操作分为两面：
1. **控制面（Control Plane）**：`.tasks/` 目录，管理逻辑任务。
2. **执行面（Execution Plane）**：`.worktrees/` 目录，管理物理代码沙盒。

```python
class WorktreeManager:
    def create(self, name: str, task_id: int = None, base_ref: str = "HEAD") -> str:
        # 1. 物理层克隆：在隐藏目录下瞬间创建一个平行代码库
        path = self.dir / name
        self._run_git(["worktree", "add", "-b", branch, str(path), base_ref])
        
        # 2. 逻辑绑定：将抽象的 Task ID 与真实的物理 Worktree 相互绑定
        if task_id is not None:
            self.tasks.bind_worktree(task_id, name)
            
        # 3. 观测层追踪：向事件总线（EventBus）发射日志
        self.events.emit("worktree.create.after", task={"id": task_id}, worktree={"name": name})
```

### 4.2 专属通道执行 (Observability)
有了独立沙盒，AI 不再使用普通的 `bash` 工具，而是必须使用 `worktree_run(name="auth-refactor", command="npm test")`。
底层 Python 拦截到这个调用后，会把 `cwd`（当前工作目录）精准切换到 `.worktrees/auth-refactor/` 下再执行代码。

此外，引入的 **`EventBus` (事件总线)** 生成的 `events.jsonl`，就像飞机的黑匣子。无论是人类主管还是 AI Lead，只要调用 `worktree_events`，就能以全局上帝视角看到所有沙盒的生老病死：
*   *[10:00] Task 12 created*
*   *[10:01] Alice claimed Task 12*
*   *[10:02] Worktree 'auth-refactor' created and bound to Task 12*
*   *[10:50] Worktree removed, Task 12 marked as completed.*

