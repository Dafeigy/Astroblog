---
title: Bash is allyou need：Coding Agent的设计思想
published: 2026-03-16
description: '在研究了一下Opencode的代码实现后，补充了一些参考资料的学习如何构建好的Coding Agent'
image: ''
tags: ['AI','Coding Agent']
category: 'AI'
draft: false 
lang: 'zh-CN'
---

# Coding Agent 的设计

基础软件的“用户”将不仅是程序员、DevOps、数据科学家等，未来的主要用户很大可能都是各种 Agent，那么怎么为 Agents 设计基础软件呢？我认为需要遵循的是Unix的哲学：**一切皆文件，一切皆可管道。**1978 年，Doug McIlroy 在 *Bell System Technical Journal* 里记录了 Unix 设计哲学。1994 年 Peter Salus 在《A Quarter Century of Unix》里把它总结成了三句话：

> - *Write programs that do one thing and do it well.*
> - *Write programs to work together.*
> - *Write programs to handle text streams, because that is a universal interface.*

在进行设计前，无论是Agent还是其他的程序产出，不妨思考以下几个问题：

1. 你构建的程序能不能一行命令，或者直接双击就跑起来？
2. 用的协议是不是 LLM 已经"见过"的标准协议？
3. 做一次自描述性审计——把自己当成一个没有直觉、不能搜索的 Agent，只看系统返回值，能搞清楚状况吗？
4. 同一操作执行两遍，结果一致吗？

Agent 时代的基础软件竞争，不只是性能和功能。谁对 Agent 更友好，谁就更容易拿到生态位。接下来我会简单的总结一下Agent-Friendly的设计思想以及Agent的实现核心组件。

## Agent-Friendly的设计思想：Bash is all you need

过去的三十年，基础软件的用户是人。遇到软件的报错问题，我们可以去百度搜索、去问同事或者技术负责人，这似乎已经是成为了一种我们的直觉。然而我们需要认识到，Agent是没有这种直觉的，只能处理系统明确返回的内容。你需要主动告诉它"我支持哪些操作""刚才出了什么问题"，他才会知道接下来该怎么做，不然他就可能卡死；Agent 的执行链也天然脆弱：幻觉、上下文溢出、网络抖动——任何一步都可能断掉。它可能忘了自己刚才执行过某个操作；编排框架也可能因为超时重试，同一操作被执行两遍很常见。

但反过来说，Agent 也能做人做不到的事：同时操作上千个数据库实例、7×24 不停、毫秒级响应。能适配好这个用户，收益是实打实的。基于这些特质，我想应该从以下几个方面考虑Agent或Agent工具建设的设计。

### 快速启动、安全沙箱

Agent 需要快速试错。假如启动一个服务要折腾半小时配置和依赖，Agent 和你都在干等，这太不优雅了。我认为一个比较好的方式是：**容器化、single binary、合理的默认值，**可以有效降低 Agent 的启动摩擦。这不只是"方便"，是准入门槛的问题。

### 遵循开放标准协议

LLM 训练时见过海量 SQL、PromQL、HTTP。用标准协议，Agent 基本可以直接上手；用私有 DSL，Agent 就得靠 prompt 里塞的说明去猜，效果往往差一截。因此这也从另一个方面说明开源建设的重要性：开源建设的基础设施提供的文档作为语料，训练得到的模型就内置了这样的知识，并借此作为下一代程序构建的基石而发挥能力。

### 自描述性

Agent 的世界是文本：JSON、SQL、日志，都是文本。二进制协议、点鼠标的 Web UI，对它来说等于不存在。文本描述的物理世界是一定存在一定的信息损失的，如果要弥补这个信息损失就只能使用更多的文本进行描述，但这与模型的上下文窗口大小产生了矛盾。这就需要提供的文本尽可能地包含需要描述对象地信息，在我看来，Rust做得非常好：极致变态优化的编译器，在构建程序时的报错都有详细的“怎么做”、“为什么”、“是什么”的文字表述。有这三条信息，Agent 才能自己纠正后重试；没有的话，它要么放弃，要么就会陷入同一个错误的循环。

### 配置最小化

Agent 不擅长在一堆配置选项里做权衡。`batch_size` 设多少合适？这种需要经验判断的问题，Agent 很难靠自己做对。**Convention over Configuration(COC)**告诉我们给出合理默认值，支持渐进式配置，配置项最好自带注释。基础软件可以零配置启动，先跑起来再说。这个思路对 Agent 特别友好——先用默认配置把链路跑通，遇到瓶颈再按需调整。

### 可撤回和 Dry-Run

Openclaw被人诟病最多的地方就是它的安全性。如果你看过它的初代（Clawdbot）的源码，相信你一定会震惊于它直接在主进程里“裸奔“。我们设计的工具，或者提供的操作模式应该都要有一个回滚机制，用于控制意外的数据误操作恢复。

