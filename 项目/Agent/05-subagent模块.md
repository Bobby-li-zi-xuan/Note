# subagent 模块

## 模块定位

subagent 是**虚拟线程并行执行器**（`com.yourorg.biboagent.subagent`）：接收 `List<Subtask>` + 执行动作 `Function<Subtask, SubtaskResult>`，用虚拟线程并行执行，支持超时控制、取消中断、短路模式，最后按顺序返回结果。它不关心"执行什么"，只负责"怎么并行执行"；调用方是 `StageScheduler`（Plan 的每个 Stage 内，同 Stage 多任务并行）。

## 执行流程

```
StageScheduler.run()
  → toSubtasks(): PlannedTask → Subtask，超时均分 perTaskMs = max(5000, remainingMs / taskCount)
  → subagentRunner.run(subtasks, canceller, action)
      → Executors.newVirtualThreadPerTaskExecutor()，每任务一个虚拟线程（提交前检查取消）
      → daemon watchdog 线程每 50ms 轮询 canceller，检测到取消 → future.cancel(true) 批量中断
      → 按顺序收集结果（支持超时、取消、短路）
```

两种执行模式：

- **AGGREGATE（独立执行）**：默认模式。单任务超时/失败仅记录结果（`SubtaskResult.timeout/failed`），循环继续收集其余结果，互不干扰。
- **DEPENDENT（依赖执行）**：任一任务失败/超时 → `cancelRemaining()` 取消后续所有 Future → 立即返回失败（短路能力）。

取消信号全链路透传：StreamController → TurnLoopCoordinator → PlanExecutor → StageScheduler → VirtualThreadSubagentRunner，用户断连到子任务终止延迟 ≤ 50ms。

## 结果收集

subagent 只按顺序返回 `List<SubtaskResult>`，聚合由调用方完成：

```java
for (var sr : subResults) taskResultMap.put(sr.subtaskId(), sr.content());
stageResults.put(stage.id(), taskResultMap);
```

最终 `PlanExecutor` 把 stageResults 转为 `toolCallResults`（toolCallId → 结果字符串），TurnLoopCoordinator 据此为每个工具调用注入独立的 `ToolExecutionResultMessage`。

## 面试题整理

1. **什么时候触发并行执行？** StageScheduler 执行含多个 PlannedTask 的 Stage 时。来源：隐式计划（LLM 一次返回多个工具调用，自动生成单 Stage 全并行 Plan）或显式计划（create_plan 规划的多任务 Stage）。
2. **为什么用虚拟线程而不是线程池？** 工具调用是典型 IO 密集操作，平台线程昂贵且易被占满；虚拟线程仅几 KB、即用即毁、吞吐极高，且代码保持同步阻塞写法，可读性与可调试性更好，规避回调地狱。
3. **AGGREGATE 与 DEPENDENT 的区别？** 核心在 Future.get 阶段的异常分支：DEPENDENT 一旦超时/异常即 cancelRemaining 短路取消后续任务；AGGREGATE 仅包裹错误，继续收集其余结果。
4. **subagent 与主 Agent 的关系？** 不是独立 Agent——不创建独立 LLM 上下文、不维护消息历史，只并行执行传入的 action；LLM 调用、工具执行、消息历史管理都在调用方。
5. **并发调用同一模型会被限流吗？** 会，当前无内部限流，并行 HTTP 会话可能触发 429。改进方向：RateLimiter / 随机抖动延迟 / 达到门限时回退串行。
6. **为什么并行任务可中断、单工具不行？** 单工具中断需额外 watchdog 线程，性价比低（丢弃虚拟线程等兜底超时即可）；并行任务资源占用多、取消后浪费大，值得 watchdog 批量 cancel(true)。
