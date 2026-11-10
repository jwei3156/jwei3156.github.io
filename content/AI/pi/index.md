---
title: "Pi源码阅读"
date: 2026-11-05T20:00:00+08:00
draft: false
---

```
https://github.com/earendil-works/pi/
```

pi通过了极简的Agent执行内核、可扩展的工具体系、上下文管理和事件机制，构建能够执行复杂任务的智能体

主要分为三个模块

- pi-ai

屏蔽不同LLM api差异

- pi-agent-core

实现Agent loop、工具执行、事件机制

- pi-coding-agent

会话管理、上下文压缩、skills、extensions

**工具设计**

默认只提供四个基础工具：read读文件、write写文件、edit修改文件、bash执行命令

**上下文管理**

session tree + compaction。使用具有父子关系的会话记录结构，并持久化jsonl，每个节点关联前一个节点

上下文接近限制时，较早的历史压缩成摘要

**事件驱动机制**

pi agent在执行过程中不断向外发出事件，例如：开始任务、调用工具结束等

**Steering**

执行中动态介入，能够在当前轮次结束后向执行过程注入新的消息



## Pi源码阅读

agent的基础执行机制

属于ReAct模式：模型推理-调用工具-获取反馈-再次推理

### 循环机制

我们看核心调用链：`agentLoop()` → `runAgentLoop()` → `runLoop()` → `streamAssistantResponse()` → `executeToolCalls()`

在**/packages/agent/src/agent-loop.ts**

- agentLoop()

```typescript
export function agentLoop(
	prompts: AgentMessage[],					// 新提交的消息
	context: AgentContext,						// 当前agent的上下文
	config: AgentLoopConfig,					// 模型配置
	signal: AbortSignal | undefined,	// 用于是否取消/中断执行
	streamFn: StreamFn,								// 实际大模型接口
): EventStream<AgentEvent, AgentMessage[]> {
  // 构造一个空流
  const stream = createAgentStream();

  // 后台异步启动，产生的事件push进流里，实现实时观测
	void runAgentLoop() {...}

	return stream;
}
```

返回的类型是stream。没有等待整个Agent执行完成，而是立即返回时间流，从而外部调用者接受到意见，能够持续观察Agent执行状态

- runAgentLoop()

```typescript
export async function runAgentLoop(
	prompts: AgentMessage[],
	context: AgentContext,
	config: AgentLoopConfig,
	emit: AgentEventSink,
	signal: AbortSignal | undefined,
	streamFn: StreamFn,
): Promise<AgentMessage[]> {
  // 除了原始用户输入，还会有工具可用性变化的信息
	const initialMessages = declareToolChanges(context, prompts);
  // 收集本地loop新产生的消息
	const newMessages: AgentMessage[] = [...initialMessages];
  // 接下来执行使用的上下文，包括既有消息和本次新增消息
	const currentContext: AgentContext = {
		...context,
		messages: [...context.messages, ...initialMessages],
	};

	await emit({ type: "agent_start" });
	await emit({ type: "turn_start" });
	for (const message of initialMessages) {
		await emit({ type: "message_start", message });
		await emit({ type: "message_end", message });
	}

	await runLoop(currentContext, newMessages, config, signal, emit, streamFn ?? getDefaultStreamFn());
	return newMessages;
}
```

总体来说，进行了薄编排，把输入整理为上下文和累积器

- runLoop()

