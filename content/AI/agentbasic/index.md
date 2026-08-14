---
title: "Agent mechanism basic"
date: 2026-08-10T20:00:00+08:00
draft: false
---

目标是学习Agent机制

## LLM到Agent

从纯语言模型LLM到智能体Agent，本质是从只读的思考到可行动的执行之间的转变

基础的LLM是一个概率预测器，通过输入N个token，预测N+1个token，并没有对话的概念，只会接着上文向下写

**SFT(Supervised Fine-Tuning)**

指令微调，为了将预训练的知识转化为解决问题的能力，教会大模型如何听懂人类的问题，并且给出期望的格式来回答

见效较快，但过度微调可能导致模型忘记通用知识，且只会模仿

**RLHF**

基于人类反馈的强化学习：在SFT模型上生成多个回答，由人工排序并训练奖励模型，再用PPO优化，并以KL约束防止偏离

之后给LLM加上了tool，模型学会了通过结构化的指令，通过在外部环境执行后返回，得到了交互能力

在工具的基础上，引入系统化的架构设计，通常包含：

```
LLM + Planning + Memory + Tool Use
```

## Tool Use

我们的目标是让LLM仅依靠自然语言文本，表达工具调用意图，并被外部程序解析

### Prompt-based范式

原来是通过在prompt中告诉模型有哪些工具，以及工具的入参格式；外部的代码通过正则或者字符串解析去执行真正的函数，结果拼回prompt

有XML或ReAct范式，类似：

```Re
你是一个具备工具调用能力的助手。请按以下格式思考和解答：

Question: 用户提出的问题
Thought: 思考当前应该做什么
Action: 要调用的工具名称，必须是 [get_weather, web_search] 之一
Action Input: 工具的输入参数，必须是合法 JSON
Observation: 工具执行后的结果（由环境填入，你不需要生成）
```

这样的方式通用性比较高，但是token消耗和格式都是问题

### Native Function Calling

原生函数调用，模型在训练阶段（SFT）就已经学习到了识别工具接口并输出特定结构化指令的能力，同时在推理阶段配合采样层的限制机制，确保生成文本符合规范

**Special tokens **

大模型的词表中除了日常语言的token，还包含了一部分不对用户展示的控制标记，可以标记工具调用指令的开始、结束等

**SFT**

在模型的微调阶段，训练数据构建成特定的格式，当模型意识到用户需要外部数据且存在对应工具时，需要调用`tool_call`

不会把工具调用和结束条件当作普通文本回复

在调用工具的时候稳定性和效率得到了极大的提升

### **JSON Schema**

向AI声明工具时，工具的入参是严格遵循JSON Schema规范，类似：

```
{
  "name": "search_logs",
  "description": "查询服务器系统日志，用于排查错误",
  "parameters": {
    "type": "object",
    "properties": {
      "service_name": {
        "type": "string",
        "description": "服务名称，例如 auth-service 或 payment-service",
        "enum": ["auth-service", "payment-service", "user-service"]
      },
      "level": {
        "type": "string",
        "description": "日志级别，例如 INFO 或 ERROR",
        "enum": ["INFO", "WARN", "ERROR"]
      }
    },
    "required": ["service_name", "level"]
  }
}
```

通过语法约束，强制保证生成的token符合类型定义

### **约束解码控制**

结合JSON Schema可显著减少语法错误，比如该生成`}`时却生成了`]`

系统拿到一个JSON Schema后，会在内存中构建状态转移图，用一个FSM确保按约束逐步采样

**FSM**

有限状态自动机，可以理解为根据当前状态和输入，确定下一步转移的规则

**约束生成**

通过FSM约束，大模型能生成符合语法的格式，再用于工具调用



## Planning

本质就是LLM通过特定的prompt范式、控制流结构与状态管理，实现多步的任务拆解、工具调用以及异常修正

### ReAct范式

ReAct范式是基础的控制流范式，把LLM的推理过程和外部工具调用交织在同一个循环中

**控制流**

单次循环由以下三个步骤组成

- Thought：结合初始目标和上下文，评估当前状态，推理下一步行动
- Action：LLM输出特定格式的指令，指定调用的外部工具和参数
- Observation：系统捕获Action，返回的结果写回上下文

在实现ReAct时，要严格约束模型的输出格式和终止条件，直到输出符合约束并满足终止条件

ReAct的优点在于实时反馈，如果返回错误信息，就能够识别报错并修正Action

局限性也很大：易陷入局部最优、上下文膨胀、复杂场景下效率低

### Plan-and-Solve范式

核心思想是将Planning和Execution解耦为两个模块

**控制流**

- 规划阶段：planner接收用户的复杂目标，一次性生成完整的子任务列表
- 执行阶段：executor严格按照子任务列表的顺序，逐个调用工具或进行推理，更新任务状态
- 输出阶段：所有的子任务结果聚合，返回给用户

prompt也需要约束为JSON或Markdown格式

缺点也明显：如果初始planning存在缺陷，后续执行也会偏差



### Tree of Thoughts范式

对于包含复杂的逻辑推理、或者多约束条件决策的复杂任务，单线推理与简单列表规划均无法提供回溯能力

ToT将模型的推理过程建模为状态空间树上的搜索问题

**核心组件**

- 思维生成器：给定当前状态节点s，生成k个可能的下一步思维z
- 状态评估器：评估当前状态节点s对实现最终目标的价值
- 搜索算法：控制在思维树的遍历逻辑，如BFS、DFS
- 回溯机制：当搜索到终点，分数低于阈值或无效，丢弃当前分支，回溯到父节点重新探索

适合搜索空间大、约束复杂、需要试错的推理任务

### State Machine 状态机策略

通过将控制流显式化，为Agent提供结构化控制与运行时容错

将Agent抽象为有向图（允许环路）

