---
title: "Cairn项目学习"
date: 2026-09-11T20:00:00+08:00
draft: false
---

## 基础概念

学习开源多agent项目

```
https://github.com/oritera/Cairn
```

底层思想跳出了传统的Multi-Agent预设固定角色+聊天轮询的局限

### 状态空间搜索

传统的LLM对话中，任务往往被视作线性的流水线或者状态机（FSM），对于渗透测试、漏洞挖掘、逆向工程等：高度不确定性、开放边界的任务，固定流程往往会效果较差

可能性陷入上下文膨胀、局部最优陷阱

Cairn将解决问题的过程建模为**图遍历**问题

形式化表示为一个四元组
$$
S = \langle S_0, G, A, T \rangle
$$

```
S0:掌握的起点，如目标的IP地址、一个二进制文件等
G :目标状态，要达到的断言，如获取flag
A :动作集合，可以对环境施加的操作，如端口扫描、尝试越权
T :转移函数，在状态S执行动作a后，观察到的新状态S'
```

状态空间显示化为图，探索意味着：**图的分支**

探索失败只是当前分支终止

### 黑板架构

Multi-Agent架构倾向于：角色扮演 + 轮询对话的方式，如侦察者向分析者汇报

缺点：通信路径固定，漏掉某个角色的信息会卡住，每个agent的信息私有，可能会幻觉或者丢失

重构的黑板机制：个体之间不直接通信，而是通过改变共享环境间接协调

有三种知识：

```
- Facts:已验证客观事实，只增不改
- Intents:待验证候选假设
- Hints:人类介入的先验知识
```

黑板架构由三部分构成

- **黑板**

系统的唯一真实源，在Cairn中是维护一张有向无环图

worker不私藏状态，历史证据结构化的方式贴在黑板上

- **知识源**

  不需要在意谁放入了数据

  - Reason worker：审视黑板现状，基于Fact，有哪些可能性，写入新的Intent
  - Explore worker：选择Intent，带执行目标去运行工具

- **控制组件**

控制下一周期激活那个知识源，不会抢占任务

当一个Intent被worker完成，会成为一个新的fact

整体来看，人类给定起点和最终目标，通过推理扩散和探索收敛，最终逐渐状态演进



## 项目架构

### Cairn Server

Cairn Server是一个FastAPI服务 + SQLite数据库

功能是：持久化并对外提供`projects / facts / intents / hints`四类数据的读写

- /src/cairn/server/cli.py	启动流程

```python
def serve(host: str, port: int, db_path: str, log_level: str, access_log: bool):
    """Start the Cairn API server."""
    # 建立数据库
    db.configure(Path(db_path))
    from cairn.server.app import app

    # 启动http服务
    uvicorn.run(
        app,
        host=host,
        port=port,
        log_level=log_level.lower(),
        access_log=access_log,
    )
```

- /src/cairn/server/app.py	FastAPI应用，注册了五个业务端点

```python
app.include_router(settings.router) 
app.include_router(projects.router)
app.include_router(hints.router)
app.include_router(intents.router)
app.include_router(export.router)
```

- /src/cairn/server/db.py	SQLite存储实现

代码太多了，就是导入sql，如果不存在就执行代码创建，注册了表结构

- /src/cairn/server/models.py	

定义了程序内部的对象结构和http请求/响应

```python
# Fact：客观事实节点类
# id是标识符，description是成立的客观事实陈述
class Fact(BaseModel):
    id: str
    description: str
```

```python
# Intent：探索意向拓扑类
# from是前驱依赖的Fact，to是产出的Fact
# worker&last_heartbeat_at：分布式租约机制，worker表示真在执行此意图的worker ID
# created_at & concluded_at：生命周期标记
class Intent(BaseModel):
    id: str
    from_: list[str] = Field(alias="from")
    to: str | None = None
    description: str
    creator: str
    worker: str | None = None
    last_heartbeat_at: str | None = None
    created_at: str
    concluded_at: str | None = None

    model_config = {"populate_by_name": True}
```

```python
# Hint:人类线索
# content表示注入的内容
class Hint(BaseModel):
    id: str
    content: str
    creator: str
    created_at: str
```

另外，还包括项目状态视图，包括：

```
ProjectReaason	推演worker互斥
ProjectDetail	黑板全景打包
ProjectMeta		项目基础状态
```

- /src/cairn/server/services.py

为上层的http接口提供了底层支持，封装了sql语句等

```python
# 生成短ID
def next_project_id
# 检查项目状态
def check_project_active
# 检查数据有效性
def validate_facts_exist
# 任务分配与状态提取
def get_intent_or_404
# 心跳检查与状态清理
def get_intent_timeout
```

- /src/cairn/server/routers/projects.py

提供整个项目项目完整生命周期的HTTP接口，能够查询项目详情、创建项目、全局锁机制、项目终结和重启