```typescript
async function runLoop(
	initialContext: AgentContext,
	newMessages: AgentMessage[],
	initialConfig: AgentLoopConfig,
	signal: AbortSignal | undefined,
	emit: AgentEventSink,
	streamFunction: StreamFn,
): Promise<void> {
	let currentContext = initialContext;
	let config = initialConfig;
	let lastCompletedTurn: PrepareNextTurnContext | undefined;
	let explicitContinuation = false;
	let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

	// 外层循环，当前Agent完成工作后，是否还有Follow-up或显示继续请求
	while (true) {
		let hasMoreToolCalls = true;

		// 内层循环，调用模型
		while (hasMoreToolCalls || pendingMessages.length > 0) {
			let preparedMessages: AgentMessage[] = [];
      // 上一轮进行收尾
			if (lastCompletedTurn) {
				const nextTurnSnapshot = await config.prepareNextTurn?.(lastCompletedTurn);
				if (nextTurnSnapshot) {
          ...
				}
				// 准备阶段，获取用户补发的消息
				if (pendingMessages.length === 0) {
					pendingMessages = (await config.getSteeringMessages?.()) || [];
				}
				await emit({ type: "turn_start" });
			}

			// 处理准备工作、工具声明
			for (const message of declareToolChanges(currentContext, [...preparedMessages, ...pendingMessages])) {
				await emit({ type: "message_start", message });
				await emit({ type: "message_end", message });
				currentContext.messages.push(message);
				newMessages.push(message);
			}
			pendingMessages = [];

      // 请求前钩子，做最后调整，比如切换模型等
			const requestUpdate = await config.prepareRequest?.(
				{
					context: currentContext,
					model: config.model,
					thinkingLevel: config.reasoning ?? "off",
				},
				signal,
			);
			if (requestUpdate) {
				...
			}

			// 获取流式回复
			const message = await streamAssistantResponse(currentContext, config, signal, emit, streamFunction);
			newMessages.push(message);
        
			// 处理错误/中止
			if (message.stopReason === "error" || message.stopReason === "aborted") {
				...
			}

			// 工具调用
			const toolCalls = message.content.filter((c) => c.type === "toolCall");
			...
			}

      // 收尾与决策
			lastCompletedTurn = {
				message,
				toolResults,
				context: currentContext,
				newMessages,
			};
			const decision = await config.finishTurn?.(lastCompletedTurn, signal);
			await emit({ type: "turn_end", message, toolResults });

			if (decision?.action === "end") {
				await emit({ type: "agent_end", messages: newMessages });
				return;
			}

			explicitContinuation = decision?.action === "continue";
			pendingMessages = (await config.getSteeringMessages?.()) || [];
			if (hasMoreToolCalls || pendingMessages.length > 0) {
				explicitContinuation = false;
			}
		}

		// 内层退出后的分支：steering（模型正在干活的时候用户输入）；follow-up（模型快停止时来消息，作为新输出重开一轮）
		const followUpMessages = (await config.getFollowUpMessages?.()) || [];
		if (followUpMessages.length > 0) {
			explicitContinuation = false;
			pendingMessages = followUpMessages;
			continue;
		}

		if (explicitContinuation) {
			explicitContinuation = false;
			continue;
		}

		break;
	}

	await emit({ type: "agent_end", messages: newMessages });
}
```

通过读取模型返回的结构化语言决定`toolCall`



### 工具调用

模型只负责生成`toolCall`，后续pi的程序负责处理

同样分为多个阶段：`execteToolCalls()` - `prepareToolCall()` - `executePreparedToolCall()` - `finalizeExecuteToolCall()` - `createToolResultMessage()`

- executeToolCalls()

决定工具执行策略，并行/串行

```typescript
async function executeToolCalls(
): Promise<ExecutedToolCallBatch> {
	const toolCalls = assistantMessage.content.filter((c) => c.type === "toolCall");
  
  //executionMode是工具自身的执行要求，比如read就是并行，而write就是串行
	const hasSequentialToolCall = toolCalls.some(
		(tc) => currentContext.tools?.find((t) => t.name === tc.name)?.executionMode === "sequential",
	);
	if (config.toolExecution === "sequential" || hasSequentialToolCall) {
		return executeToolCallsSequential(currentContext, assistantMessage, toolCalls, config, signal, emit);
	}
	return executeToolCallsParallel(currentContext, assistantMessage, toolCalls, config, signal, emit);
}
```

使用批次进行穿并行判断

- prepareToolCall()

进行执行前检查

```typescript
async function prepareToolCall(...): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
  // 找到对应工具
	const tool = tools.find((t) => t.name === toolCall.name);
	if (!tool) {
		return {
			kind: "immediate",
			result: createErrorToolResult(`Tool ${toolCall.name} not found`),
			isError: true,
		};
	}

	try {
    // 准备参数
		const preparedToolCall = prepareToolCallArguments(tool, toolCall);
    // schema校验，校验参数 
		const validatedArgs = validateToolArguments(tool, preparedToolCall);
    
    // 执行工具前，调用外部逻辑进行检查
    if (config.beforeToolCall) { ... }
		...
	} catch (error) {}
}
```

- executePreparedToolCall()

```typescript
async function executePreparedToolCall(...): Promise<ExecutedToolCallOutcome> {
	...
	try {
    // 执行工具
		const result = await prepared.tool.execute(
			prepared.toolCall.id,
			prepared.args as never,
			signal,
			(partialResult) => {
				if (!acceptingUpdates) return;
        // 工具持续反馈执行进度
				updateEvents.push(Promise.resolve(onUpdate(partialResult)));
			},
		);
    ...
	} catch (error) {
		const durationMs = elapsed();
		acceptingUpdates = false;
		await Promise.all(updateEvents);
		return {
			result: createErrorToolResult(error instanceof Error ? error.message : String(error)),
			isError: true,
			durationMs,
		};
	} finally {}
}
```

