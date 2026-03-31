# 06 — Prompt Cache 优化

> Claude Code 不只是“会调用模型”，它还在认真优化每一轮调用到底要不要重算

## 为什么这一章值得单独讲

很多人看到 Prompt Cache，会以为那只是 API 层的小优化。

但从 Claude Code 源码看，它根本不是边角料，而是会反过来影响：

- Prompt 如何分层
- 哪些内容能进稳定前缀
- 哪些内容要迁移到消息层
- 哪些配置要在会话内保持稳定

换句话说：

> Cache 在这里不是“最后顺手加一下”，而是架构约束之一。

## 本章关键源码入口

| 文件 | 作用 |
|------|------|
| `src/constants/prompts.ts` | 定义静态/动态分界标记 |
| `src/utils/queryContext.ts` | 明确哪些内容属于 API cache-key 前缀 |
| `src/context.ts` | 生成会进入上下文前缀的系统/用户上下文 |
| `src/bootstrap/state.ts` | 保留某些会话级状态，避免无谓打爆缓存 |

## 新手先改掉一个过于简单的理解

很多文章会把 Claude Code 的缓存优化概括成：

> 它把 system prompt 分成静态和动态两部分。

这句话只说对了一半。

从 `src/utils/queryContext.ts` 的注释可以看出来，真正会参与 API 缓存前缀构建的，不只是 `systemPrompt`，还包括：

- `systemPrompt`
- `userContext`
- `systemContext`

所以 Claude Code 关心的并不是“System Prompt 要稳定”这么简单，而是：

> 整个请求前缀里，哪些内容必须稳定，哪些内容必须隔离。

## `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 的意义

在 `src/constants/prompts.ts` 里，最醒目的标记之一就是：

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

它本质上是在做一件事：

> 给 System Prompt 的组装结果打一条分界线，告诉后续逻辑哪里之前尽量稳定，哪里之后允许变化。

这条边界非常重要，因为一旦把高波动内容混进前缀，缓存收益就会迅速下降。

## 为什么 Claude Code 会对“动态内容泄漏”这么敏感

因为动态内容一旦混进稳定区域，就会引发连锁问题：

1. 这轮请求的前缀和上一轮不同
2. 缓存更难复用
3. 更容易发生 cache creation
4. 成本和延迟都上去

这也是为什么 Claude Code 源码里反复出现类似这种思路：

- 稳定内容尽量长期稳定
- 波动内容尽量移出稳定前缀
- 某些高波动内容甚至不要放进 System Prompt

## `CLAUDE.md` 为什么不进入稳定前缀

如果你已经看过上一章，就会更容易理解：

- `CLAUDE.md` 很重要
- 但它也很容易随着项目变化而变化

如果把它粗暴塞进稳定 prompt 前缀，会让本来可复用的部分变得更不稳定。

所以 Claude Code 更愿意把这类项目级规则放进 `userContext` 这一侧，由系统统一管理，而不是污染那条尽量稳定的前缀。

## MCP 指令为什么经常被“迁出” Prompt

MCP 是缓存优化里最典型的麻烦制造者之一。

原因很直白：

- MCP 服务器可能连上、断开、重载
- 每个服务器还可能带来自己的 `instructions`
- 这些内容本身就高波动

如果它们直接进入稳定 Prompt 前缀，那缓存几乎注定会频繁失效。

所以 Claude Code 源码才会特别强调：

- `mcp_instructions` 是危险的高波动段
- 更好的方案是把它迁移到 `mcp_instructions_delta` 这类消息级机制

这是一种非常典型的“为了缓存而重构数据放置位置”的工程思路。

## 为什么“工具描述”也会影响缓存

很多人只盯着 System Prompt，却忽略了工具描述同样会进入模型上下文。

这就带来一个新问题：

- 如果工具列表或工具描述常常变化
- 那模型侧看到的工具 schema 也会变化
- 这同样可能让缓存收益下降

所以 Claude Code 会尽量避免把高波动信息放进工具描述里，例如动态 agent 列表这类信息就不适合长期留在稳定 schema 中。

## 为什么源码里会有很多“会话内锁定”的状态

你在 `src/bootstrap/state.ts` 里能看到不少类似“latched”或“session-stable”思路的注释。

这背后反映的是一个很现实的问题：

> 某些开关、头信息、行为参数如果在会话中途来回变，会不断破坏缓存前缀。

所以 Claude Code 并不总是追求“最新状态立刻生效”，而是会权衡：

- 现在切换会不会打断缓存收益
- 某个值是否应该在一段会话内保持稳定
- 某些变化是否更适合在下一轮或下一会话再生效

这其实是很成熟的产品工程思维。

## 缓存优化为什么会反过来塑造架构

Claude Code 的一个很值得学习的地方在于：

它没有把缓存当成纯粹的底层实现细节，而是允许缓存需求反过来改变上层设计。

例如：

- Prompt 要分静态/动态
- `CLAUDE.md` 不适合直接塞进稳定 prompt
- MCP 指令要从 prompt 挪到 attachment
- 动态 agent 列表不能一直留在工具描述里

这说明缓存不是“后优化”，而是“前置约束”。

## 新手应该如何理解这件事

可以把 Prompt Cache 想成一个问题：

> 这轮请求里，哪些信息每次都差不多，值得复用；哪些信息变化太快，不该污染可复用区域？

Claude Code 的所有设计，几乎都在围绕这个问题不断做减法和隔离。

## 新手常见误区

### 误区 1：缓存优化就是加一个 API 参数

不对。它会影响 Prompt、上下文通道、工具 schema，甚至会影响状态管理方式。

### 误区 2：只要 System Prompt 稳定就够了

不对。真正的 cache-key 前缀比这更广，`userContext` 和 `systemContext` 也很重要。

### 误区 3：动态内容放哪都一样

不对。放在稳定前缀里和放在消息层里，缓存后果完全不同。

## 本章小结

Claude Code 的 Prompt Cache 优化告诉我们：

- 缓存是架构问题，不是纯 API 细节
- 稳定内容和高波动内容必须明确隔离
- Prompt、context、tool schema 都会影响缓存收益
- 真正成熟的系统会允许“缓存需求”反过来塑造数据放置方式

## 下一步

- [01 — System Prompt 分层设计](../01-system-prompt/)：回头再看静态/动态分层，会更能理解它为什么不是形式主义
- [09 — 启动性能优化](../09-startup-optimization/)：继续看 Claude Code 在缓存之外还做了哪些系统级性能优化
