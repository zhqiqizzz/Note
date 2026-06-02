**LangChain 是一个用来开发 LLM 应用和 AI Agent 的框架/生态**。

你可以把它理解成：

```
大模型 API
+ Prompt 管理
+ 工具调用
+ RAG 知识库
+ Agent 流程
+ 各种第三方集成
```

它不是一个模型，而是帮你把模型、工具、数据库、搜索、文档、向量库这些东西连接起来。

**它主要解决什么**

比如你不用框架时，做一个知识库问答要自己写：

```
读取文档
切分文档
生成 embedding
存向量数据库
用户提问
检索相关片段
拼 prompt
调用模型
返回答案
```

LangChain 可以提供这些模块：

```
Document Loader
Text Splitter
Embedding
Vector Store
Retriever
Prompt Template
LLM / Chat Model
Chain
Agent
Tool
```

**LangChain 适合做什么**

常见场景：

```
RAG 知识库问答
聊天机器人
文档分析
联网搜索总结
调用工具的 Agent
SQL 数据库问答
自动化工作流
多步骤 LLM 应用
```

**核心概念**

```
Model：接不同大模型，比如 OpenAI、Anthropic、本地模型
Prompt：管理提示词模板
Chain：把多个步骤串起来
Tool：模型可以调用的工具
Agent：模型自己决定下一步调用什么工具
Retriever：从知识库检索资料
Vector Store：向量数据库
Memory：对话记忆
```

**LangChain 和 LangGraph**

现在很多复杂 Agent 更推荐看 **LangGraph**。

简单理解：

```
LangChain：组件库，帮你连接模型、工具、RAG 等
LangGraph：更适合做复杂、有状态、可控的 Agent 工作流
```

如果是初学：

```
先学 LangChain 的 RAG 和 Tool
再学 LangGraph 的状态图和 Agent
```

**和 OpenAI Agents SDK 的区别**

```
LangChain：
生态广，支持很多模型、向量库、工具，适合多供应商集成。

OpenAI Agents SDK：
更贴 OpenAI 官方生态，工具调用、handoff、tracing 比较直接。

LangGraph：
适合复杂流程控制，比如多步骤、有状态、可恢复 Agent。
```

一句话总结：

**LangChain 是一个 LLM 应用开发框架，主要帮你把大模型和外部数据、工具、流程连接起来。**

**LangGraph 是 LangChain 生态里的“有状态 Agent / 工作流编排框架”。**

如果说 LangChain 更像一堆 LLM 应用组件：

```
模型
Prompt
Tool
Retriever
Vector Store
Agent
```

那 LangGraph 更像：

```
把这些组件组织成一个可控的流程图
```

它特别适合做 **复杂、多步骤、有状态、可恢复的 AI Agent**。

**核心理解**

LangGraph 的名字里有 Graph，意思是你把 Agent 流程写成一张图：

```
节点 Node：一个步骤
边 Edge：下一步走向
状态 State：流程中共享的数据
条件边 Conditional Edge：根据结果决定走哪条路
```

比如一个研究 Agent：

```
用户问题
  -> 规划
  -> 搜索
  -> 总结
  -> 判断是否需要继续搜索
      -> 是：回到搜索
      -> 否：输出答案
```

用图表示就是：

```
start
  -> plan
  -> search
  -> summarize
  -> should_continue?
      -> search
      -> final
```

**它和普通 Chain 的区别**

普通 Chain 往往是线性的：

```
A -> B -> C -> 输出
```

LangGraph 可以有：

```
分支
循环
状态
中断
恢复
人工确认
多 Agent 协作
```

比如 Agent 调工具不是固定一次，可能是：

```
想一步
调用工具
看结果
再想一步
再调用工具
直到完成
```

这类“循环 + 状态”的流程，用 LangGraph 更自然。

**核心概念**

==State==

流程共享状态。

例如：

```ts
{
  question: string;
  messages: Message[];
  searchResults: string[];
  finalAnswer?: string;
}
```

每个节点都可以读取和更新 state。

==Node==

一个处理步骤。

比如：

```
planNode：生成计划
searchNode：调用搜索工具
answerNode：生成最终回答
```

==Edge==

连接节点。

```
plan -> search
search -> answer
```

==Conditional== Edge

根据条件选择下一步。

比如：

```
如果资料不够，继续 search
如果资料足够，进入 answer
```

==Checkpoint==

保存状态，让长任务可以恢复。

比如 Agent 跑到一半中断了，可以从上次状态继续。

==Human-in-the-loop==

支持人工介入。

比如：

```
Agent 想发邮件
暂停
让用户确认
用户确认后继续执行
```

**为什么 Agent 需要 LangGraph**

很多 Agent 不是简单的一问一答，而是：

```
需要多轮推理
需要调用多个工具
需要循环检查结果
需要保存中间状态
需要失败恢复
需要人工审批
需要可观测 trace
```

例如自动办公 Agent：

```
读取用户需求
  -> 判断任务类型
  -> 查日历
  -> 草拟邮件
  -> 等用户确认
  -> 发送邮件
  -> 记录结果
```

这不是一个简单函数，而是一个流程图。

**LangGraph 适合什么场景**

适合：

```
复杂 Agent
多工具调用
多步骤工作流
需要循环决策
需要状态保存
需要人工确认
需要长时间运行
多 Agent 协作
RAG + 工具 + 判断流程
```

不太需要 LangGraph 的场景：

```
简单聊天
简单 RAG 问答
固定的一次性文本生成
一个 prompt 就能完成的小任务
```

这些用普通 LangChain 或直接调用模型 API 就够了。

**一个简单例子**

假设你要做一个客服 Agent：

```
用户问题
  -> 判断是否需要查知识库
  -> 查知识库
  -> 判断是否能回答
  -> 能回答：生成答案
  -> 不能回答：转人工
```

LangGraph 思路：

```
start
  -> classify
  -> retrieve
  -> decide
      -> answer
      -> human
```

这比写一堆 if/else 更清楚，也更容易扩展。

**和 LangChain 的关系**

简单说：

```
LangChain 提供零件
LangGraph 负责组装流程
```

LangGraph 里面可以使用 LangChain 的：

```
Chat Model
Tool
Retriever
Prompt
Vector Store
```

但 LangGraph 更关注：

```
流程控制
状态管理
节点编排
循环
恢复
人工介入
```

**和 OpenAI Agents SDK 的区别**

```
OpenAI Agents SDK：
更偏官方 OpenAI Agent 体验，工具调用、handoff、tracing 简洁。

LangGraph：
更偏复杂流程编排，适合高度可控、状态化、多步骤 Agent。

LangChain：
更偏组件生态。
```

**一句话总结**

LangGraph 是一个把 AI Agent 写成“状态图”的框架。

它解决的是：

```
Agent 下一步该做什么
状态怎么保存
工具怎么循环调用
什么时候结束
失败后怎么恢复
什么时候让人确认
```