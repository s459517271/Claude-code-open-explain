# 00 — 全局架构概览

> 先看大图景：Claude Code 是“模型 + 本地编排层 + 安全约束”的组合体

## 本章适合谁

如果你刚接触这个项目，或者你已经打开了很多源码文件但不知道它们彼此是什么关系，先读这一章。

这章的目标只有一个：让你先建立 Claude Code 的整体心智模型，再去看后面的 Prompt、工具、权限和上下文细节。

## 先说结论

Claude Code 的架构可以先粗暴地记成一句话：

> 一个持续运行的 agent loop，外面包着 prompt 组装、工具系统、权限系统、上下文管理和扩展层。

也就是说，真正反复跑起来的核心并不复杂，复杂的是围绕这个 loop 的工程化设施。

## 如果你是新手，先分清 3 个角色

### 1. 模型

模型负责理解需求、决定下一步做什么、生成文字或工具调用请求。

### 2. Claude Code

Claude Code 负责把模型放进一个可控的本地运行环境里。它会准备 prompt、提供工具、执行权限检查、管理上下文、渲染终端 UI。

### 3. 你的电脑

文件系统、Shell、Git、MCP 服务器、IDE、网络等能力都在本地环境里。模型本身不能直接碰这些能力，必须通过 Claude Code 暴露出来的工具间接访问。

## 最小心智模型

```text
用户输入
  ↓
Claude Code 收集上下文
  ↓
Claude Code 组装 prompt 和工具列表
  ↓
模型返回：
  - 直接文字
  - 或工具调用请求
  ↓
Claude Code 判断能不能执行该工具
  ↓
执行工具，把结果回送给模型
  ↓
重复，直到模型返回最终文本
```

所以它不是“模型一次性回答”，而是“模型和本地运行时之间的多轮协作”。

## 系统分层

```text
终端 UI 层
  React + Ink
  负责输入、输出、流式渲染、权限弹窗、状态展示

CLI 与启动层
  src/main.tsx
  负责参数解析、初始化配置、装配运行时

对话编排层
  src/QueryEngine.ts
  负责一次会话中每一轮请求的状态管理和参数准备

执行循环层
  src/query.ts
  负责真正的模型调用、工具执行、结果回传循环

基础能力层
  Prompt、Tools、Permissions、Context、Compaction、MCP

集成服务层
  API、OAuth、LSP、Analytics、Remote、Plugins、Skills
```

## 本章最重要的源码入口

| 文件 | 作用 | 为什么值得先看 |
|------|------|----------------|
| `src/main.tsx` | 程序入口与启动装配 | 看懂 CLI 从哪开始把系统拼起来 |
| `src/QueryEngine.ts` | 一轮会话请求的组织者 | 看懂一次 turn 是怎么被准备出来的 |
| `src/query.ts` | 真正的 agent loop | 看懂“模型 -> 工具 -> 模型”的循环 |
| `src/constants/prompts.ts` | System Prompt 组装 | 看懂模型为什么会被约束成现在这样 |
| `src/tools.ts` | 工具注册表 | 看懂 Claude Code 给模型开放了哪些能力 |
| `src/utils/permissions/*` | 权限与安全控制 | 看懂为什么它不会直接乱执行 |
| `src/services/compact/*` | 上下文压缩 | 看懂长对话为什么还能继续 |
| `src/services/mcp/*` | MCP 集成 | 看懂外部能力如何接进来 |

## 一次完整请求是怎么流动的

### 第 1 步：启动阶段先把运行时搭起来

`src/main.tsx` 做的不是业务核心，而是把所有业务逻辑装进一个可运行程序。它会做这些准备：

- 读取配置和环境变量
- 初始化权限上下文
- 加载内置工具、插件、技能、MCP
- 初始化 UI、Telemetry、LSP 等外围能力
- 创建后续对话会用到的状态容器

### 第 2 步：`QueryEngine` 准备本轮请求

当用户输入一条消息后，`src/QueryEngine.ts` 会负责：

- 确定本轮要用哪个模型
- 获取 System Prompt、userContext、systemContext
- 包装权限检查函数
- 维护消息历史、文件状态缓存、token 用量
- 把这些参数交给真正的 `query()` 循环

### 第 3 步：`query()` 进入真正的 agent loop

`src/query.ts` 才是这套系统真正意义上的心脏。它会不断重复：

1. 调用模型
2. 流式读取模型输出
3. 如果输出里包含工具调用，就执行工具
4. 把工具结果追加回消息历史
5. 再次调用模型

直到模型不再请求工具，只剩最终文本。

### 第 4 步：工具、权限、上下文在循环中不断参与

这一层你可以把它理解成 3 个护栏：

- **工具系统**：决定“能做什么”
- **权限系统**：决定“现在允不允许做”
- **上下文系统**：决定“模型这次能看到什么”

Claude Code 的绝大多数工程复杂度，都是为了把这 3 个问题处理好。

## 为什么核心 loop 很简单，但代码还是很多

表面上看，主流程像这样：

```text
while (模型还在调工具) {
  执行工具
  把结果发回模型
}
```

但产品级 CLI 不能只停在这一步，因为现实里还要解决：

- 工具执行前是否有权限
- 命令是否危险
- 文件路径是否合法
- 上下文是否快溢出
- Prompt 是否会打爆缓存
- MCP 服务器会不会动态变化
- 多 Agent 怎么隔离上下文
- UI 如何实时显示每一步状态

所以 Claude Code 不是“算法复杂”，而是“工程边界条件复杂”。

## Claude Code 的设计哲学

### 1. 尽量把决策交给模型

Claude Code 没有一个很重的外部任务规划器去替模型做决策。它更像是在给模型搭一个受控工作台。

### 2. 尽量把约束交给工程层

模型负责决策，不代表它拥有无限权力。Prompt、权限、沙箱、路径验证、模式切换，这些都由工程层硬性约束。

### 3. 让复杂集中在外围，而不是核心循环里

`query.ts` 的主循环仍然保持相对直接，这是一个非常重要的取舍。它让系统更容易调试，也更容易扩展。

## 建议的源码阅读顺序

如果你已经准备打开源码，建议按这个顺序：

1. `src/main.tsx`
2. `src/QueryEngine.ts`
3. `src/query.ts`
4. `src/constants/prompts.ts`
5. `src/tools.ts`
6. `src/utils/permissions/PermissionMode.ts`
7. `src/services/compact/autoCompact.ts`
8. `src/services/mcp/client.ts`

不要一上来就钻进 `utils/permissions` 或 `services/mcp` 的深层文件，那样很容易迷路。

## 本章小结

先把 Claude Code 看成一层“本地编排系统”，后面的章节就容易理解很多：

- Prompt 决定模型怎么想
- Tool 决定模型能做什么
- Permission 决定工具是否能执行
- Context 决定模型这次看到了什么
- MCP 决定系统可以扩展到什么边界

## 下一步

- [01 — System Prompt 分层设计](../01-system-prompt/)：看懂模型行为是怎么被一步步塑形的
- [02 — Agent Loop 核心循环](../02-agentic-loop/)：看懂一次请求从输入到输出到底经历了什么
