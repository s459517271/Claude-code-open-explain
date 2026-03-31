# Decode CC

> 深度解读 Claude Code 的架构设计、运行链路与工程取舍

<p align="center">
  <strong>不只告诉你“它做了什么”，还会讲清楚“它为什么这样设计”</strong>
</p>

<p align="center">
  <a href="#这个项目是什么">这个项目是什么</a> •
  <a href="#如果你是新手先看这里">新手导读</a> •
  <a href="#阅读路线">阅读路线</a> •
  <a href="#目录">目录</a> •
  <a href="#声明">声明</a>
</p>

---

## 这个项目是什么

Claude Code 可以理解为一个运行在本地终端里的 AI 编程编排器。

它自己不是模型本身，而是站在模型和你的电脑之间，负责做这些事：

- 收集当前工作目录、Git 状态、用户配置等上下文
- 动态组装 System Prompt 和工具描述
- 把请求发给模型，并流式接收返回结果
- 在本地执行文件读写、Shell、搜索、MCP 等工具
- 在执行前做权限校验、安全检查和上下文管理

本仓库的目标，不是搬运源码，也不是只贴结论，而是把 `Claude-code-open` 这份公开源码里真正重要的设计讲明白，尤其面向第一次接触 Agent CLI、Prompt Cache、MCP、多 Agent 这些概念的读者。

## 这个项目不是什么

- 不是 Claude Code 的官方文档
- 不是 Claude Code 的源码镜像仓库
- 不是“看一眼目录就下结论”的浅层解读

这里更像一份“源码导读 + 架构讲义”。

## 为什么值得读

Claude Code 的难点不在某一个算法，而在于它把很多真实工程问题揉在了一起：

- 大模型如何稳定地调用本地工具
- 如何在高自由度和高安全性之间取得平衡
- 如何把上下文、缓存、权限、插件、MCP 这些系统拼成一个可靠产品
- 为什么核心 loop 可以很简单，但外围工程层非常厚

如果你以后想做 AI Coding Agent、带工具调用的终端助手、IDE/CLI 里的大模型应用，Claude Code 都是非常好的样本。

## 如果你是新手，先看这里

### 先建立 5 个最小概念

1. **System Prompt**：给模型的长期规则，决定角色、边界和风格。
2. **Tool**：模型不能直接操作电脑，只能先请求工具，再由 CLI 代为执行。
3. **Agent Loop**：一次回答如果需要多个工具，就会不断重复“模型思考 -> 调工具 -> 回传结果”。
4. **Context**：模型每次调用时看到的全部输入，包括历史消息、System Prompt、工具结果等。
5. **MCP**：一种把外部工具、资源和能力接入 Claude Code 的协议。

### 再记住一句话

> Claude Code 的本质不是“一个会写代码的黑盒”，而是“一个把模型、安全、工具和上下文组织起来的本地编排层”。

## 这次解读对应哪份源码

本文档主要对应你本地的：

```text
D:\visual_ProgrammingSoftware\毕设and简历Projects\Claude-code-open
```

按当前公开快照粗略统计，仓库规模大约在：

- `1900+` 个文件
- `48` 万行代码量级
- 主体实现集中在 `src/`

阅读本仓库文档时，文中出现的源码路径默认都指向那份源码仓库的相对路径，例如 `src/main.tsx`、`src/QueryEngine.ts`。

## 快速建立整体直觉

```text
你的终端输入
  ↓
src/main.tsx
  启动 CLI、加载配置、初始化 UI 与运行时
  ↓
src/QueryEngine.ts
  组装本轮请求所需的模型、Prompt、权限和上下文
  ↓
src/query.ts
  进入真正的 agent loop：调用模型、执行工具、回写结果
  ↓
src/tools.ts + src/Tool.ts
  管理全部内置工具与工具契约
  ↓
src/utils/permissions/*
  决定这次工具调用能不能执行
  ↓
src/services/compact/*
  决定上下文满了以后怎么压缩和续跑
  ↓
src/services/mcp/*
  把外部 MCP 能力接入到同一套工具体系
```

## 阅读路线

### 路线 A：完全新手

建议按下面顺序读：

1. [00-overview](./00-overview/)
2. [02-agentic-loop](./02-agentic-loop/)
3. [03-tool-system](./03-tool-system/)
4. [04-permission-model](./04-permission-model/)
5. [05-context-management](./05-context-management/)
6. [01-system-prompt](./01-system-prompt/)
7. [06-prompt-caching](./06-prompt-caching/)

### 路线 B：想自己做 Agent CLI

建议优先看：

1. [02-agentic-loop](./02-agentic-loop/)
2. [03-tool-system](./03-tool-system/)
3. [04-permission-model](./04-permission-model/)
4. [08-mcp-integration](./08-mcp-integration/)
5. [07-multi-agent](./07-multi-agent/)