调用工具，并且之后的`finalizeExecuteToolCall()`能够处理最终结果，包括调整返回内容、补充数据等





### 上下文管理

pi的上下文思路是，保存完整的探索过程，但是只把历史交给模型

文件内容在：`packages\coding-agent\src\core\session-manager.ts`、`packages\coding-agent\src\core\compation\compations.ts`

比如agent执行了复杂任务，调用了多次工具，会出现一些问题：

上下文窗口、探索过程分支、任务持久化

整体的探索过程保存在树状节点中

- SessionEntryBase

```typescript
export interface SessionEntryBase {
	type: string;
	id: string;									// 保留ID
	parentId: string | null;		// 保留父节点ID
	timestamp: string;
}
```

允许历史记录回到父节点，而且采用追加式，创建新分支不需要删除旧分支的历史

- buildSessionPath()

```typescript
function buildEntryIndex(entries: SessionEntry[], byId?: Map<string, SessionEntry>): Map<string, SessionEntry> {
	if (byId) return byId;
  // 建立ID索引
	const index = new Map<string, SessionEntry>();
	for (const entry of entries) {
		index.set(entry.id, entry);
	}
	return index;
}

function buildSessionPath(...): SessionEntry[] {
	const index = buildEntryIndex(entries, byId);
  // 找到当前叶节点
	let leaf: SessionEntry | undefined;
	...
  // 追踪到父节点，翻转路径
	const path: SessionEntry[] = [];
	let current: SessionEntry | undefined = leaf;
	while (current) {
		path.push(current);
		current = current.parentId ? index.get(current.parentId) : undefined;
	}
	path.reverse();
	return path;
}
```

采取这种机制，Session和Context是两种。Session存储整体的对话记录、所有的分支结构等内容；之后的Branch存储当前叶节点到根节点到历史路径；最后的Context是经过筛选和压缩、实际参与模型推理的消息，给LLM调用



### 压缩上下文

pi的默认策略是在剩余小于16384的token时压缩，默认保留20000的最近token不压缩

- CompationEntry

```typescript
export interface CompactionEntry<T = unknown>
  extends SessionEntryBase {
  type: "compaction";
  // 早期历史摘要
  summary: string;
  // 从哪个历史节点开始保留原始消息
  firstKeptEntryId: string;

  tokensBefore: number;
  details?: T;
  usage?: Usage;
  fromHook?: boolean;
  systemMessage?: SystemMessage;
}
```

- buildContextEntries()

```typescript
export function buildContextEntries(...): SessionEntry[] {
  // 获取当前分支
	const path = buildSessionPath(entries, leafId, byId);
  // 找到最近一次压缩
	let compaction: CompactionEntry | null = null;
	for (const entry of path) {
		if (entry.type === "compaction") {
			compaction = entry;
		}
	}
	// 没有压缩就原样返回
	if (!compaction) {return path;}
	...
	const contextEntries: SessionEntry[] = [compaction];
  // 保留近期消息
	let foundFirstKept = false;
	for (let i = 0; i < compactionIdx; i++) {
		const entry = path[i];
		if (entry.id === compaction.firstKeptEntryId) {
			foundFirstKept = true;
		}
		if (foundFirstKept && !(entry.type === "message" && entry.message.role === "system")) {
			contextEntries.push(entry);
		}
	}
  // 拼接压缩后的历史
	contextEntries.push(...path.slice(compactionIdx + 1));
	return contextEntries;
}
```

输出了结构化的语义摘要，从而替代旧历史，并且有恢复坐标



### 扩展机制

在不修改Pi核心代码的情况下，添加新的工具、记忆机制等内容

pi的扩展机制实现了提供新功能，而且介入agent的执行生命周期，在指定的执行阶段运行

示例结构：

```typescript
import type {
  ExtensionAPI
} from "@earendil-works/pi-coding-agent";
// 向外导出一个函数，pi加载模块后可以调用函数来初始化拓展
// ExtensionAPI 是扩展接口，能够不直接获取loop中的变量就完成一些操作
export default function (pi: ExtensionAPI) {
	// on是事件注册，意思是在agent_start时，调用提供的函数
  pi.on("agent_start", async () => {
    console.log("Agent started!");
  });

  pi.on("agent_end", async () => {
    console.log("Agent finished!");
  });
}
```