- /src/cairn/server/routers/intents.py

作为探索任务的微观生命周期的HTTP接口，能够创建意图intents、上报心跳、认领/释放意图，结项

比如：释放intents

```python
def release(project_id: str, intent_id: str, body: HeartbeatRequest):
    with get_conn() as conn:
        check_project_active(conn, project_id)
        row = get_releasable_open_intent_or_404(conn, project_id, intent_id, body.worker)

        if row["worker"] == body.worker:
            conn.execute(
                "UPDATE intents SET worker = NULL WHERE id = ? AND project_id = ?",
                (intent_id, project_id),
            )
            row = conn.execute(
                "SELECT * FROM intents WHERE id = ? AND project_id = ?",
                (intent_id, project_id),
            ).fetchone()

        return intent_to_model(conn, row, project_id)
```

都是通过sql操作的

总体来看，Server没有调用agent，只负责：接收http请求 - 校验 - 读写sql -  返回json，可以理解为作为**状态寄存器**的功能

### Dispatcher

Dispatcher是独立运行的守护进程：周期性地向Server查询当前图的状态，根据预设的规则做判断，启动真实的worker去干活，最后的结果写回server

**任务载荷**

```
bootstrap :项目创建的任务，只有origin和goal，快速初探
reason    :全局研判，黑板上只有fact，没有intent，批量写入intent
explore   :具体探索，黑板上空闲代办的intent，产出fact
```

- /src/cairn/cli.py

启动入口，加载`dispatch.yaml`(配置文件)，构造DispatcherLoop跑循环

```python
    loop = DispatcherLoop(config_path)
    try:
        if startup_healthcheck_only:
            loop.run_startup_healthchecks_only()
            return
        loop.run(once=once)
```

- /src/cairn/dispatcher/config.py

```python
# 确认不同任务种类的超时
class ReasonTaskConfig(BaseModel)
# 运行时调度策略配置
class RuntimeConfig(BaseModel)

# worker规格
class WorkerConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")   # 禁止多传未定义的额外字段

    name: str                                   # Worker 标识名称
    type: WorkerType                            # claudecode / codex / pi / mock
    task_types: list[TaskType]                  # 该 Worker 允许接哪些任务 (如 ["reason", "explore"])
    max_running: int = Field(gt=0)              # 该类 Worker 最大并发运行数
    priority: int = Field(ge=0)                 # 调度优先级 (数值越大越优先分配)
    env: dict[str, str] = Field(default_factory=dict) # 独立的特定环境变量
```

最后支持mock测试，即不接入大模型测试能否跑通

- /src/cairn/dispatcher/protocol/client.py

Dispatcher和Server之间的唯一通道

```python
# 定义返回对象
class ApiResult
# 使用多线程管理
class CairnClient
# 读取状态与配置，使用pydantic转换成强类型对象
def list_projects(self) -> list[ProjectSummary]
```

总体上，server中的服务端接口，封装为了python方法调用

- /src/cairn/dispatcher/scheduler/loop.py

核心循环类，在每一个周期循环执行：

```python
  self._reap_futures()              # 回收已完成的任务
  self._reap_cleanup_futures()      # 回收已完成的容器清理
  summaries = self.client.list_projects()
  self._initialize_reason_checkpoints(summaries)
  self._refresh_runtime_projects(summaries)
  self._cancel_inactive_tasks(summaries)   # 非 active 项目硬停止
  self._queue_container_cleanups(summaries)
  self._dispatch_available(summaries)      # 核心：派发
  time.sleep(self.config.runtime.interval)
```

调度入口为`_dispatcher_available()`

```python
# 在判断项目不满，且active
# 先轮询运行中的项目，再启动空闲项目
            for summary in idle_projects:
                if self._running_project_count(active) >= self.config.runtime.max_running_projects:
                    self._log_changed(
                        "dispatch/idle-limit",
                        logging.INFO,
                        "stop idle project dispatch because max_running_projects reached running_projects=%s",
                        self._running_project_count(active),
                    )
                    return
                if self._try_dispatch_project(summary):
                    dispatched = True
                    break
```

每个项目都会调用`_try_dispatch_project()`

```python
# 进行reason,bootstrap,explore三类任务的派发
if self._project_requires_bootstrap(project):
     return self._dispatch_initial_project(project)
```

- /src/cairn/dispatcher/tasks/reason.py

任务文件的结构基本一致，以`reason.py`为例

定义了任务在子线程的具体执行闭环，全局研判

确保沙盒和环境可用的情况下：

