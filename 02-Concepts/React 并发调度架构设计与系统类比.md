---
type: concept
status: active
created: 2026-05-18
tags:
  - concept
  - react
  - concurrency
  - scheduler
  - architecture
  - os-analogy
---

# React 并发调度架构设计与系统类比

## 一句话定义

React 并发模式本质上是把渲染从"一口气做完"改造为"可暂停、可恢复、可插队"的协作式调度，在单线程 JavaScript 主线程里模拟了操作系统的时分与抢占调度思想。

## 它解决什么问题

长任务独占主线程导致 UI 无响应。React 需要在不引入多线程的前提下，让高优先级交互（点击、输入）能插队执行，同时保证低优先级任务（大数据列表渲染）不会被彻底饿死。

## 出现前提

- React 16 之前的递归渲染不可中断，一旦开始就必须跑完整棵树
- Fiber 架构（React 16）提供了可中断的数据结构基础
- 浏览器提供了 MessageChannel 宏任务机制，可以在帧间隙推进工作
- React 18 之后 Concurrent Mode 正式落地，调度机制趋于稳定

## 核心机制

### 1. 十句话讲清并发调度

1. 渲染被拆成一个个小单元——每个 Fiber 节点是一个工作单元，处理完一个检查一次是否让出。
2. 判断依据是 `shouldYield()`——当前时间片（5ms）用完就让出，没用完就继续。
3. 让出后通过 MessageChannel 发一个宏任务，下一帧浏览器处理完绘制和输入后再继续渲染。
4. 如果中途有更高优先级的更新进来，React 丢弃当前低优先级的半成品，优先处理高优先级。
5. 调度的入口是 `ensureRootIsScheduled`——对比新旧任务优先级，决定复用还是取消重调。
6. 同一事件内多次 `setState` 共享同一个调度任务，这就是自动批处理。
7. Scheduler 维护一个按过期时间排序的小顶堆 `taskQueue`，`workLoop` 不断取堆顶任务执行，超时就 break。
8. 优先级分三层：事件类型决定 Lane → Lane 映射为 Scheduler 优先级 → Scheduler 优先级决定过期时间。
9. Lane 用二进制位表示优先级，合并用 `|`、比较用 `&`、取最高用 `lanes & -lanes`，全是 O(1) 位运算。
10. `workLoopConcurrent` 每处理完一个 Fiber 检查 `shouldYield()`，超时则保留 `workInProgress` 指针作为断点；后续从断点继续。

### 2. 架构师视角：五项设计决策

1. **架构目标**：把渲染从不可中断的递归改造成可中断、可恢复的协作式调度，解决长任务阻塞 UI 的核心问题。
2. **调度层设计**：独立的 Scheduler 模块用小顶堆管理任务队列，MessageChannel 宏任务驱动执行循环，每 5ms 主动让出线程——本质上是单线程里模拟操作系统的抢占式调度。
3. **数据结构选型**：Fiber 树让每个节点成为可独立处理的工作单元；Lane 用 31 位二进制位图表达优先级，任务合并、子集判断、最高优先级选取全部降为 O(1) 位运算。
4. **中断与恢复**：超时则保留 `workInProgress` 指针作为断点继续跑；高优先级插队则丢弃当前半成品树，用新的 `renderLanes` 重新开始。
5. **批处理收敛**：`ensureRootIsScheduled` 作为所有更新的收口函数，利用微任务时序将同一事件循环内的多次 `setState` 合并为一次调度。

### 3. 批处理版本演进

| 版本 | 行为 |
|------|------|
| React 16–17 | 只在合成事件内自动批处理；setTimeout、Promise、原生事件中每次 setState 都触发渲染 |
| React 18 | 引入 Automatic Batching，所有场景自动合并（通过 `ensureRootIsScheduled` + 微任务） |
| React 19 | 沿用 18 机制，优化调度路径 |

### 4. React 调度 vs 操作系统抢占式调度

| 维度 | 操作系统 | React Scheduler |
|------|---------|----------------|
| 中断触发 | 硬件时钟中断（每 10ms） | `shouldYield()` 检查（每个 Fiber 完成后） |
| 上下文切换 | 内核强制切换 | 主动 break workLoop + MessageChannel 让出 |
| 状态保存 | PCB 保存寄存器/栈 | `workInProgress` 指针保存 Fiber 树断点 |
| 优先级 | 进程优先级（nice 值） | Lane 优先级（二进制位图） |
| 就绪队列 | 红黑树 | taskQueue（小顶堆，按 expirationTime） |
| 时间片 | 时间片轮转（Round Robin） | 5ms frameInterval 轮转 |

