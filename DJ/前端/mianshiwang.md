
### 一次回合中数据如何变化

|阶段|MongoDB|内存 Session|前端|
|---|---|---|---|
|提交前|当前问题未回答|`idle`|`waiting`|
|接收回答|保存 answer、turnId|`evaluating`|显示候选人回答|
|Agent 评估|保存 Session 检查点|`evaluating`|正在分析回答|
|Agent 决策|更新 agentState|`generating`|显示维度/难度|
|生成问题|创建问题占位|保存流式累积结果|流式显示问题|
|生成完成|写入问题和参考答案|`idle`|`waiting`|
|回合完成|保存 lastCompletedTurnId|清除 processingTurnId|清除 pendingTurn|
|失败|回答仍保留|`retryable`|允许重试|
系统先*评估*
```
{
  "dimension": "SSE",
  "score": 58,
  "strengths": [
    "能够说明 SSE 用于实时推送"
  ],
  "gaps": [
    "没有解释事件协议",
    "没有说明断线恢复",
    "没有说明代理服务器缓冲问题"
  ],
  "needsFollowUp": true
}
```

然后再根据评估结果和面试状态*决策*
```
{
  "action": "follow_up",
  "dimension": "SSE",
  "difficulty": "medium",
  "reasonCode": "insufficient_depth"
}
```

*服务端策略校验逐条拆解*
模型决策通过 Zod 后，进入：

```
enforceAgentPolicy(
  state,
  decision,
  elapsedMinutes,
  targetDuration,
);
```
规则一：达到总时长，强制结束
规则二：达到题数上限，强制结束
规则三：接近时间上限，不再追问
规则四：追问次数越界
规则五：模型选择未知维度

### 最终决策如何控制下一题 Prompt

Agent 不直接生成题目，只生成一个受控指令：

```
{
  "action": "follow_up",
  "dimension": "SSE",
  "difficulty": "medium",
  "reasonCode": "insufficient_depth",
  "instruction": "针对候选人刚才回答中的缺口继续追问，只提出一个问题。"
}
```

如果是切换维度：

```
{
  "action": "new_dimension",
  "dimension": "MongoDB",
  "difficulty": "hard",
  "reasonCode": "coverage_gap",
  "instruction": "切换到尚未覆盖的能力维度，只提出一个问题。"
}
```

这个 JSON 被序列化成 `agentDirective`，传给问题生成 Prompt。

### 为什么选择 SSE

常见实时通信方案：

| 方案        | 通信方向      | 适合场景         |
| --------- | --------- | ------------ |
| 普通 HTTP   | 请求一次、响应一次 | 快速 CRUD      |
| 轮询        | 前端定时查询    | 报告状态、支付状态    |
| SSE       | 服务端持续推送   | AI 文本生成、任务进度 |
| WebSocket | 双向长连接     | 实时游戏、语音、协同编辑 |

模拟面试的文字生成具有这些特点：

```
前端提交一次回答
服务端持续返回多个事件
同一时刻主要是服务端向前端推送
基于 HTTP 即可
```

因此 SSE 比 WebSocket 更轻量。

但报告生成使用轮询更合适，因为报告是一个持久化后台任务：

```
前端刷新页面
→ GET 报告进度
→ 数据库告诉前端真实状态
```

所以项目里是两种方案并存：

```
当前面试回合：SSE
后台报告任务：BullMQ + 状态轮询
```

### 项目的事件协议

当前主要事件：

| 事件                 | 含义      | 前端状态    |
| ------------------ | ------- | ------- |
| `start`            | 开场白流式生成 | 显示面试官开场 |
| `agent_evaluating` | 正在评估回答  | 正在分析回答  |
| `agent_deciding`   | 正在选择策略  | 显示维度、难度 |
| `thinking`         | 准备生成问题  | AI 思考中  |
| `question`         | 下一题内容   | 流式更新消息  |
| `reference_answer` | 参考答案    | 更新对应答案  |
| `waiting`          | 下一题生成完成 | 允许用户输入  |
| `end`              | 面试结束    | 进入报告阶段  |
| `error`            | 当前流程失败  | 允许安全重试  |
|                    |         |         |
### 为什么采用累计内容

优点：

- 前端不容易丢字符；
- 某一段事件重复时不会重复拼接；
- 状态更新简单；
- 更容易处理语音播报增量。

缺点：

- 每个事件都重复发送之前内容；
- 长文本总传输量接近 O(n²)；
- 问题很长时带宽浪费。

首版可以接受，因为面试题通常不长。

如果未来输出长报告，可以改成：
### 面试时的完整回答模板

> 我没有让大模型直接控制面试流程，而是把 Agent 拆成回答评估、策略决策和问题生成三个阶段。评估结果与决策结果都通过 Zod 做运行时校验，模型输出非法 JSON 时只进行一次格式修复。
> 
> Zod 通过后还不能直接执行，因为结构合法不代表业务合法。服务端会根据题数、时长、维度白名单、单维度追问次数和连续追问次数再次校验。如果模型决策越界，后端会将其降级为切换维度或结束。
> 
> 最终数据库只保存能力维度、评分、动作、难度和受控原因码，不保存模型思维链。问题生成模型也不能自行结束 V1 面试，是否结束只能由通过服务端规则校验后的 Agent 决策决定。
> 
> 这样模型负责智能判断，NestJS 负责权限、状态和业务边界，避免模型输出直接影响权益和数据库。
### 你现在应该能说出的三分钟版本

> 用户提交回答时，前端会为当前回合生成 `turnId` 并保存在 Pinia 中，网络失败重试时复用同一个 ID。请求经过 JWT Guard 后，Controller 建立 SSE 连接，并通过 RxJS Subject 把 Service 产生的阶段事件转成 SSE 数据。
> 
> Service 首先使用 Redis 锁串行化同一 `sessionId` 的回答请求，然后从内存或 MongoDB 恢复会话，并通过 `userId` 校验资源归属。后端会检查 `lastCompletedTurnId` 和 `processingTurnId`，分别处理重复请求和并发冲突。
> 
> 业务上先持久化候选人回答，再调用 Agent。Agent 分两步完成结构化回答评估和策略决策，输出经过 Zod 校验，并由服务端规则进一步限制题数、时长和追问次数。合法决策才会进入下一题生成。
> 
> 如果 SSE 断开，AbortController 会中止模型流，但已经保存的回答和 Agent 检查点会保留，回合标记为可重试。前端使用相同 `turnId` 恢复，不会重复保存回答或生成两道题。

### 你在面试中可以这样讲

> 我把一次面试回答建模成一个带 `turnId` 的业务回合。后端获得会话锁后，先校验资源归属和回合幂等，再把候选人回答写入 `qaList` 和 `sessionState`，之后才调用 Agent。
> 
> Agent 的评估、决策和下一题生成属于后续阶段，即使模型超时或 SSE 断开，用户回答也不会丢失。生成下一题时，我会先创建带 `turnId` 的问题占位项，生成成功后再更新问题和参考答案，最后保存 `lastCompletedTurnId` 检查点。
> 
> 因此同一回合重试时，系统能够判断回答是否已经保存、问题占位是否已经创建、回合是否已经完成，从而避免重复保存回答或生成两道题。面试结束后只更新为 `queued` 并投递 BullMQ，报告 Worker 独立完成后续状态流转。