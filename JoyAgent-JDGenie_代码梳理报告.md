# JoyAgent-JDGenie 系统性代码梳理报告

> **项目定位**: 业界首个开源高完成度轻量化通用多智能体产品  
> **项目地址**: https://github.com/jd-opensource/joyagent-jdgenie  
> **报告日期**: 2026-02-08  

---

## 目录

- [模块一：架构全景图 (10,000 英尺视角)](#模块一架构全景图-10000-英尺视角)
- [模块二：核心抽象与生命周期 (1,000 英尺视角)](#模块二核心抽象与生命周期-1000-英尺视角)
- [模块三：关键流程深度剖析 (100 英尺视角)](#模块三关键流程深度剖析-100-英尺视角)
- [模块四：实践与洞察 (落地视角)](#模块四实践与洞察-落地视角)

---

## 模块一：架构全景图 (10,000 英尺视角)

### 1. 核心类比

> 这个多智能体系统就像一家**现代化的咨询公司**：
>
> - **前台接待 (`ui/`)** 是客户的交互界面，接受客户诉求并实时展示工作进展
> - **项目经理 (`genie-backend/`)** 是公司的核心大脑 — 他接到客户需求后，决定该用"快速响应"(ReAct) 还是"先规划再执行"(Plan-Solve) 的工作模式，拆分任务、分配给不同的专业团队
> - **专业团队 (`genie-tool/`)** 是公司的各个技术部门 — 包括代码工程师(CodeInterpreter)、市场调研员(DeepSearch)、报告撰稿人(Report)、数据分析师(DataAnalysis)等，每个部门独立运作，接受项目经理的调度
> - **外部顾问对接处 (`genie-client/`)** 是公司与外部专家沟通的桥梁，通过 MCP 协议对接第三方工具服务（如 12306 查票、天气查询等）
> - 整个公司的**通信系统是 SSE 流式传输** — 保证客户能实时看到每一步工作进展，而非等到最终报告

### 2. 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         用户浏览器 (localhost:3000)                       │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ SSE / REST
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ui/ (React 19 + TypeScript + Vite 6)                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐    │
│  │ ChatView │  │ ActionView│  │ Dialogue │  │ querySSE.ts (SSE通信) │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ SSE POST /web/api/v1/gpt/queryAgentStreamIncr
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              genie-backend/ (Java 17 + Spring Boot 3.2.2)               │
│                              端口: 8080                                  │
│  ┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐     │
│  │GenieController│───▶│AgentHandlerFactory│───▶│ ReactHandlerImpl │     │
│  │  /AutoAgent   │    │  (策略选择)       │    │ PlanSolveHandler │     │
│  └───────────────┘    └──────────────────┘    └──────────────────┘     │
│           │                                            │                │
│           ▼                                            ▼                │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │           Agent 核心框架 (com.jd.genie.agent)                │       │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │       │
│  │  │BaseAgent │  │ReActAgent│  │Planning  │  │Executor  │   │       │
│  │  │(基类)    │  │(ReAct)   │  │Agent     │  │Agent     │   │       │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │       │
│  │  ┌──────────────┐  ┌─────┐  ┌──────┐  ┌───────────────┐  │       │
│  │  │ToolCollection│  │ LLM │  │Memory│  │ SSEPrinter    │  │       │
│  │  └──────────────┘  └─────┘  └──────┘  └───────────────┘  │       │
│  └─────────────────────────────────────────────────────────────┘       │
└──────────┬────────────────────────────────┬────────────────────────────┘
           │ HTTP REST                       │ HTTP REST
           ▼                                 ▼
┌─────────────────────────┐    ┌─────────────────────────────────┐
│  genie-tool/ (Python)    │    │  genie-client/ (Python)          │
│  FastAPI 端口: 1601       │    │  FastAPI 端口: 8188               │
│                          │    │                                   │
│  /v1/tool/deepsearch     │    │  /v1/tool/list (列出MCP工具)      │
│  /v1/tool/code_interpreter│   │  /v1/tool/call (调用MCP工具)      │
│  /v1/tool/report         │    │         │                        │
│  /v1/tool/auto_analysis  │    │         │ SSE (MCP协议)          │
│  /v1/tool/nl2sql         │    │         ▼                        │
│  /v1/file_tool/*         │    │  ┌─────────────────────┐        │
└──────────────────────────┘    │  │ 外部MCP Server      │        │
                                │  │ (12306/天气/自定义)   │        │
                                │  └─────────────────────┘        │
                                └─────────────────────────────────┘
```

### 3. 目录结构解析

```
joyagent-jdgenie/
├── genie-backend/          # [Java] 核心调度引擎 - "项目经理"
│   └── src/main/java/com/jd/genie/
│       ├── agent/          # Agent 核心框架层
│       │   ├── agent/      #   Agent 类族 (BaseAgent, ReActAgent, PlanningAgent, ExecutorAgent, SummaryAgent)
│       │   ├── tool/       #   工具系统 (BaseTool 接口, ToolCollection, 各 Tool 实现)
│       │   ├── llm/        #   LLM 封装层 (LLM.java, LLMSettings, TokenCounter)
│       │   ├── dto/        #   数据传输对象 (Message, Memory, Plan, ToolCall)
│       │   ├── enums/      #   枚举定义 (AgentType, AgentState, RoleType)
│       │   ├── printer/    #   输出适配器 (SSEPrinter, LogPrinter)
│       │   └── prompt/     #   Prompt 模板 (PlanningPrompt, ToolCallPrompt)
│       ├── controller/     # REST 入口 (GenieController, DataAgentController)
│       ├── service/        # 业务服务层 (AgentHandlerService, MultiAgent 编排)
│       │   └── impl/       #   策略实现 (ReactHandlerImpl, PlanSolveHandlerImpl, AgentHandlerFactory)
│       ├── config/         # 全局配置 (GenieConfig)
│       └── data/           # 数据层 (JDBC Catalog, NL2SQL 支撑)
│
├── genie-tool/             # [Python] 工具执行引擎 - "专业团队"
│   ├── genie_tool/
│   │   ├── api/            #   FastAPI 路由 (tool.py, file_manage.py)
│   │   ├── tool/           #   工具实现 (code_interpreter, report, deepsearch, nl2sql, auto_analysis)
│   │   ├── model/          #   数据模型 (protocal.py, context.py)
│   │   ├── util/           #   工具函数 (llm_util, prompt_util, file_util)
│   │   └── prompt/         #   YAML Prompt 模板
│   └── server.py           #   FastAPI 入口 (端口 1601)
│
├── genie-client/           # [Python] MCP 协议客户端 - "外部顾问对接处"
│   ├── app/
│   │   └── client.py       #   SSE 客户端 (SseClient)
│   └── server.py           #   FastAPI 入口 (端口 8188)
│
├── ui/                     # [React+TS] 前端界面 - "前台接待"
│   ├── src/
│   │   ├── components/     #   核心组件 (ChatView, ActionView, Dialogue)
│   │   ├── pages/          #   页面 (Home)
│   │   └── utils/          #   SSE 通信 (querySSE.ts)
│   └── package.json
│
├── Genie_start.sh          # 一键启动脚本（启动全部4个服务）
├── check_dep_port.sh       # 依赖与端口检查脚本
└── docs/                   # 文档和图片资源
```

### 4. 技术栈与依赖

| 模块 | 语言/框架 | 核心依赖 | 角色说明 |
|------|-----------|----------|----------|
| **genie-backend** | Java 17 + Spring Boot 3.2.2 | OkHttp 4.9.3, FastJSON/Jackson, MyBatis Plus 3.5.14, H2 Database, Apache Calcite 1.34.0, Qdrant Client 1.10.0 | 核心调度引擎，负责 Agent 编排、LLM 交互、工具分发 |
| **genie-tool** | Python 3.11 + FastAPI | LiteLLM, OpenAI SDK, smolagents (HuggingFace), pandas/numpy/scipy/statsmodels, Jinja2, sse-starlette | 工具执行引擎，提供搜索/代码/报告/数据分析等能力 |
| **genie-client** | Python 3.10+ + FastAPI | mcp 1.9.4 (Model Context Protocol SDK) | MCP 协议代理，桥接外部第三方工具服务 |
| **ui** | React 19 + TypeScript + Vite 6 | Ant Design 5, Tailwind CSS 4, @microsoft/fetch-event-source, ECharts, react-markdown | 前端交互界面，支持实时流式展示 |

---

## 模块二：核心抽象与生命周期 (1,000 英尺视角)

### 1. 最重要的三个抽象

#### 抽象一：`BaseAgent` — 智能体基类

**源文件**: `genie-backend/src/main/java/com/jd/genie/agent/agent/BaseAgent.java`

```java
public abstract class BaseAgent {
    // === 核心属性 ===
    private Memory memory;                   // 对话记忆（List<Message>）
    public ToolCollection availableTools;    // 可用工具集合
    protected LLM llm;                       // 大模型客户端
    protected AgentContext context;          // 共享上下文（跨 Agent 传递）
    private AgentState state;                // 状态机: IDLE -> RUNNING -> FINISHED / ERROR
    private int maxSteps;                    // 最大执行步数（防止无限循环）

    // === 核心方法 ===
    public abstract String step();           // 【模板方法】子类实现单步逻辑
    public String run(String query) {        // 主循环
        while (currentStep < maxSteps && state != FINISHED) {
            step();                          // 调用子类的 step()
        }
    }
    public String executeTool(ToolCall cmd);                      // 执行单个工具
    public Map<String,String> executeTools(List<ToolCall> cmds);  // 并发执行多工具（CountDownLatch）
    public void updateMemory(RoleType, String, ...);              // 更新对话记忆
}
```

**设计意图**: BaseAgent 采用**模板方法模式** — `run()` 定义了所有 Agent 的执行骨架（循环 + 状态管理 + 异常处理），子类只需实现 `step()` 即可。

**继承体系**:

```
BaseAgent (abstract)
├── ReActAgent (abstract) ─── step() = think() + act()
│   ├── PlanningAgent      ── 任务规划：生成/更新执行计划
│   ├── ExecutorAgent      ── 任务执行：调用工具完成子任务
│   └── ReactImplAgent     ── 简单 ReAct：单 Agent 快速响应
└── SummaryAgent           ── 任务总结：汇总结果、提取文件列表
```

#### 抽象二：`BaseTool` — 工具接口

**源文件**: `genie-backend/src/main/java/com/jd/genie/agent/tool/BaseTool.java`

```java
public interface BaseTool {
    String getName();                    // 工具名称（供 LLM function calling 识别）
    String getDescription();             // 工具描述（告诉 LLM "何时该调用我"）
    Map<String, Object> toParams();      // JSON Schema 参数定义
    Object execute(Object input);        // 实际执行逻辑
}
```

**设计意图**: BaseTool 将 "工具的自我描述"（给 LLM 看的元信息）与 "工具的执行能力" 统一在一个接口中，实现**可插拔架构**。新增工具只需实现 4 个方法 + `toolCollection.addTool()` 注册即可。

**内置工具一览**:

| 工具类 | 名称 | 说明 | 对应 genie-tool 端点 |
|--------|------|------|---------------------|
| `DeepSearchTool` | `deep_search` | 多引擎深度搜索 | `/v1/tool/deepsearch` |
| `CodeInterpreterTool` | `code_interpreter` | Python 代码执行 | `/v1/tool/code_interpreter` |
| `ReportTool` | `report` | 报告生成 (HTML/MD/PPT) | `/v1/tool/report` |
| `FileTool` | `file_tool` | 文件读写操作 | — (本地) |
| `DataAnalysisTool` | `data_analysis` | 数据诊断分析 | `/v1/tool/auto_analysis` |
| `PlanningTool` | `planning` | 计划管理 | — (本地) |
| `McpTool` | (动态) | MCP 外部工具 | 通过 genie-client 代理 |

#### 抽象三：`AgentContext` — 共享上下文

**源文件**: `genie-backend/src/main/java/com/jd/genie/agent/agent/AgentContext.java`

```java
@Builder
public class AgentContext {
    private String requestId;              // 请求唯一标识
    private String query;                  // 用户原始问题
    private String task;                   // 当前执行任务
    private ToolCollection toolCollection; // 共享工具集
    private List<File> productFiles;       // 产出文件列表（跨 Agent 累积）
    private List<File> taskProductFiles;   // 当前任务产出文件
    private Printer printer;               // SSE 输出器
    private String sopPrompt;              // SOP 标准流程提示词
    private String basePrompt;             // 基础提示词
    private String dateInfo;               // 当前日期信息
    private Integer agentType;             // Agent 类型标识
    private boolean isStream;              // 是否流式输出
    private String templateType;           // 模板类型（fix/empty）
}
```

**设计意图**: AgentContext 是贯穿整个请求生命周期的**"共享黑板"**。多个 Agent（PlanningAgent、ExecutorAgent、SummaryAgent）通过它共享工具集合、产出文件、SSE 输出通道，避免了 Agent 之间的直接耦合。

### 2. 对象的生命周期 — 一次完整的 Agent 执行

以用户发送 "帮我分析最近美元和黄金的走势，生成 HTML 报告" 为例：

```
                            请求生命周期
                            ═══════════

  ┌──────────┐    ┌──────────────────┐    ┌──────────────────┐
  │  浏览器   │───▶│  UI (React)      │───▶│ Backend (Spring) │
  │          │ SSE│  querySSE.ts     │POST│ GenieController   │
  └──────────┘    └──────────────────┘    └────────┬─────────┘
                                                    │
                                    ┌───────────────▼────────────────┐
                                    │ 1. 构建 AgentContext            │
                                    │    - requestId, query           │
                                    │    - ToolCollection (注册工具)  │
                                    │    - SSEPrinter (流式输出)      │
                                    └───────────────┬────────────────┘
                                                    │
                                    ┌───────────────▼────────────────┐
                                    │ 2. AgentHandlerFactory         │
                                    │    根据 agentType 选择 Handler  │
                                    │    ├─ REACT → ReactHandlerImpl │
                                    │    └─ PLAN_SOLVE → PlanSolve.. │
                                    └───────────────┬────────────────┘
                                                    │
                      ┌─────────────────────────────▼──────────────────────────────┐
                      │ 3. PlanSolveHandlerImpl.handle()                           │
                      │                                                             │
                      │   Phase 1: 规划                                             │
                      │   ┌─────────────┐                                          │
                      │   │PlanningAgent│──▶ LLM 生成计划 ──▶ SSE: plan_thought    │
                      │   │  .run()     │──▶ PlanningTool ──▶ SSE: task (任务列表) │
                      │   └─────────────┘                                          │
                      │         │                                                   │
                      │   Phase 2: 执行循环                                         │
                      │   ┌─────────────┐                                          │
                      │   │ExecutorAgent│──▶ think(): LLM 选择工具                  │
                      │   │  .run()     │──▶ act(): HTTP调用 genie-tool             │
                      │   └─────────────┘──▶ SSE: tool_thought, tool_result        │
                      │         │ (结果反馈给 PlanningAgent，更新计划)              │
                      │         │ (循环直到 PlanningAgent 返回 "finish")            │
                      │         │                                                   │
                      │   Phase 3: 总结                                             │
                      │   ┌─────────────┐                                          │
                      │   │SummaryAgent │──▶ 汇总所有 Memory ──▶ SSE: result       │
                      │   └─────────────┘                                          │
                      └────────────────────────────────────────────────────────────┘
```

**关键数据流转路径**:

1. **请求转换**: `GptQueryReq` (前端) → `AgentRequest` (服务层) → `AgentContext` (Agent 层)
2. **记忆积累**: Agent 执行过程中，通过 `Memory` 对象持续积累对话历史 (USER / ASSISTANT / TOOL 消息)
3. **工具调用**: `ToolCall` → `executeTool()` → HTTP 调用 genie-tool → 返回 String 结果
4. **实时推送**: 全程通过 `SSEPrinter` 向前端推送中间状态（`plan_thought`, `tool_thought`, `tool_result`, `task`, `result`）

---

## 模块三：关键流程深度剖析 (100 英尺视角)

> **选择的核心流程: Plan-Solve (规划-执行) 多 Agent 协作流程**
>
> 这是 JoyAgent 最核心、最有特色的流程，体现了 "multi-level and multi-pattern thinking" 的设计理念。

### 1. 流程启始

**入口文件**: `genie-backend/src/main/java/com/jd/genie/service/impl/PlanSolveHandlerImpl.java`

```java
@Override
public String handle(AgentContext agentContext, AgentRequest request) {
    // Step 0: SOP 召回 - 查找是否有相似任务的标准流程模板
    handleSopRecall(agentContext, request);

    // Step 1: 创建三大核心 Agent
    PlanningAgent planning = new PlanningAgent(agentContext);  // 规划者
    ExecutorAgent executor = new ExecutorAgent(agentContext);   // 执行者
    SummaryAgent summary = new SummaryAgent(agentContext);      // 总结者

    // Step 2: 规划阶段 - PlanningAgent 调用 LLM 生成任务列表
    String planningResult = planning.run(agentContext.getQuery());

    // Step 3: 进入"规划-执行"循环...
}
```

### 2. 关键决策点

#### 决策点 A: ReAct vs Plan-Solve 模式选择

**位置**: `MultiAgentServiceImpl.searchForAgentRequest()`

```java
// deepThink == 0 → REACT（快速单 Agent 模式，适合简单问题）
// deepThink != 0 → PLAN_SOLVE（多 Agent 规划执行模式，适合复杂任务）
Integer agentType = (params.getDeepThink() != null && params.getDeepThink() == 0)
    ? AgentType.REACT.getValue()
    : AgentType.PLAN_SOLVE.getValue();
```

**设计模式**: 策略模式 — `AgentHandlerFactory` 遍历所有 `AgentHandlerService` 实现类，调用 `support()` 方法匹配对应的 Handler。

#### 决策点 B: 单任务 vs 并行任务执行

**位置**: `PlanSolveHandlerImpl.handle()` 第 53-86 行

```java
// PlanningAgent 通过 <sep> 分隔符标记可并行执行的任务
List<String> planningResults = Arrays.stream(planningResult.split("<sep>"))
        .map(task -> "你的任务是：" + task)
        .collect(Collectors.toList());

if (planningResults.size() == 1) {
    // ✅ 单任务: 直接用主 Executor 执行
    executorResult = executor.run(planningResults.get(0));
} else {
    // ✅ 多任务: 为每个任务创建 slaveExecutor，并发执行
    for (String task : planningResults) {
        ExecutorAgent slaveExecutor = new ExecutorAgent(agentContext);
        slaveExecutor.getMemory().addMessages(executor.getMemory().getMessages());  // 共享历史记忆
        ThreadUtil.execute(() -> {
            String taskResult = slaveExecutor.run(task);
            tmpTaskResult.put(task, taskResult);
            taskCount.countDown();
        });
    }
    ThreadUtil.await(taskCount);  // 等待所有并发任务完成
    // 合并所有 slaveExecutor 的 Memory 到主 Executor
}
```

**设计亮点**: 这是项目"高并发 DAG 执行引擎"的核心实现 — PlanningAgent 通过语义化的 `<sep>` 分隔符标记任务间的并行关系，Handler 自动创建多个 ExecutorAgent 实例并发执行，完成后合并 Memory。

#### 决策点 C: 继续规划 vs 终止

```java
planningResult = planning.run(executorResult);  // 将执行结果反馈给 PlanningAgent

if ("finish".equals(planningResult)) {
    // → 所有任务完成，进入总结阶段
    TaskSummaryResult result = summary.summaryTaskResult(executor.getMemory().getMessages(), request.getQuery());
    agentContext.getPrinter().send("result", taskResult);
    break;
}
if (planning.getState() == AgentState.ERROR || executor.getState() == AgentState.ERROR) {
    // → 异常终止
    agentContext.getPrinter().send("result", "任务执行异常，请联系管理员，任务终止。");
    break;
}
// → 否则继续下一轮 task 分配和执行
```

### 3. 数据流转详解

```
用户 Query: "分析美元和黄金走势并生成报告"
    │
    ▼
[PlanningAgent] ── LLM Function Call ──▶ PlanningTool.execute()
    │  生成 Plan:
    │    Step 1: "搜索美元走势数据"         (not_started)
    │    Step 2: "搜索黄金走势数据"         (not_started)  ← 与Step1可并行(<sep>)
    │    Step 3: "数据分析和对比"           (not_started)
    │    Step 4: "生成HTML报告"            (not_started)
    │
    ▼ 返回 "搜索美元走势数据<sep>搜索黄金走势数据"
[PlanSolveHandler] ── 检测到 <sep> ──▶ 创建 2 个 slaveExecutor 并发执行
    │
    ├──▶ [ExecutorAgent-1] think() → LLM 选择 DeepSearchTool
    │       act() → HTTP POST http://127.0.0.1:1601/v1/tool/deepsearch
    │       返回美元走势搜索结果 → updateMemory(TOOL, result, toolCallId)
    │
    └──▶ [ExecutorAgent-2] think() → LLM 选择 DeepSearchTool
            act() → HTTP POST http://127.0.0.1:1601/v1/tool/deepsearch
            返回黄金走势搜索结果 → updateMemory(TOOL, result, toolCallId)
    │
    ▼ 合并 Memory，反馈给 PlanningAgent
[PlanningAgent] ── 更新 Plan, 标记 Step 1/2 完成 ──▶ 返回 "数据分析和对比"
    │
    ▼
[ExecutorAgent] think() → LLM 选择 CodeInterpreterTool
    │  act() → HTTP POST http://127.0.0.1:1601/v1/tool/code_interpreter
    │  返回分析结果
    │
    ▼ 反馈给 PlanningAgent → 返回 "生成HTML报告"
[ExecutorAgent] think() → LLM 选择 ReportTool
    │  act() → HTTP POST http://127.0.0.1:1601/v1/tool/report
    │  返回 HTML 报告文件 → 加入 productFiles
    │
    ▼ 反馈给 PlanningAgent → 返回 "finish"
[SummaryAgent] ── summaryTaskResult() ──▶ LLM 总结所有 Memory
    │  提取 productFiles (HTML 报告文件)
    │
    ▼
[SSEPrinter] ── send("result", {taskSummary, fileList}) ──▶ 前端渲染报告
```

### 4. 异常处理机制

| 层级 | 位置 | 处理方式 |
|------|------|----------|
| **Agent 级** | `BaseAgent.run()` catch Exception | 设置 `state = AgentState.ERROR`，向上传播 |
| **Handler 级** | `PlanSolveHandlerImpl` 循环内检查 | 检测 `ERROR` 状态，发送终止消息给前端 |
| **工具级** | `BaseAgent.executeTool()` catch Exception | 返回 `"Tool xxx Error."` 字符串（优雅降级，不中断流程） |
| **SSE 级** | `GenieController` | 10秒心跳检测 + onError/onTimeout/onCompletion 回调 |
| **SOP 召回** | `handleSopRecall()` catch Exception | 注释明确"SOP召回失败不影响主流程" — 容错设计 |
| **全局保护** | `maxSteps` 上限 | 防止 Agent 无限循环，达到上限自动终止并返回提示 |

---

## 模块四：实践与洞察 (落地视角)

### 1. 如何运行第一个示例

#### 方式一: Docker 一键启动（推荐新手）

```bash
# 1. 克隆项目
git clone https://github.com/jd-opensource/joyagent-jdgenie.git
cd joyagent-jdgenie

# 2. 配置 LLM（必须步骤！）
#    编辑 genie-backend/src/main/resources/application.yml:
#      - 设置 base_url、apikey、model (如 deepseek-chat)
#      - 使用 DeepSeek 时注意 max_tokens: 8192
#    编辑 genie-tool/.env_template:
#      - 设置 OPENAI_API_KEY、OPENAI_BASE_URL、DEFAULT_MODEL
#      - 设置 SERPER_SEARCH_API_KEY（搜索功能需要）
#      - 使用 DeepSeek 时设置 DEEPSEEK_API_KEY、DEEPSEEK_API_BASE

# 3. 构建并启动
docker build -t genie:latest .
docker run -d -p 3000:3000 -p 8080:8080 -p 1601:1601 --name genie-app genie:latest

# 4. 浏览器访问 http://localhost:3000
```

#### 方式二: 手动启动（适合开发调试）

```bash
# 前置要求: JDK 17, Python 3.11, pip install uv

# 1. 检查依赖和端口占用
sh check_dep_port.sh

# 2. 一键启动所有服务
#    前端(3000) + 后端(8080) + 工具服务(1601) + MCP客户端(8188)
sh Genie_start.sh
# Ctrl+C 可一键停止所有服务
```

**首次体验**: 启动后打开 `http://localhost:3000`，选择 "网页模式"，输入 "帮我分析最近美元和黄金的走势并生成报告"，即可体验完整的多 Agent 协作流程。

### 2. 可拓展的思考题

**"如果要为 Agent 增加长期记忆能力（跨会话记忆），应该修改或扩展哪个模块？"**

当前 `Memory` 类 (`genie-backend/.../dto/Memory.java`) 是纯内存的 `List<Message>`，会话结束即丢失。要实现长期记忆：

- **存储层**: 扩展 `AgentContext`，添加 `LongTermMemory` 接口 + Qdrant 向量库持久化（Qdrant 已在技术栈中）
- **注入时机**: 在 `BaseAgent.run()` 开头检索历史相似任务的记忆，注入 System Prompt
- **持久化时机**: 在 `SummaryAgent` 总结完成后，将关键结论向量化存入 Qdrant
- **关联性**: README 中提到的 "cross task workflow memory" 创新点正是此方向的设计预留，当前代码尚未完整实现，是一个极好的开源贡献方向

### 3. 潜在的难点与坑

#### 难点一: LLM 配置与兼容性

- 项目默认使用 **OpenAI API 格式**，但 `LLM.java` 中对 Claude 有专门的消息格式转换逻辑（Claude 不允许连续的同角色消息、System Message 处理不同）
- 使用 **DeepSeek** 时需注意 `max_tokens` 限制为 8192，且需在 `application.yml`（后端）和 `.env`（工具服务）**两处同步修改**配置
- 如果使用**不支持 Function Calling** 的模型，系统会回退到 "struct parse" 模式（正则解析 JSON），准确率会明显下降
- `LLM.java` 中的 `askTool()` 方法同时支持 native function calling 和 struct parsing 两条路径，调试时需注意区分

#### 难点二: 多服务协调与端口冲突

- 系统由 **4 个独立服务**组成（UI:3000 / Backend:8080 / Tool:1601 / MCP:8188），启动有隐含的依赖顺序
- `Genie_start.sh` 脚本内置了健康检查逻辑，但手动逐个启动时需确保所有服务就绪后再使用
- `genie-tool` 使用 `uv` 管理 Python 虚拟环境，国内网络环境下依赖下载可能超时
- `genie-client` 的 MCP SDK 版本依赖较严格（`mcp==1.9.4`），版本不匹配会导致连接失败
- **建议新手优先使用 Docker 方式**，遇到问题时参考 `check_dep_port.sh` 的输出逐项排查

---

## 附录：设计模式速查表

| 设计模式 | 应用位置 | 说明 |
|---------|---------|------|
| **模板方法** | `BaseAgent.run()` → `step()` | 定义 Agent 执行骨架，子类只需实现 `step()` |
| **策略模式** | `AgentHandlerFactory` + `AgentHandlerService` | 根据 agentType 动态选择 ReAct/PlanSolve Handler |
| **工厂模式** | `AgentHandlerFactory.getHandler()` | 创建/选择合适的 Handler 实例 |
| **Builder 模式** | `AgentContext.builder()` | 构建复杂的共享上下文对象 |
| **观察者模式** | `SSEPrinter` + `SseEmitter` | SSE 流式推送实时状态给前端 |
| **命令模式** | `BaseTool` + `ToolCollection` | 工具注册、查找、统一执行 |
| **黑板模式** | `AgentContext` | 多 Agent 通过共享上下文间接通信 |

---

> **报告结束**。如有疑问或需要深入探讨某个模块，欢迎继续交流。