### 小结

**"Do one thing and do it well"** → 配置最小化、快速启动。`grep`、`sed`、`awk` 各司其职，才让组合成为可能。一个工具什么都能做、什么都要配置，Agent 就不知道从哪下手。

**"Work together"** → 开放标准协议、MCP。Unix 管道的精髓是：程序不需要知道上下游是谁，只要遵守同一个接口约定。MCP 做的事本质上一样——定义标准连接层，让 Agent 把任意工具接入工作流。

**"Text streams as universal interface"** → 自描述性的第一层。McIlroy 那个年代就看清楚了：文本是最通用的接口，任何程序都能读写，不需要协商格式。SQL、JSON、日志都是文本，不是巧合，是几十年实践筛出来的选择。

有意思的是，Agent 时代最受欢迎的一批工具——Claude Code、Gemini CLI、OpenClaw——基本都是 CLI-first。跟它们交互就是在终端敲命令、读文本输出、用管道组合。这不是复古，是 Unix 哲学在 AI 时代的自然延伸：当用户变成 Agent，终端和文本流比 GUI 更适合作为交互层。

Unix 哲学为什么在 Agent 时代重新焕发？因为它的核心假设跟 Agent 的工作方式高度吻合：**无状态、可组合、文本驱动**。五十年前为分时系统写下的那些原则，现在依然是给 AI Agent 设计基础软件很好的参照。接下来的内容将会是Agent核心组件的逐步引导设计指南，希望可以为正在研究程序设计的你提供一些思路和参考。

## Coding Agent：自洽的循环工具调用程序

一句话概括Coding Agent，那就是：Coding Agent是一个使用循环来控制工具调用并完成指定用户输入提示词需求任务的程序。最简单的Agent只有三个组件：循环、工具调用以及LLM的交互。随着你的任务变得复杂，构建一个（或多个）拥有复杂的Agent其实也不过是在这三者的基础上进行加法，按照功能分类的话，我想大致可以分为以下几类：

第一类：**核心的循环与工具调用。**实现这里的内容就可以基本完成一个Agent（泛用性Agent）。

第二类：**计划执行与协调。**这里的内容实现可以为Agent添加一定的记忆能力与目标导向与跟踪能力。

第三类：**记忆管控与上下文压缩。**实现这里的内容可以充分利用LLM的上下文空间。

第四类：**并发与后台任务。**实现这里的内容可以提高Agent的执行效率。

第五类：**多Agent写作模式。**依赖于第四类模块，实现术业专攻的Agent集群，可充分发挥使用不同模型的Agent的特色能力（比如让Gemini做前端、Codex做后端、Kimi做资料收集等）。Opencode的插件`oh-my-opencode`就是采用了这种模式进行设计。

以上这五类相互正交，他们共同组成一个完整的、可靠的Agent系统。大部分情况下，一般的普通用户只需关心第一类到第三类的实现集成即可，对于Coding工作相关的人员，需要对第四类和第五类进行考虑。接下来，我会结合Opencode的实现介绍一个Coding Agent应具备的组件与设计思路，以及为什么这样设计。

### Agent-Loop: 智能体运转核心

Agent本质上就是一个可以执行一系列bash指令的循环，参与到编码工作的流程中。在Coding Agent的工具出现前，用户充当了循环执行的角色：问题发送给LLM，LLM给出推理结果，然后用户再手动或者以一种简单的方式把生成的内容粘贴到代码文件中。我们能注意到：语言模型能推理代码, 但碰不到真实世界 -- 不能读文件、跑测试、看报错。没有循环, 每次工具调用用户都得手动把结果粘回去。

为了解决这个不足，设计Agent的人想到：用一个循环一直督促模型进行内容生成的迭代，并且赋予它使用工具的能力（Function Call），让它在每一个循环中都调用若干次工具，并将调用结果作为下一次循环的输入以继续完善工作，直至不再需要使用工具调用。