**关键差异**：

- **粒度**：OS 在任意指令处中断；React 只能在 Fiber 节点边界让出——单个组件 render 太慢时时间切片救不了。
- **协作式 vs 强制式**：OS 不需要进程配合；React 需要 `workLoopConcurrent` 每次循环显式调用 `shouldYield()`。
- **"抢占"的实质**：React 等当前 Fiber 处理完才响应高优先级，延迟最多一个 Fiber 的处理时间。

**一句话总结**：React 用「高频检查点 + 主动让出 + 断点恢复」在单线程里模拟了多线程 OS 的时间片轮转和优先级抢占，代价是中断粒度受限于 Fiber 节点边界。

### 5. 完整流程（按时间顺序）

```
setState()
  → ensureRootIsScheduled（对比优先级、决定是否新建调度）
    → 微任务合并（同一事件内多次更新收敛为一次）
      → Scheduler 创建 task，推入 taskQueue
        → MessageChannel 触发宏任务
          → performWorkUntilDeadline → workLoop
            → 取堆顶 task → performConcurrentWorkOnRoot
              → prepareFreshStack → workLoopConcurrent
                → beginWork（处理 Fiber）
                  → shouldYield()?
                    → NO: 继续下一个 Fiber
                    → YES: break，返回续体函数，下一帧继续
              → 全部 Fiber 处理完 → completeWork
                → commitRoot → 同步提交 DOM 变更
```

## 收益

- 从 OS 调度的高度理解 React 的设计意图，建立系统观
- 通过类比快速掌握小顶堆、时间片、优先级抢占在 React 中的具体落地
- 批处理版本演进表帮助理解 React 18 的 Automatic Batching 是解决什么历史问题
- 端到端流程图为阅读源码和排查问题提供全局导航

## 代价

- 协作式让出的粒度受限于 Fiber 单元，单个组件 render 过重时仍然会卡顿
- 低优先级 render 可能做到一半被废弃，产生额外重算成本
- 三层优先级映射（事件→Lane→Scheduler）增加了调试的间接性

## 常见误区

- **"React 真的实现了抢占式调度"**：React 是协作式调度，只能在 Fiber 边界主动让出，不是像 OS 那样任意时刻强制切换。
- **"时间切片能让任何渲染都不卡"**：如果一个组件自身 render 就耗时 50ms，时间切片没法在中间打断，只能等它跑完。
- **"小顶堆就绪队列和 OS 的完全一样"**：OS 用红黑树，React 用小顶堆——数据结构不同但思想一致：始终优先取最高优先级的任务。

## 关联节点
- [[前端]]
- [[React 并发模式与调度机制]]
- [[React Lane 模型与位运算优先级]]
- [[React useState 环形链表与更新机制]]
- [[浏览器事件循环与渲染机制]]

## 适用场景
- 从系统架构高度理解 React 调度设计，而非停留在 API 层面
- 面试或技术分享中解释"React Fiber 为什么快"时给出结构化答案
- 将前端调度与操作系统知识串联，建立跨领域的知识网络
- 分析 `startTransition`、`useDeferredValue` 等 API 的设计动机时溯源到调度层
- 排查并发模式下的疑难问题（任务为什么被打断、为什么被重做、为什么饥饿）

## 一个真实例子

用户在搜索框快速输入 "hello"，每次按键触发搜索列表更新。React 不是每按一个键就完整渲染一次——`ensureRootIsScheduled` 在微任务中将多次按键的更新合并，Scheduler 以高优先级（InputContinuousLane）调度渲染。如果用户打字速度超过渲染速度，低优先级的中间结果直接被丢弃，只渲染最新输入对应的列表。这就是协作式调度在真实交互中的运作方式。

## 信息来源

- 来源：[usehook - React 原理专栏](https://reactprinciple.usehook.cn/)（react@19.0.0，第 12–24 章）
- 原始笔记：`/Users/rhazen/Library/Application Support/Dia/User Data/Default/AgentServer/contexts/83B9A0FE-710B-448D-A03E-930867918822/work/React并发模式总结.md`
- 整理日期：2026-05-18
