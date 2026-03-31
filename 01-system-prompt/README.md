# 01 — System Prompt 分层设计

> Claude Code 的 Prompt 不是一段写死的长文本，而是一套按规则装配的系统

## 先建立一个新手最容易忽略的事实

很多人第一次看 Agent 产品时，会把 System Prompt 想成“一段总说明”。

Claude Code 不是这样。

在源码里，System Prompt 更像是：

- 一组可组合的规则片段
- 一套明确区分静态内容和动态内容的装配逻辑
- 一个同时服务于“行为控制”和“缓存稳定性”的工程结构

这意味着它不仅影响模型回答质量，也直接影响成本、性能和可维护性。

## 本章先分清 3 个概念

### 1. `systemPrompt`

这是发给模型的高优先级规则集合。比如身份定义、做事原则、输出风格、工具使用规则，都在这里。

### 2. `userContext`

这是和当前用户、当前项目更相关的上下文。例如 `CLAUDE.md`、项目说明、局部目录规则等，更适合放在这里。

### 3. `systemContext`

这是运行环境信息。例如操作系统、当前目录、Git 状态之类的信息。

这 3 类内容都会影响模型，但它们不应该以同一种方式注入。这就是 Claude Code 在 Prompt 设计上非常成熟的地方。

## 本章关键源码入口

| 文件 | 作用 |
|------|------|
| `src/constants/prompts.ts` | 定义 System Prompt 的组装逻辑和各段内容 |
| `src/utils/queryContext.ts` | 把 `systemPrompt`、`userContext`、`systemContext` 一起取出来 |
| `src/context.ts` | 生成系统上下文和用户上下文 |

如果你只读一个文件，先读 `src/constants/prompts.ts`。

## `getSystemPrompt()` 到底在做什么

在 `src/constants/prompts.ts` 里，`getSystemPrompt()` 会按顺序生成一组字符串片段。

你可以把它理解成：

```text
先放稳定规则
  ↓
插入静态/动态边界
  ↓
再放会话级、环境级、个性化内容
```

从设计上看，这说明 Claude Code 并不是“想到什么就往 prompt 里拼什么”，而是先决定：

- 哪些内容应该长期稳定
- 哪些内容会随会话变化
- 哪些变化会破坏缓存
- 哪些变化应该迁移到消息层而不是 prompt 层

## 静态区与动态区

源码里最关键的标记之一是：

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

它的意义不是让代码更好看，而是：

> 明确告诉系统，哪些 prompt 片段应该尽量保持稳定，哪些片段允许随会话变化。

### 你可以先这样理解

```text
静态区
  身份、通用规则、做事原则、工具使用哲学

动态区
  当前会话指导、语言偏好、输出风格、MCP 指令、环境差异
```

### 为什么一定要这样分

因为 Claude Code 非常在意 Prompt Cache。

如果动态信息混进了原本稳定的前缀里，就会导致缓存更容易失效。所以“Prompt 怎么写”在这里不仅是提示词问题，更是缓存工程问题。

## 静态区里通常放什么

### 1. 身份与角色

例如 `getSimpleIntroSection()` 会给模型一个非常克制的身份定义：

- 你是一个交互式代理
- 你的目标是帮助用户完成软件工程任务
- 你需要遵守后续列出的规则和工具边界

这类定义不会每轮都变，因此适合放在静态区。

### 2. 做事原则

`getSimpleDoingTasksSection()` 是非常值得认真读的一段。它不是在教模型“怎么写漂亮话”，而是在规定一种很明确的工程风格：

- 不要擅自扩 scope
- 不要为了假设中的未来需求提前抽象
- 不要为了“一次性操作”硬造公共工具
- 先理解现有代码，再修改
- 如果验证没做，就不能假装它已经成功

这一段其实就是 Claude Code 的“工程人格”。

### 3. 风险操作约束

`getActionsSection()` 处理的是另一个很重要的问题：