- 状态（State）：state是贯穿整个图的全局数据结构，包含业务状态的集合，如：

```
class AgentState(TypedDict):
    input_goal: str               # 用户初始目标
    plan: List[str]               # 当前正在执行的步骤列表
    current_step_index: int       # 当前执行到了第几步
    step_results: Dict[int, Any]  # 各步骤的执行结果
    error_logs: List[str]         # 工具调用的异常记录
    retry_count: int              # 当前步骤的重试次数
    is_finished: bool             # 终止标志位
```

- 节点：节点是纯函数或包含LLM调用的异步操作，接收当前state，执行特定任务，仅返回需要更新或追加的状态增量
- 转移边：普通边就将执行流从A转移到B

条件边：路由函数根据state中字段的值判断跳转方向

**动态重规划策略**

当工具调用因网络超时、参数错误等问题失败时，动态重规划确保Agent不会崩溃，具有一定的自我修正路径能力

- 被动触发：工具明确返回error或exception，状态机捕获，重新规划评估剩余步骤
- 主动触发：通过`Evaluator`节点利用LLM检测当前的数据是否真正解决了当前问题，如果数据不达标，仍然判定为失败

重规划节点的prompt同样需要裁剪上下文

**状态持久化**

每个节点执行完成后，系统自动将state序列化写入Redis等，中途退出继续读取运行

生产级的Agent通常都建立在状态机框架上，加上强约束ReAct循环的混合体



## Multi Agent

LLM配合工具，对于复杂任务会出现上下文膨胀、角色冲突、错误累积等问题

多Agent之间交互也需要强类型的结构化数据，包括一些核心字段

```
trace_id	链路追踪ID，贯穿任务的生命周期
control_intent	交互信息
TASK_EXECUTE	要求执行任务。
TASK_RESULT	返回执行结果。
TASK_ERROR	返回错误与异常堆栈。
```

### 架构类型

**层级制架构**

采用一种集中控制、分布式执行的结构，上层负责决策和调度，下层负责具体执行

- 调度节点：不直接调用工具执行任务，仅管理控制流
- 执行节点：只负责特定领域的任务执行，挂载限定范围的工具

**点对点拓扑**

不存在全局的调度节点，所有agent平等，通过消息总线进行信息传递

**共享内存与同步**

多Agent中，状态同步能够保证各节点数据的一致性，同时防止上下文过长

针对短时内存就在context内部，全局共享状态就在Redis等中，长时记忆在向量数据库中。



## Memory

memory机制本质上是系统管理、过滤、持久化与检索上下文的控制层

如果把所有历史记录全部提交LLM，会造成上下文的溢出

### working memory

运行时状态，作为Agent运行期间保存状态的一个数据结构，例如

```
class WorkingMemory(BaseModel):
    run_id: str                      # 当前运行的唯一 ID
    user_id: str                     # 用户 ID
    session_id: str                  # 当前会话 ID
    task_goal: str                   # 用户当前要解决的最终目标
    current_plan: List[str]          # 当前拆解的步骤清单 ["步骤1", "步骤2"]
    current_step_index: int          # 正在执行第几步
    last_action: Optional[Dict]      # 上一步调用的工具及参数
    last_observation: Optional[str]  # 上一步工具返回的原始结果
    token_count: int                 # 当前上下文已经占用的 Token 数
```

### short-Term memory

会话级历史缓存，在多轮对话中保持上下文连贯，在同一个session中，跨run，使用Redis等内存数据库

```
[
  {
    "role": "user",
    "content": "帮我分析下订单表查询慢的原因",
    "timestamp": 1773129600
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "id": "call_001",
        "name": "execute_sql",
        "args": {"query": "EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;"}
      }
    ]
  }
]
```

支持过期机制与滑动窗口压缩

### Long-Term Memory

长期记忆，解决跨session、跨任务的信息持久化，明确分为：事实数据库和经验向量库，使用数据库存储

**事实数据库**

负责语义记忆，保存用户、系统环境或业务规则的明确事实，例如

```
CREATE TABLE user_semantic_memory (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL,
    category VARCHAR(32) NOT NULL,     -- 事实分类: 'preference', 'environment', 'business_rule'
    fact_key VARCHAR(128) NOT NULL,    -- 事实键: 'preferred_language', 'db_type'
    fact_value TEXT NOT NULL,          -- 事实值: 'Python', 'PostgreSQL 15'
    confidence FLOAT DEFAULT 1.0,      -- 置信度 (0.0 - 1.0)
    source_session_id VARCHAR(64),     -- 来源会话
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT unique_user_fact UNIQUE(user_id, fact_key)
);
```

**经验向量库**

负责情节记忆，保存曾经解决过的完整任务案例与错误反思，以便复用经验

```
{
  "id": "trace_doc_9981",
  "vector": [0.0124, -0.0451, 0.0892, "...1536维 Embedding 向量..."], 
  "payload": {
    "task_goal": "解决 PostgreSQL 大表查询超时问题",
    "environment": "PostgreSQL 15 / Linux",
    "success": true,
    "execution_trace": [
      {"step": 1, "action": "EXPLAIN ANALYZE", "result": "发现 Seq Scan 全表扫描"},
      {"step": 2, "action": "CREATE INDEX", "result": "创建 user_id 索引后查询降至 5ms"}
    ],
    "reflection": "此类超时主因通常是缺少索引，应优先检查执行计划，而不是盲目调大 statement_timeout",
    "created_at": "2026-03-01T10:00:00Z"
  }
}
```

**检索**

记忆需要通过特定的关键词、向量等混合检索，找到得分最高的Top-K条记忆给LLM

之后甚至可以通过总结反思机制，把解决过程总结为skill，不过当前智能体对skill的沉淀与复用能力仍不成熟





