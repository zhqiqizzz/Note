
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