- loadExtension()

```typescript
async function loadExtension(path): Promise<{ extension: Extension | null; error: string | null }> {
  
	try {
    // 返回模块的默认的工厂函数
		const factory = await loadExtensionModule(resolvedPath, cacheToken);
		time(`${extensionPath} module import`, "extensions");
		// 初始化
		const extension = await initializeExtension(factory, extensionPath, resolvedPath, cwd, eventBus, runtime);

		return { extension, error: null };
	} catch (err) {
		const message = err instanceof Error ? err.message : String(err);
		return { extension: null, error: `Failed to load extension: ${message}` };
	}
}
```

这部分负责初始化注册的能力，实际上包括处理资源加载、扩展路径、扩展上下文等细节

- runner.ts

我们需要区分，`event`是agent在运行过程中发生的事情，而`hook`是允许外部代码在特定时机执行、并且可能会影响原始流程。比如能够在调用工具、新的一轮推理时执行

举例hook

```typescript
	async emitToolCall(event: ToolCallEvent): Promise<ToolCallEventResult | undefined> {
		// 举例在调用工具前添入执行流程
		for (const { handlers } of snapshotEventHandlers(this.extensions, "tool_call")) {
			for (const handler of handlers) {
				const handlerResult = await handler(event, ctx);
				if (handlerResult) {
					result = handlerResult as ToolCallEventResult;
					if (result.block) {
						return result;
					}
				}
			}
		}
		return result;
	}
```

- registerTool()

除了能够注册hook功能，同时注册工具，在LLM需要的时候调用







## 实例构建

我们以ctf比赛为例，构件一个基于pi的多agent协作框架

### 创建实例

pi提供给其他程序有调用的类和函数

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,	
  ModelRuntime,						// 模型管理能力
  SessionManager,
  SettingsManager,
  type AgentSession
} from '@earendil-works/pi-coding-agent';
```

获取这些内容，使用ts可以创建一个pi实例，同时能够对pi进行调整，包括其生命周期等封装为接口。

### 使用实例

对于实例，接下来需要在其中注入prompt、组织任务等，使用了如下流程

```typescript
private async execute(e: Executor, direction: string) {
  // 1. 确定预算、工作目录、会话目录
  // 2. 创建 Pi agent
  e.agent = await this.factory({
    role: e.role,
    system: ...,
    tools: ...,
    onTurn: ...,
    onEvent: ...,
    onCompact: ...,
  });
  // 3. 组织首次任务
  let prompt = ...;
  // 4. 让 agent 执行；必要时继续 prompt
  while (...) {
    await e.agent.prompt(prompt);
    ...
  }
}
```

factory就是我们创建的pi实例，具体的调用交给上一步；

不同的角色，可能prompt不同、预算和来源不同；

这里加入了控制预算功能，以轮次进行控制，比如每道题目只执行50次循环

```typescript
onTurn: async (turn: Turn) => {
  if (!valid()) return true;
  failedTurn = turn.outcome !== 'completed';

  if (turn.outcome === 'completed') {
    e.used++;

    if (e.role === 'worker' && e.used >= budget) {
      this.close('turn_budget');
    }
  }
  // 把完整 turn 写入文件，记录活动和执行状态
  // 根据活动情况，可能触发 Watcher
  return this.closing || e.used >= budget;
},
```

在底层的实例完成一次循环时，return中也加入onTurn，便于计数

### 工具封装

对于工具的调用，我的想法是封装为调用接口，通过参数来调用

```
export const tools: ToolFactory[] = [ ... ];
```

工具的定义同样适用完整的factory进行定义

```typescript
export interface ToolFactory {
  name: string;
  roles: readonly Role[];
  enabled?(config: Config): boolean;
  allowDuringClosing?: boolean;

  create(
    context: ToolContext,
    services: ToolServices
  ): ToolDefinition;
}
```

定义的工具同样以一个固定的结构给模型，开始的时候就能够获取全量的工具

run_commander应该注册在shell / docker中





### 信息共享

多agent需要实现信息共享机制

```typescript
await session.sendCustomMessage(
  {
    customType: 'blackboard-update',
    content: text,
    display: false,
  },
  {
    triggerTurn: false,
  }
);
```

准备好消息后，可以在运行时注入

```typescript
await session.steer(
  '新的共享信息：登录接口需要使用另一种参数，请优先验证。'
)y
```







































































