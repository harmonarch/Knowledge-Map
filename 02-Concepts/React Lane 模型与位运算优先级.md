---
type: concept
status: active
created: 2026-05-17
tags:
  - concept
  - react
  - lane
  - bitwise
  - priority
  - scheduler
---

# React Lane 模型与位运算优先级

## 一句话定义

React 用 31 位二进制掩码（Lane）表示更新的优先级，值越小优先级越高，通过位运算实现优先级的合并、判断、删除和最高优先级提取。

## 它解决什么问题

React 需要同时管理多种不同优先级的更新（同步输入、过渡动画、空闲任务等），如果用传统的枚举或数组来表示和管理优先级集合，判断包含关系、合并、删除等操作会涉及遍历和比较，性能开销大。Lane 用单个整数的位运算一次性完成这些操作，既高效又简洁。

## 出现前提

- 不同来源的更新有明确的优先级差异（用户输入 > 数据加载 > 离屏渲染）
- 同一时刻可能存在多个待处理的优先级
- 调度器需要快速判断“当前最高优先级是什么”
- 需要一套从 Lane 到 Scheduler 优先级的映射机制

## 核心机制

### Lane 分层结构

| Lane 类型 | 位范围 | 用途 |
|---|---|---|
| `SyncLane` | 第 2 位 | 同步更新，最高优先级 |
| `InputContinuousLane` | 第 4 位 | 连续输入（拖拽、滚动） |
| `DefaultLane` | 第 6 位 | 默认更新 |
| `TransitionLanes` | 第 7–21 位 | 过渡更新，共 15 条 Lane |
| `RetryLanes` | 第 22–25 位 | 重试，共 4 条 Lane |
| `SelectiveHydrationLane` | 第 26 位 | 选择性注水 |
| `IdleLane` | 第 30 位 | 空闲任务 |
| `OffscreenLane` / `DeferredLane` | 第 31–32 位 | 离屏渲染与延迟任务 |

### 四种核心位运算

1. **`&` 按位与 —— 判断包含关系**：同位都为 1 才得 1，用于判断某个 Lane 是否属于某个集合。源码封装为 `includesTransitionLane()` 和 `intersectLanes()`。

2. **`|` 按位或 —— 合并优先级**：同位都为 0 才得 0，用于合并多个优先级。合并后值变大，以较低优先级为准。源码封装为 `mergeLanes()`。典型用例：`SyncUpdateLanes = SyncLane | InputContinuousLane | DefaultLane`。

3. **`~` 按位取反 + `&` —— 删除优先级**：每一位 0 变 1、1 变 0，再与原集合求交，被删位归零。源码封装为 `removeLanes(set, subset)`，即 `set & ~subset`。

4. **`lanes & -lanes` —— 取最高优先级**：利用补码（`-x = ~x + 1`）提取集合中值最小的那个位。源码封装为 `getHighestPriorityLane()`。推导过程：`lanes = 0b00001110`，`-lanes = 0b11110010`，`&` 后得 `0b00000010`，即最低有效位（最高优先级）。

### Lane → EventPriority → SchedulerPriority 转换链

Lane 数量太多不宜直接用于调度，React 设计了 EventPriority 作为中间收敛层：

| EventPriority | 对应 Lane | Scheduler 等级 |
|---|---|---|
| `DiscreteEventPriority` | `SyncLane` | `ImmediateSchedulerPriority` |
| `ContinuousEventPriority` | `InputContinuousLane` | `UserBlockingSchedulerPriority` |
| `DefaultEventPriority` | `DefaultLane` | `NormalSchedulerPriority` |
| `IdleEventPriority` | `IdleLane` | `IdleSchedulerPriority` |

转换函数 `lanesToEventPriority()` 先取最高优先级 Lane，再逐级比对，最终在 `scheduleTaskForRootDuringMicrotask` 中通过 switch 映射到 Scheduler 等级。

### 位运算的通用性 —— 权限系统类比

同样的位运算模式适用于权限系统：

```ts
const VIP1 = 0b00000001, VIP2 = 0b00000010;
const user = VIP1 | VIP2;          // 合并权限
const updated = user & ~VIP1;       // 删除权限
const highest = updated & -updated; // 取最高权限
```

## 收益

- 一次位运算完成优先级集合的合并/判断/删除，O(1) 复杂度
- Lane 的位范围划分让不同更新类型天然隔离
- 通过 `lanes & -lanes` 一行代码提取最高优先级，无需遍历
- EventPriority 中间层将细粒度 Lane（31 位）收敛为 5 个调度等级，简化调度逻辑

## 代价

- 位运算的语义不够直观，阅读源码时需要心算二进制
- Lane 的位范围分配是硬编码的，扩展需要调整位偏移
- TransitionLanes 有 15 条，理解它们之间的优先级差异需要额外背景

## 常见误区

- **"Lane 值越大优先级越高"**：实际上值越小优先级越高，`SyncLane`（第 2 位）是最小值也是最高优先级。
- **"Lane 和 SchedulerPriority 是一一对应的"**：中间还有 EventPriority 这一层，Lane 先映射到 EventPriority，再映射到 SchedulerPriority。
- **"`lanes & -lanes` 是特殊语法"**：这是补码的基本性质——一个数和它的补码相与，结果是最低有效位。

## 关联节点
- [[前端]]
- [[React 并发模式与调度机制]]
- [[React useState 环形链表与更新机制]]

## 适用场景
- 阅读 `ReactFiberLane.js` 源码时理解位运算的含义
- 理解为什么 `startTransition` 的更新优先级低于普通 `setState`
- 设计自己的优先级或权限系统时参考位运算模式
- 排查“为什么我的更新被延迟了”时理解 Lane 的分配逻辑

## 一个真实例子

用户在搜索框中连续输入，同时触发搜索结果列表的过渡更新。输入框的 `SyncLane` 和 `InputContinuousLane` 优先级远高于列表的 `TransitionLane`。`getHighestPriorityLane()` 始终返回输入相关的 Lane，调度器据此分配 `ImmediateSchedulerPriority`，确保输入反馈不被列表渲染阻塞。

## 信息来源

- 来源：[usehook - React 原理专栏 / 13. Lane 模型](https://reactprinciple.usehook.cn/13.lane)
- 原始笔记：`/Users/rhazen/Library/Application Support/Dia/User Data/Default/AgentServer/contexts/ED2DD252-6739-4750-903D-DB6CD4C8423C/work/React Lane 模型总结.md`
- 整理日期：2026-05-17
