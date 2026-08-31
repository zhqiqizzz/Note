
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
### 你现在应该能说出的三分钟版本

> 用户提交回答时，前端会为当前回合生成 `turnId` 并保存在 Pinia 中，网络失败重试时复用同一个 ID。请求经过 JWT Guard 后，Controller 建立 SSE 连接，并通过 RxJS Subject 把 Service 产生的阶段事件转成 SSE 数据。
> 
> Service 首先使用 Redis 锁串行化同一 `sessionId` 的回答请求，然后从内存或 MongoDB 恢复会话，并通过 `userId` 校验资源归属。后端会检查 `lastCompletedTurnId` 和 `processingTurnId`，分别处理重复请求和并发冲突。
> 
> 业务上先持久化候选人回答，再调用 Agent。Agent 分两步完成结构化回答评估和策略决策，输出经过 Zod 校验，并由服务端规则进一步限制题数、时长和追问次数。合法决策才会进入下一题生成。
> 
> 如果 SSE 断开，AbortController 会中止模型流，但已经保存的回答和 Agent 检查点会保留，回合标记为可重试。前端使用相同 `turnId` 恢复，不会重复保存回答或生成两道题。