### 路线 C：更关心性能与产品化

建议优先看：

1. [06-prompt-caching](./06-prompt-caching/)
2. [05-context-management](./05-context-management/)
3. [09-startup-optimization](./09-startup-optimization/)
4. [10-feature-flags](./10-feature-flags/)
5. [11-security](./11-security/)

## 目录

| 章节 | 主题 | 这章解决什么问题 | 新手为什么要看 |
|------|------|------------------|----------------|
| [00-overview](./00-overview/) | 全局架构概览 | Claude Code 到底由哪些层组成？ | 建立全局图，避免一上来就陷进细节 |
| [01-system-prompt](./01-system-prompt/) | System Prompt 分层设计 | Prompt 不是一段字符串，而是一套装配系统，这意味着什么？ | 理解模型行为为什么可控 |
| [02-agentic-loop](./02-agentic-loop/) | Agent Loop 核心循环 | 一次用户请求如何变成多轮工具调用？ | 看懂整个产品的心脏 |
| [03-tool-system](./03-tool-system/) | 工具系统架构 | 工具如何注册、描述、执行、返回结果？ | 看懂 AI 为什么能操作本地环境 |
| [04-permission-model](./04-permission-model/) | 权限安全模型 | 工具调用为什么不是直接执行？ | 理解安全边界和确认机制 |
| [05-context-management](./05-context-management/) | 上下文管理与压缩 | 对话越来越长时，系统如何继续工作？ | 理解长上下文和压缩策略 |
| [06-prompt-caching](./06-prompt-caching/) | Prompt Cache 优化 | 为什么源码里到处都在想办法保持 prompt 稳定？ | 理解成本优化与架构约束 |
| [07-multi-agent](./07-multi-agent/) | 多 Agent 协作 | Claude Code 什么时候会拆分子 Agent？ | 理解并行化和任务隔离 |
| [08-mcp-integration](./08-mcp-integration/) | MCP 协议集成 | 外部能力如何无缝接进 Claude Code？ | 理解扩展性从哪里来 |
| [09-startup-optimization](./09-startup-optimization/) | 启动性能优化 | 一个 CLI 为什么也要抠启动毫秒？ | 理解产品级性能工程 |
| [10-feature-flags](./10-feature-flags/) | Feature Flag 体系 | 为什么代码里有这么多隐藏模块和条件导入？ | 学会从 Flag 读产品演进 |
| [11-security](./11-security/) | 安全机制深度分析 | Claude Code 如何避免把高权限 Agent 做成危险软件？ | 理解真实可落地的安全防线 |

## 怎样配合源码一起读

### 推荐方法

1. 先读本章 README，弄清楚本章的核心问题。
2. 再打开文中列出的源码入口文件，不要求一开始逐行读懂。
3. 重点看“函数之间如何协作”，不要一开始纠结每个细节。
4. 每章读完后，再回到总图里确认它在整个系统中的位置。

### 不推荐的方法

- 一上来就全局搜索关键字，然后被大量结果淹没
- 把所有 `feature(...)` 分支都当成稳定公开功能
- 看到某个工具或某个 Hook 就立刻下结论，忽略前后的上下文

## 参考材料

### 源码与逆向资料

- [instructkr/claude-code](https://github.com/instructkr/claude-code)
- [hitmux/HitCC](https://github.com/hitmux/HitCC)
- [Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)
- [ghuntley/claude-code-source-code-deobfuscation](https://github.com/ghuntley/claude-code-source-code-deobfuscation)

### 分析文章

- [How Claude Code Actually Works (KaraxAI)](https://karaxai.com/posts/how-claude-code-works-systems-deep-dive/)
- [Under the Hood of Claude Code (Pierce Freeman)](https://pierce.dev/notes/under-the-hood-of-claude-code/)
- [Architecture & Internals (Bruniaux)](https://cc.bruniaux.com/guide/architecture/)
- [Digging into the Source (Dave Schumaker)](https://daveschumaker.net/digging-into-the-claude-code-source-saved-by-sublime-text/)

## 贡献

欢迎提交 PR 或 Issue，尤其欢迎这几类补充：

- 纠正文档中的技术细节错误
- 为章节补充流程图、时序图和示意图
- 增加适合初学者的例子、术语解释和阅读提示
- 补充某个具体模块的深挖章节

## 声明

本项目仅用于教育和技术研究。  
本文档不包含 Claude Code 原始源码，只包含对公开可得源码快照的结构分析、架构解读与工程说明。相关知识产权归原项目权利方所有。

---

<p align="center">
  <sub>如果这份文档对你有帮助，欢迎 Star。本仓库会继续把“源码能看见什么”讲得更清楚。</sub>
</p>

## 友情链接

https://linux.do
