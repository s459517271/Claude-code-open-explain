# 10 — Feature Flag 体系

> 在 Claude Code 里，Feature Flag 不只是开关，它还是产品实验、构建裁剪和架构演进的入口

## 为什么这一章不只是“列几个 Flag”

新手第一次看到 Claude Code 源码里的 `feature('...')`，很容易觉得这只是普通条件分支。

但在这个项目里，Feature Flag 至少承担了 3 种角色：

- 控制哪些功能对外可见
- 决定哪些代码在构建时直接被删除
- 暴露产品正在尝试的未来方向

所以读 Flag，不是在读琐碎开关，而是在读 Claude Code 的演进轨迹。

## 本章关键源码入口

| 文件 | 作用 |
|------|------|
| `src/main.tsx` | 入口层大量 Feature Flag 条件导入 |
| `src/tools.ts` | 工具层 Feature Flag 控制 |
| `src/commands.ts` | 命令层 Feature Flag 控制 |
| `src/services/analytics/growthbook.ts` | 运行时 Flag 获取逻辑 |

## 先分清两类 Flag

Claude Code 的 Flag 大体可以分成两层：

### 1. 构建时 Flag

典型形式：

```typescript
feature('KAIROS')
```

这一类 Flag 的重点不只是“运行时是否启用”，更重要的是：

> 如果构建时就知道这个分支不会用，构建器可以直接把整段代码删掉。

### 2. 运行时 Flag

典型形式会和 GrowthBook 配合，例如：

```typescript
getFeatureValue_CACHED_MAY_BE_STALE(...)
```

这一类 Flag 更偏向：

- 灰度发布
- A/B 测试
- 会话内行为控制

## 为什么构建时 Flag 这么重要

如果没有构建时 Flag，很多实验性模块即使“没开”，也仍可能：

- 被打进包里
- 参与模块解析
- 增加启动负担
- 影响依赖关系

Claude Code 选择让一部分 Flag 在构建期就产生作用，本质上是在做一件事：

> 不只是把功能藏起来，而是让不用的功能根本不存在于这份构建里。

这对大型 CLI 来说非常关键。

## 为什么运行时 Flag 也不能少

构建时 Flag 很强，但它解决不了所有问题。

真实产品还需要：

- 对部分功能先给少量用户
- 做实验对比
- 在不同用户群体里逐步放量
- 允许某些行为在不重新发版的情况下调整

这就是运行时 Flag 的价值。

所以 Claude Code 实际采用的是：

> 构建期裁掉一批，运行期再细调一批。

## 从源码能看出哪些产品方向

只看 Flag 名字，就能看到很多线索，例如：

- `KAIROS`
- `PROACTIVE`
- `COORDINATOR_MODE`
- `VOICE_MODE`
- `HISTORY_SNIP`
- `KAIROS_BRIEF`
- `KAIROS_GITHUB_WEBHOOKS`
- `EXPERIMENTAL_SKILL_SEARCH`

即使你还没读完整个实现，也能先推断出 Claude Code 正在探索：

- 更主动的 agent 行为
- 更强的多 Agent 协调
- 语音和通知类能力
- 更细的上下文压缩手段
- 更丰富的技能与自动化工作流

## 但读 Flag 时一定要避免一个误区

> 有 Flag 不等于功能已经稳定，也不等于对外公开。

源码里能看到某个能力存在，并不代表：

- 当前构建一定带上了它
- 普通用户一定能触发它
- 它已经达到正式发布标准

所以 Flag 更适合作为“产品演进信号”，而不是“功能清单”。

## 为什么 `feature()` 和性能优化强相关

这一点很多人一开始不会想到。

当 `feature()` 参与构建期裁剪时，它直接影响：

- 包体大小
- 模块加载数量
- 启动时间
- 初始化时依赖图复杂度

也就是说，在 Claude Code 里，Feature Flag 不只是产品实验工具，还是性能工程工具。

## 为什么运行时 Flag 常常带着“可能过时缓存”语义

源码里常见的名字像 `getFeatureValue_CACHED_MAY_BE_STALE(...)`，这个命名本身就很有信息量。

它传达了一个务实立场：

> 有些 Flag 值不需要每秒都绝对新鲜，允许在一段会话内保持稳定，反而更利于系统整体行为稳定。

这和 Prompt Cache 一章是相通的。  
Claude Code 很清楚：不是所有状态都值得实时抖动。

## 用户类型为什么也像一层隐式 Flag

源码里除了显式的 `feature('...')`，还经常会看到类似：

- `process.env.USER_TYPE === 'ant'`

这说明 Claude Code 的功能暴露不仅由产品 Flag 控制，还和用户分层、内部外部环境相关。

从架构上看，这等于又加了一层能力门控：

- 某些逻辑只在内部环境出现
- 某些提示词或行为只对内部实验开放
- 某些模式即使代码存在，也不会对外走到

## 新手应该怎样用这一章辅助读源码

一个很实用的方法是：

### 先看 Flag 名，再看它包住了什么

这样你更容易分辨：

- 这是核心主链路
- 这是实验性分支
- 这是未来方向预留

### 再判断它影响的是哪一层

- 启动层
- 命令层
- 工具层
- Prompt 层
- 多 Agent 层
- MCP 层

这样你会比单纯“搜关键字”更容易建立结构感。

## 新手常见误区

### 误区 1：Feature Flag 只是产品经理开关

不对。在 Claude Code 里，它还承担构建裁剪和架构隔离职责。

### 误区 2：看到 Flag 就等于看到了可用功能

不对。很多 Flag 只是说明能力存在，不代表当前构建或当前用户可用。

### 误区 3：Flag 越多说明架构越乱

不一定。关键要看 Flag 是否帮助隔离实验、控制发布和降低主路径负担。

## 本章小结

Claude Code 的 Feature Flag 体系说明：

- Flag 既是产品发布机制，也是工程裁剪机制
- 构建时 Flag 影响最终代码形态
- 运行时 Flag 影响灰度与实验
- Flag 名字本身就能透露产品方向
- 理解 Flag，等于理解 Claude Code 的一部分演进路线图

## 下一步

- [09 — 启动性能优化](../09-startup-optimization/)：回头再看，你会更明白 Feature Flag 为什么能直接影响启动表现
- [01 — System Prompt 分层设计](../01-system-prompt/)：很多 Prompt 差异和能力差异也会受到用户类型与功能开关影响