![Agent-Loop](https://pic1.imgdb.cn/item/69ac54bc3dbab1ba32d90856.png)

具体的流程如下：

1. 用户 prompt 作为第一条消息。
2. 将消息和工具定义一起发给 LLM。
3. 追加助手响应。检查 `stop_reason` -- 如果模型没有调用工具, 结束。
4. 执行每个工具调用, 收集结果, 作为 user 消息追加。回到第 2 步。

我们将上述流程组合到一起，就形成了`agent_loop()`:

```python
def agent_loop(query):
    messages = [{"role": "user", "content": query}]
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                output = run_bash(block.input["command"])
                results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": output,
                })
        messages.append({"role": "user", "content": results})
```

恭喜你！你成功构建了一个简单的Agent：一个可以运行bash指令的、循环闭环控制的Agent。

> 早期的Copilot、Cursor都是这种模式。尽管在生成新代码并生成diff预览时，提供了用户的确认选项，但是我们需要理解一个点：他们本身没有感知代码空间的能力。他们使用了简单的文件读写的bash指令，却无法感知自己编写的代码是否存在语法错误。

### Tool-Use: 智能体的能力基石

只有 `bash` 时, 所有操作都走 shell。`cat` 截断不可预测, `sed` 遇到特殊字符就崩, 每次 bash 调用都是不受约束的安全面。那假如我们自己编写趁手的工具呢？我们可以让Agent的能力不再局限于Bash命令，编写适合自己使用场景的工具其实也是AI基建的重要内容，而这些似乎很少被大厂和开发者提到（这是因为对大部分的开发者来讲，造轮子要么是很自然而然想到的事情，要么是完全不会产生念头的事情）。打造自己的工具吧（其实也可以改造、利用别人的工具）！

> 事实上，这也是我不喜欢OpenClaw(前身ClawdBot)的原因之一。在ClawdBot最早推出的时候，翻阅代码发现它没有做任何隔离措施，存在非常高的数据安全风险。当时的宣发口径是：“购买一台全新的Mac Mini来探索ClawdBot的能力吧！”。这其实是非常不负责的态度，但奈何其确实扩充了Agent功能（不只是简单的读写文件，而是有更多的贴近普通人日常电子设备操作）吸引了很多普通用户加入使用，目前它也在不断完善安全机制。

我们注意到，在既要安全又要不打破Agent-Loop结构的情况下，可以通过编写专用工具来解决这一情况。

![Tool-Use](https://pic1.imgdb.cn/item/69ace6e83dbab1ba32da1935.png)

我们简单看看这是怎么实现的：

1. 创建Agent操作沙箱检测机制。在设计工具时，**每个涉及文件读写操作的工具应当有一个操控的工作路径的安全检查：**

   ```python
   def safe_path(p: str) -> Path:
       path = (WORKDIR / p).resolve()
       if not path.is_relative_to(WORKDIR):
           raise ValueError(f"Path escapes workspace: {p}")
       return path
   
   def run_read(path: str, limit: int = None) -> str:
       text = safe_path(path).read_text()
       lines = text.splitlines()
       if limit and limit < len(lines):
           lines = lines[:limit]
       return "\n".join(lines)[:50000]
   ```

2. 将工具名映射到处理函数。

   ```python
   TOOL_HANDLERS = {
       "bash":       lambda **kw: run_bash(kw["command"]),
       "read_file":  lambda **kw: run_read(kw["path"], kw.get("limit")),
       "write_file": lambda **kw: run_write(kw["path"], kw["content"]),
       "edit_file":  lambda **kw: run_edit(kw["path"], kw["old_text"],
                                           kw["new_text"]),
   }
   ```

3. 循环中按名称查找处理函数。

   ```python
   for block in response.content:
       if block.type == "tool_use":
           handler = TOOL_HANDLERS.get(block.name)
           output = handler(**block.input) if handler \
               else f"Unknown tool: {block.name}"
           results.append({
               "type": "tool_result",
               "tool_use_id": block.id,
               "content": output,
           })
   ```


到这里，你知道怎么去定义和使用工具了。高质、安全的工具设计具备以下特征：

1. 有限或可控的工作路径。
2. 工具（集）的映射与统一管理。
3. Agent-Loop中的工具调用的流程处理。

### TODO-List: 简单的智能体执行管理方案

多步任务中, 模型会丢失进度 -- 重复做过的事、跳步、跑偏。这种现象会随着对话越长越严重: 工具结果不断填满上下文, 系统提示的影响力逐渐被稀释。一个 10 步重构可能做完 1-3 步就开始即兴发挥, 因为 4-10 步已经被挤出注意力了。

> 如果你是用过Cline这一VSCode的AI辅助编程插件，你会发现它经常会出现在某些执行节点中重复进行。但这个属于工具设计的缺陷：以简单的文件读取为例，Cline设计的读取工具没有针对存在文件但文件是空文本的情况进行处理。

为了解决这一问题，我们可以为Agent引入一个TODO待办管理模块，监控任务的执行器工况。同一时间只允许一个 `in_progress`。其他任务项要么处于已完成状态`completed`，要么则处于未进行`pending`状态：

```python 
class TodoManager:
    def update(self, items: list) -> str:
        validated, in_progress_count = [], 0
        for item in items:
            status = item.get("status", "pending")
            if status == "in_progress":
                in_progress_count += 1
            validated.append({"id": item["id"], "text": item["text"],
                              "status": status})
        if in_progress_count > 1:
            raise ValueError("Only one task can be in_progress")
        self.items = validated
        return self.render()
```

我们也可以把TODO管理作为一个工具添加到Tool-Use的工具映射中：

```python
TOOL_HANDLERS = {
    # ...base tools...
    "todo": lambda **kw: TODO.update(kw["items"]),
}
```

另外一个值得注意的点是，Agent可能写代码写得颠鸾倒凤不知天地为何物，这时就需要有一个机制来提醒Agent现在手头上的任务都有什么，我们称为`nag_reminder`：

```python
if rounds_since_todo >= 3 and messages:
    last = messages[-1]
    if last["role"] == "user" and isinstance(last.get("content"), list):
        last["content"].insert(0, {
            "type": "text",
            "text": "<reminder>Update your todos.</reminder>",
        })
```

> "同时只能有一个 in_progress" 强制顺序聚焦。nag reminder 制造问责压力 -- 你不更新计划, 系统就追着你问。

### Sub-Agent：智能体上下文空间高效利用

智能体工作越久, LLM中的`messages` 数组越胖。每次读文件、跑命令的输出都永久留在上下文里。"这个项目用什么测试框架?" 可能要读 5 个文件, 但父智能体只需要一个词: `pytest`。为了解决这一问题，我们可以设计Sub-Agent（父子Agent）机制：在我们不关心执行的过程和中间结果时，父Agent可以把这种脏活累活交给Sub-Agent去做。Sub-Agent执行完毕的结果以摘要总结的形式交还给Agent。

这个机制的设计是这样的：

1. 父智能体有一个 `task` 工具。子智能体拥有除 `task` 外的所有基础工具 (禁止递归生成)：
   ```python
   PARENT_TOOLS = CHILD_TOOLS + [
       {"name": "task",
        "description": "Spawn a subagent with fresh context.",
        "input_schema": {
            "type": "object",
            "properties": {"prompt": {"type": "string"}},
            "required": ["prompt"],
        }},
   ]
   ```

2. 子智能体以 `messages=[]` 启动, 运行自己的循环。只有最终文本返回给父智能体。

   ```python
   def run_subagent(prompt: str) -> str:
       sub_messages = [{"role": "user", "content": prompt}]
       for _ in range(30):  # safety limit
           response = client.messages.create(
               model=MODEL, system=SUBAGENT_SYSTEM,
               messages=sub_messages,
               tools=CHILD_TOOLS, max_tokens=8000,
           )
           sub_messages.append({"role": "assistant",
                                "content": response.content})
           if response.stop_reason != "tool_use":
               break
           results = []
           for block in response.content:
               if block.type == "tool_use":
                   handler = TOOL_HANDLERS.get(block.name)
                   output = handler(**block.input)
                   results.append({"type": "tool_result",
                       "tool_use_id": block.id,
                       "content": str(output)[:50000]})
           sub_messages.append({"role": "user", "content": results})
       return "".join(
           b.text for b in response.content if hasattr(b, "text")
       ) or "(no summary)"
   ```

   子智能体可能跑了 30+ 次工具调用, 但整个消息历史直接丢弃。父智能体收到的只是一段摘要文本, 作为普通 `tool_result` 返回。

### Skill-Loading: 按需动态加载提示词

当希望智能体遵循特定领域的工作流: git 约定、测试模式、代码审查清单。全塞进系统提示太浪费 -- 10 个技能, 每个 2000 token, 就是 20,000 token, 大部分跟**当前任务**毫无关系。解决方案也很简单，我们设计一个SkillLoader模块，判断到需要加载相应技能时，使用一个tool-use把skill的prompt加载到系统提示词里面即可。不过在这之前，最好先明白Skills的管理结构：

1. 每个技能是一个目录, 包含 `SKILL.md` 文件和 YAML frontmatter：

   ```bash
   skills/
     pdf/
       SKILL.md       # ---\n name: pdf\n description: Process PDF files\n ---\n ...
     code-review/
       SKILL.md       # ---\n name: code-review\n description: Review code\n ---\n ...
   ```

2. SkillLoader 递归扫描 `SKILL.md` 文件, 用目录名作为技能标识。

   ```python
   class SkillLoader:
       def __init__(self, skills_dir: Path):
           self.skills = {}
           for f in sorted(skills_dir.rglob("SKILL.md")):
               text = f.read_text()
               meta, body = self._parse_frontmatter(text)
               name = meta.get("name", f.parent.name)
               self.skills[name] = {"meta": meta, "body": body}
   
       def get_descriptions(self) -> str:
           lines = []
           for name, skill in self.skills.items():
               desc = skill["meta"].get("description", "")
               lines.append(f"  - {name}: {desc}")
           return "\n".join(lines)
   
       def get_content(self, name: str) -> str:
           skill = self.skills.get(name)
           if not skill:
               return f"Error: Unknown skill '{name}'."
           return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"
   ```

3. 修改提示词。第一层写入系统提示。第二层不过是 dispatch map 中的又一个工具：

   ```python
   SYSTEM = f"""You are a coding agent at {WORKDIR}.
   Skills available:
   {SKILL_LOADER.get_descriptions()}"""
   
   TOOL_HANDLERS = {
       # ...base tools...
       "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
   }
   ```

### Context-Compact: 上下文空间利用的杀手锏

上下文窗口是有限的。读一个 1000 行的文件就吃掉 ~4000 token; 读 30 个文件、跑 20 条命令, 轻松突破 100k token。不压缩, 智能体根本没法在大项目里干活。这里提出一种可行的分级压缩方案：

1. 第一层压缩`micro_compact`：每次 LLM 调用前, 将旧的 tool result 替换为占位符。

   ```python
   def micro_compact(messages: list) -> list:
       tool_results = []
       for i, msg in enumerate(messages):
           if msg["role"] == "user" and isinstance(msg.get("content"), list):
               for j, part in enumerate(msg["content"]):
                   if isinstance(part, dict) and part.get("type") == "tool_result":
                       tool_results.append((i, j, part))
       if len(tool_results) <= KEEP_RECENT:
           return messages
       for _, _, part in tool_results[:-KEEP_RECENT]:
           if len(part.get("content", "")) > 100:
               part["content"] = f"[Previous: used {tool_name}]"
       return messages
   ```

2. 第二层压缩`auto_compact`: token 超过阈值时, 保存完整对话到磁盘, 让 LLM 做摘要。

   ```python
   def auto_compact(messages: list) -> list:
       # Save transcript for recovery
       transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
       with open(transcript_path, "w") as f:
           for msg in messages:
               f.write(json.dumps(msg, default=str) + "\n")
       # LLM summarizes
       response = client.messages.create(
           model=MODEL,
           messages=[{"role": "user", "content":
               "Summarize this conversation for continuity..."
               + json.dumps(messages, default=str)[:80000]}],
           max_tokens=2000,
       )
       return [
           {"role": "user", "content": f"[Compressed]\n\n{response.content[0].text}"},
           {"role": "assistant", "content": "Understood. Continuing."},
       ]
   ```

3. 第三层压缩`manual_compact`: `compact` 工具按需触发同样的摘要机制。

4. 循环整合三层压缩结构：

   ```python
   def agent_loop(messages: list):
       while True:
           micro_compact(messages)                        # Layer 1
           if estimate_tokens(messages) > THRESHOLD:
               messages[:] = auto_compact(messages)       # Layer 2
           response = client.messages.create(...)
           # ... tool execution ...
           if manual_compact:
               messages[:] = auto_compact(messages)       # Layer 3
   ```

### Task-System: Session管理项目的利器

我们曾经使用过一个TODO-List来管理任务执行情况。然而，这种方式只是内存中的扁平清单: 没有顺序、没有依赖、状态只有做完没做完。真实目标是有结构的：任务 B 依赖任务 A, 任务 C 和 D 可以并行, 任务 E 要等 C 和 D 都完成。**如果没有显式的关系, 智能体分不清什么能做、什么被卡住、什么能同时跑。**而且清单只活在内存里, 一旦上下文空间爆了那就什么都没有了。

解决方案很显而易见：把原本的TODO-List持久化到硬盘里面。每个任务是一个 JSON 文件, 有状态、前置依赖 (`blockedBy`) 和后置依赖 (`blocks`)，他们构成一个DAG（有向无环图）结构。任务图随时回答三个问题:

1. **什么可以做?** -- 状态为 `pending` 且 `blockedBy` 为空的任务。
2. **什么被卡住?** -- 等待前置任务完成的任务。
3. **什么做完了?** -- 状态为 `completed` 的任务, 完成时自动解锁后续任务。

这个架构的实现会有点复杂，但回报也是非常可观的，因为后面还可以集成后台任务、agent集群和worktree隔离机制。我们来看看具体的实现：

1. **TaskManager**: 每个任务一个 JSON 文件, CRUD + 依赖图。

   ```python
   class TaskManager:
       def __init__(self, tasks_dir: Path):
           self.dir = tasks_dir
           self.dir.mkdir(exist_ok=True)
           self._next_id = self._max_id() + 1
   
       def create(self, subject, description=""):
           task = {"id": self._next_id, "subject": subject,
                   "status": "pending", "blockedBy": [],
                   "blocks": [], "owner": ""}
           self._save(task)
           self._next_id += 1
           return json.dumps(task, indent=2)
   ```

2. **依赖解除**: 完成任务时, 自动将其 ID 从其他任务的 `blockedBy` 中移除, 解锁后续任务。

   ```python
   def _clear_dependency(self, completed_id):
       for f in self.dir.glob("task_*.json"):
           task = json.loads(f.read_text())
           if completed_id in task.get("blockedBy", []):
               task["blockedBy"].remove(completed_id)
               self._save(task)
   ```

3. **状态变更 + 依赖关联**: `update` 处理状态转换和依赖边。

   ```python
   def update(self, task_id, status=None,
              add_blocked_by=None, add_blocks=None):
       task = self._load(task_id)
       if status:
           task["status"] = status
           if status == "completed":
               self._clear_dependency(task_id)
       self._save(task)
   ```

4. 映射到tool_use：

   ```python
   TOOL_HANDLERS = {
       # ...base tools...
       "task_create": lambda **kw: TASKS.create(kw["subject"]),
       "task_update": lambda **kw: TASKS.update(kw["task_id"], kw.get("status")),
       "task_list":   lambda **kw: TASKS.list_all(),
       "task_get":    lambda **kw: TASKS.get(kw["task_id"]),
   }
   ```

### Background-Tasks：高性能魅力时刻

我们知道，有些命令是很耗时的，比如构建项目、网络资源访问，比如 `npm install`、`pytest`、`docker build`。阻塞式循环下模型只能干等。用户说 "装依赖, 顺便建个配置文件", 智能体却只能一个一个来。这你受得了吗？作为一个高性能的Agent，你当然要充分利用起来整个计算机的资源！我们可以构建一个`BackgroundManager`使用后台任务机制来完成这些耗时费力的任务。

1. `BackgroundManager` 用线程安全的通知队列追踪任务。

   ```python
   class BackgroundManager:
       def __init__(self):
           self.tasks = {}
           self._notification_queue = []
           self._lock = threading.Lock()
   ```

2. `run()` 启动守护线程, 立即返回。

   ```python
   def run(self, command: str) -> str:
       task_id = str(uuid.uuid4())[:8]
       self.tasks[task_id] = {"status": "running", "command": command}
       thread = threading.Thread(
           target=self._execute, args=(task_id, command), daemon=True)
       thread.start()
       return f"Background task {task_id} started"
   ```

3. 子进程完成后, 结果进入通知队列。

   ```python
   def _execute(self, task_id, command):
       try:
           r = subprocess.run(command, shell=True, cwd=WORKDIR,
               capture_output=True, text=True, timeout=300)
           output = (r.stdout + r.stderr).strip()[:50000]
       except subprocess.TimeoutExpired:
           output = "Error: Timeout (300s)"
       with self._lock:
           self._notification_queue.append({
               "task_id": task_id, "result": output[:500]})
   ```

4. 每次 LLM 调用前排空通知队列。

   ```python
   def agent_loop(messages: list):
       while True:
           notifs = BG.drain_notifications()
           if notifs:
               notif_text = "\n".join(
                   f"[bg:{n['task_id']}] {n['result']}" for n in notifs)
               messages.append({"role": "user",
                   "content": f"<background-results>\n{notif_text}\n"
                              f"</background-results>"})
               messages.append({"role": "assistant",
                   "content": "Noted background results."})
           response = client.messages.create(...)
   ```

   > 注意：Agent-Loop是保持单线程执行的，**被并行化的只有子进程 I/O** 。

### Agent-Swarm/Agent-Team：团队作战

子智能体 Sub-Agent 是一次性的: 生成、干活、返回摘要、消亡。没有身份, 没有跨调用的记忆。后台任务Background-Tasks 能跑 shell 命令, 但做不了 LLM 引导的决策。**真正的团队协作需要三样东西: (1) 能跨多轮对话存活的持久智能体, (2) 身份和生命周期管理, (3) 智能体之间的通信通道。**我们需要一个可以管理团队的`TeammateManager`实现这些功能。`TeammateManager`设计逻辑如下：

1. `TeammateManager` 通过 config.json 维护团队名册。

   ```python
   class TeammateManager:
       def __init__(self, team_dir: Path):
           self.dir = team_dir
           self.dir.mkdir(exist_ok=True)
           self.config_path = self.dir / "config.json"
           self.config = self._load_config()
           self.threads = {}
   ```

2. `spawn()` 创建队友并在线程中启动 agent loop。

   ```python
   def spawn(self, name: str, role: str, prompt: str) -> str:
       member = {"name": name, "role": role, "status": "working"}
       self.config["members"].append(member)
       self._save_config()
       thread = threading.Thread(
           target=self._teammate_loop,
           args=(name, role, prompt), daemon=True)
       thread.start()
       return f"Spawned teammate '{name}' (role: {role})"
   ```

3. MessageBus: append-only 的 JSONL 收件箱。`send()` 追加一行; `read_inbox()` 读取全部并清空。

   ```python
   class MessageBus:
       def send(self, sender, to, content, msg_type="message", extra=None):
           msg = {"type": msg_type, "from": sender,
                  "content": content, "timestamp": time.time()}
           if extra:
               msg.update(extra)
           with open(self.dir / f"{to}.jsonl", "a") as f:
               f.write(json.dumps(msg) + "\n")
   
       def read_inbox(self, name):
           path = self.dir / f"{name}.jsonl"
           if not path.exists(): return "[]"
           msgs = [json.loads(l) for l in path.read_text().strip().splitlines() if l]
           path.write_text("")  # drain
           return json.dumps(msgs, indent=2)
   ```

4. 每个队友在每次 LLM 调用前检查收件箱, 将消息注入上下文。

   ```python
   def _teammate_loop(self, name, role, prompt):
       messages = [{"role": "user", "content": prompt}]
       for _ in range(50):
           inbox = BUS.read_inbox(name)
           if inbox != "[]":
               messages.append({"role": "user",
                   "content": f"<inbox>{inbox}</inbox>"})
               messages.append({"role": "assistant",
                   "content": "Noted inbox messages."})
           response = client.messages.create(...)
           if response.stop_reason != "tool_use":
               break
           # execute tools, append results...
       self._find_member(name)["status"] = "idle"
   ```

### Team Communication Protocol：团队通信协议

虽然同时运行多个Agent可能会提高产出效率，但是也不避免地出现协调问题：因为信息的流通是通过一个indexbox实现的，只是单向的（这里和正常的电子邮箱收件箱机制不一样）。我们应该需要参考TCP的握手确保信息的可靠传输以及状态的稳定存储。有限状态机（Fiinite-State-Machine，FSM）用于表征Agent通信的结果状态。

有两个场景值得我们关注：一个是Shutdown某个Agent（或者说让它休眠、未激活），一个是协调任务分配。Shutdown一个Agent需要另一个Agent 的参与（**这意味着他们是平级关系，而不是父子关系**），我们不妨称这个参与者为`发起人`。

#### Shutdown a Agent:

1. 发起人生成 request_id, 通过收件箱发起 shutdown 请求。

   ```python
   shutdown_requests = {}
   
   def handle_shutdown_request(teammate: str) -> str:
       req_id = str(uuid.uuid4())[:8]
       shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
       BUS.send("lead", teammate, "Please shut down gracefully.",
                "shutdown_request", {"request_id": req_id})
       return f"Shutdown request {req_id} sent (status: pending)"
   ```

2. 队友收到请求后, 用 approve/reject 响应。

   ```python
   if tool_name == "shutdown_response":
       req_id = args["request_id"]
       approve = args["approve"]
       shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
       BUS.send(sender, "lead", args.get("reason", ""),
                "shutdown_response",
                {"request_id": req_id, "approve": approve})
   ```

   

同样的 `pending -> approved | rejected` 状态机可以套用到任何请求-响应协议上。这也意味着，任务审批也可以这样操作。

#### Task Assignment

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

### Autonomous Agents：自洽、自动、自主

上面的通信协议实现了队友只在被明确指派时的行动。但这样负责分任务的化，领导Agent得给每个队友写 prompt, 任务看板上 10 个未认领的任务也需要手动分配。如果要实现真正的自洽，需要队友自己扫描任务看板, 认领没人做的任务, 做完再找下一个。我们可以设计一个机制让他们自己找活干：

1. Teammate Loop

   Teammate Loop分两个阶段: WORK 和 IDLE。LLM 停止调用工具 (或调用了 `idle`) 时, 进入 IDLE。

   ```python
   def _loop(self, name, role, prompt):
       while True:
           # -- WORK PHASE --
           messages = [{"role": "user", "content": prompt}]
           for _ in range(50):
               response = client.messages.create(...)
               if response.stop_reason != "tool_use":
                   break
               # execute tools...
               if idle_requested:
                   break
   
           # -- IDLE PHASE --
           self._set_status(name, "idle")
           resume = self._idle_poll(name, messages)
           if not resume:
               self._set_status(name, "shutdown")
               return
           self._set_status(name, "working")
   ```

2. IDLE Task Pulling

   空闲阶段循环轮询收件箱和任务看板。

   ```python
   def _idle_poll(self, name, messages):
       for _ in range(IDLE_TIMEOUT // POLL_INTERVAL):  # 60s / 5s = 12
           time.sleep(POLL_INTERVAL)
           inbox = BUS.read_inbox(name)
           if inbox:
               messages.append({"role": "user",
                   "content": f"<inbox>{inbox}</inbox>"})
               return True
           unclaimed = scan_unclaimed_tasks()
           if unclaimed:
               claim_task(unclaimed[0]["id"], name)
               messages.append({"role": "user",
                   "content": f"<auto-claimed>Task #{unclaimed[0]['id']}: "
                              f"{unclaimed[0]['subject']}</auto-claimed>"})
               return True
       return False  # timeout -> shutdown
   ```

3. Task Scan:

   找 pending 状态、无 owner、未被阻塞的任务。

   ```python
   def scan_unclaimed_tasks() -> list:
       unclaimed = []
       for f in sorted(TASKS_DIR.glob("task_*.json")):
           task = json.loads(f.read_text())
           if (task.get("status") == "pending"
                   and not task.get("owner")
                   and not task.get("blockedBy")):
               unclaimed.append(task)
       return unclaimed
   ```

4. 身份重注入

   上下文过短 (说明发生了压缩) 时, 在开头插入身份块。这是为了解决长对话后忘记自身身份的一个patch。

   ```python
   if len(messages) <= 3:
       messages.insert(0, {"role": "user",
           "content": f"<identity>You are '{name}', role: {role}, "
                      f"team: {team_name}. Continue your work.</identity>"})
       messages.insert(1, {"role": "assistant",
           "content": f"I am {name}. Continuing."})
   ```

   

### Worktree + Task Isolation：代码空间隔离

假设有两个Agent在同时对工作区的同一个文件进行修改，那就会产生恶劣的影响。未提交的改动互相污染, 谁也没法干净回滚。Task-System管 "做什么" 但不管 "在哪做"。解决方法是: **给每个任务一个独立的 git worktree 目录, 用任务 ID 把两边关联起来。**

1. **创建任务。** 先把目标持久化。

   ```python
   TASKS.create("Implement auth refactor")
   # -> .tasks/task_1.json  status=pending  worktree=""
   ```

2. **创建 worktree 并绑定任务。** 传入 `task_id` 自动将任务推进到 `in_progress`。

   ```python
   WORKTREES.create("auth-refactor", task_id=1)
   # -> git worktree add -b wt/auth-refactor .worktrees/auth-refactor HEAD
   # -> index.json gets new entry, task_1.json gets worktree="auth-refactor"
   ```

   绑定同时写入两侧状态：

   ```python
   def bind_worktree(self, task_id, worktree):
       task = self._load(task_id)
       task["worktree"] = worktree
       if task["status"] == "pending":
           task["status"] = "in_progress"
       self._save(task)
   ```

3. **在 worktree 中执行命令。** `cwd` 指向隔离目录。

   ```python
   subprocess.run(command, shell=True, cwd=worktree_path,
                  capture_output=True, text=True, timeout=300)
   ```

4. **收尾。** 两种选择:

   - `worktree_keep(name)` -- 保留目录供后续使用。
   - `worktree_remove(name, complete_task=True)` -- 删除目录, 完成绑定任务, 发出事件。一个调用搞定拆除 + 完成。

   ```python
   def remove(self, name, force=False, complete_task=False):
       self._run_git(["worktree", "remove", wt["path"]])
       if complete_task and wt.get("task_id") is not None:
           self.tasks.update(wt["task_id"], status="completed")
           self.tasks.unbind_worktree(wt["task_id"])
           self.events.emit("task.completed", ...)
   ```

5. **事件流。** 每个生命周期步骤写入 `.worktrees/events.jsonl`:

   ```python
   {
     "event": "worktree.remove.after",
     "task": {"id": 1, "status": "completed"},
     "worktree": {"name": "auth-refactor", "status": "removed"},
     "ts": 1730000000
   }
   ```

   事件类型: 

   1. `worktree.create.before/after/failed`
   2. `worktree.remove.before/after/failed`
   3. `worktree.keep`
   4. ``task.completed`。

   崩溃后从 `.tasks/` + `.worktrees/index.json` 重建现场。会话记忆是易失的; 而磁盘状态是持久的。

## 总结&参考资料

学习Agent的设计还是蛮有意思的，因为它本身并不复杂，但却有一种四两拨千斤的意思在里面。对于一个普通的程序员来讲，自己用一门陌生的语言来实现一个Agent其实是最好的实践方式，我或许会在下一篇博文中介绍一下我的实践。本文的大部分资料来源于以下优质内容：

1. [Learn Claude Code](https://learn-claude-agents.vercel.app/en/)：我的主要学习资料。
2. [为 OpenClaw(Agent) 设计的基础软件长什么样？](https://mp.weixin.qq.com/s/9AfA7nD_LnDy85TrQcfS-A?click_id=1)：Unix哲学与Agent设计思想的高度总结。