```python
# 把fact-intent图谱喂给大模型
open_intents = [
            {
                "id": intent.id,
                "from": intent.from_,
                "description": intent.description,
                "worker": intent.worker,
            }
            for intent in project.intents
            if intent.to is None
        ]
        allowed_fact_ids = [fact.id for fact in project.facts if fact.id != "goal"]
        prompt = render_prompt(
            load_prompt(config.runtime.prompt_group, "reason.md"),
            {
                "graph_yaml": write_graph_snapshot_reference(
                    container_manager,
                    container_name,
                    export_yaml.strip(),
                    phase="reason_execute",
                ),
                "fact_ids": format_fact_ids(allowed_fact_ids),
                "open_intents": format_open_intents(open_intents),
                "max_intents": str(config.tasks.reason.max_intents),
            },
        )
```

大模型执行完之后，会强制要求其结构符合JSON规范

```python
		try:
            model_output = driver.extract_response_text(result.stdout, result.stderr)
            payload = parse_json_output(model_output)
            kind, data = validate_reason_payload(
                payload, open_intents_empty=not open_intents, max_intents=config.tasks.reason.max_intents,
            )
        except Exception as exc:
            return "failed"
```

之后返回处理逻辑`if kind == "intents":`等，对应不同的处理方法

- /src/cairn/dispatcher/tasks/explore.py

定义单个任务的执行逻辑与结果沉淀，具有很强的鲁棒性，微观执行

一、任务初始化与心跳

```python
    driver = get_driver(worker.type, config.runtime.execution)
    task_started = time.perf_counter()
    healthcheck_timeout = config.runtime.healthcheck_timeout
    lease = HeartbeatLease.for_intent(client, project.project.id, intent.id, worker.name, config.runtime.interval)
    lease.start()
    try:
        container_name = container_manager.ensure_running(project.project.id)
```

根据配置获取对应到Driver，并确保当前的沙箱就绪；

定时发送heartbeat，确保可行

二、探索promt注入

```python
        prompt = render_prompt(
            load_prompt(config.runtime.prompt_group, "explore.md"),
            {
                "graph_yaml": write_graph_snapshot_reference(
                    container_manager,
                    container_name,
                    export_yaml.strip(),
                    phase="explore_execute",
                ),
                "intent_id": intent.id,
                "intent_description": intent.description,
            },
        )
```

发送prompt，执行intent_description中的内容

三、确认输出

进入判定流，如果正常输出并且按要求返回了json，提取其中的description

如果返回了rejected或者结构不对，打入冷却标签

四、兜底恢复

防止模型命令执行超时或者JSON输出格式有误，浪费token

利用刚才的session句柄，向模型发送explore_conclude.md模板

总结输出并返回标准JSON

```python
def _try_conclude_fallback(...):
    # 1. 前置保护：必须驱动支持会话复用，且心跳未断、任务未被取消
    if not driver.supports_conclude() or not session:
        best_effort_release(client, project_id, intent.id, worker.name)
        return "failed"
    ...
    # 2. 加载专用总结提示词 explore_conclude.md
    prompt = render_prompt(
        load_prompt(config.runtime.prompt_group, "explore_conclude.md"),
    )
    # 3. 复用同一个 Session，只做结论提取，不再执行重命令
    conclude_argv = driver.build_conclude(worker, prompt, session)
    result = _run_process(..., timeout=config.tasks.explore.conclude_timeout, ...)
```

- /src/cairn/dispatcher/workers/base.py

定义agent的驱动接口，如`driver.build_execute()`等，具体针对每个agent的实现在`/workers/adapters`中

```python
# 参数封装
@dataclass(slots=True)
class DriverResult:
    argv: list[str]
    session: str | None = None
# 驱动核心抽象
# 其中有：能力环境、健康检查、命令指令拼装
class WorkerDriver(abc.ABC):
    type_name: str
```

**总结**

Dispatcher负责推进整个状态空间搜索引擎与调度



### Container

项目容器，作为worker/agent的执行环境，每个项目对应一个独立的运行容器，Dispatcher在这个环境里启动Agent进程

scheduler\loop.py决定在本地或者docker容器中执行命令

- /src/cairn/dispatcher/runtime/backend.py

dispatcher调用的接口

- /src/cairn/dispatcher/runtime/containers.py

每个项目一个docker容器

```python
class ContainerManager:
    _PREFIX = "cairn-dispatch-"

    def __init__(self, config: ContainerConfig):
        self._config = config
        self._client = docker.from_env()
        self._ensure_running_locks: dict[str, threading.Lock] = {}
        self._ensure_running_locks_guard = threading.Lock()
```

创建docker，等待dispatcher往里面exec

- /src/cairn/dispatcher/runtime/process.py

执行每一个命令的封装

- /src/cairn/dispatcher/runtime/local_backend.py

不建容器，直接在本地运行命令



### worker

只负责接受dispatcher渲染好的prompt，在容器/本地运行后，输出一段结构化json

- src\cairn\dispatcher\workers\base.py & health.py

接口和探活

- src\cairn\dispatcher\workers\adapters\claudecode.py

调用claudecode

- src\cairn\dispatcher\workers\adapters\pi.py

输出json事件流













question

```
沙箱
session

```