> 模型再聪明，也不能默认替用户做高风险决定。

所以这段 prompt 会反复强调：

- 对可逆的本地动作可以相对积极
- 对不可逆、影响共享系统、可能破坏用户工作的动作要先确认
- 一次授权不等于永久授权

### 4. 输出方式与风格

`getOutputEfficiencySection()`、`getOutputStyleSection()` 之类的内容决定模型怎么说话。

这层看起来像文案问题，但其实很重要，因为它会影响：

- 输出长度
- 信息密度
- 是否适合在终端里阅读
- 是否利于持续多轮协作

## 动态区里通常放什么

在 `prompts.ts` 里，动态区大量通过 `systemPromptSection(...)` 注册。

你可以把它理解成“按需计算的 prompt 片段”。典型内容包括：

- 会话指导
- 记忆内容
- 环境信息
- 语言偏好
- 输出风格
- MCP 指令
- 某些由功能开关控制的附加提示

这种写法的好处是：

- 不会把所有分支硬塞在一个巨大字符串里
- 每一段有自己的身份和职责
- 方便做缓存、调试和替换

## 为什么 `CLAUDE.md` 不直接塞进 System Prompt

直觉上你可能会觉得：既然 `CLAUDE.md` 是项目规则，那不是最应该放进 System Prompt 吗？

但源码没有这么做，原因很务实：

- `CLAUDE.md` 更容易变化
- 它高度依赖当前项目或目录
- 如果直接混进稳定的 prompt 前缀，会更容易让缓存失效

所以 Claude Code 把它放进了 `userContext` 这一侧，让它作为会话上下文参与，而不是污染稳定的系统前缀。

## 为什么 MCP 指令被特别对待

在 `prompts.ts` 里，你会看到这类写法：

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => ...,
  'MCP servers connect/disconnect between turns',
)
```

这个名字故意起得很重，就是在提醒开发者：

> 这段内容会动，而且它一动就可能破坏原本稳定的 prompt。

原因很简单：

- MCP 服务器可能在运行中连接、断开或重载
- 每个服务器可能还带有自己的 `instructions`
- 如果这些信息直接嵌进稳定 prompt，就会导致缓存频繁重建

因此源码里后来又引入了 `mcp_instructions_delta` 这类思路，把这类高波动信息从 System Prompt 迁到消息附件层。

## 真正要学的，不只是 Prompt 文案，而是 Prompt 架构

这一章最值得学的不是“Claude Code prompt 具体写了什么句子”，而是下面这套架构思维：

### 1. 把稳定规则和会话变化拆开

这是缓存和可维护性的基础。

### 2. 把人格、规则、风格、环境分层

这样每层更容易单独调整，不会互相污染。

### 3. 对高波动信息做特殊迁移

像 MCP 指令、动态 agent 列表这类内容，不适合长期待在稳定 prompt 前缀里。

### 4. Prompt 是系统设计的一部分

在 Claude Code 里，Prompt 不是运营文案，而是运行时结构的一部分。

## 新手最常见的误区

### 误区 1：Prompt 只是模型口气问题

不是。它还影响工具行为、安全边界、缓存命中和用户体验。

### 误区 2：Prompt 越长越强

Claude Code 的设计告诉你，关键不是长，而是分层、稳定、可维护。

### 误区 3：所有上下文都该塞进 System Prompt

不是。不同信息要进不同通道，否则成本和缓存都会出问题。

## 本章小结

Claude Code 的 Prompt 设计体现出一个很成熟的思路：

- 用静态区承载长期不变的行为规则
- 用动态区承载会话和环境差异
- 用上下文分层保护缓存稳定性
- 用特殊机制处理高波动信息

## 下一步

- [02 — Agent Loop 核心循环](../02-agentic-loop/)：这些 prompt 最终是如何进入一次真实请求的
- [06 — Prompt Cache 优化](../06-prompt-caching/)：进一步看 Claude Code 为什么如此执着于保持 prompt 稳定
