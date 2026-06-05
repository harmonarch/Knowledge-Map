---
type: concept
status: evergreen
created: 2026-06-25
tags:
  - concept
  - react-native
  - mobile
  - build-system
  - js-engine
source: "React Native 跨平台进阶专栏 + 延伸研究"
---

# React Native 底层原理

## 一句话定义

React Native 的底层由三层技术栈构成：Metro 打包器（依赖解析 + Babel 转译 + Bundle 生成）、JS 引擎（Hermes / JSC 的编译策略与内存管理）、原生壳（Gradle / Xcode 编译原生代码并嵌入引擎）。理解这三层是跨过"会用"到"能排坑"的关键。

## 它解决什么问题

使用 React Native 开发时，很多问题表面是 JS 代码问题，根因在底层：
- 改了 `import` 路径 Metro 找不到 → 需要理解 resolver 和 Babel 的职责分工
- 启动慢、内存高 → 需要理解 Hermes 的 AOT 字节码策略 vs JSC/V8 的 JIT 策略
- 原生模块报错 → 需要理解 JS 引擎与原生代码的桥接机制
- 打包产物的结构 → 需要理解 bundle 是怎么生成、嵌入 App 的

这个框架帮你建立 React Native 的完整心智模型，让排坑有方向可循。

## 核心机制

### 1. 四层配置各管一件事（以路径别名为切入点）

配置 `@/` 指向 `src/` 需要在四个文件中分别配置，因为编译链路中有四个独立的模块解析节点：

| 配置文件 | 解决什么 | 失败后果 |
|------|------|------|
| `babel.config.js` | 代码转译时把 `@/xxx` 替换成实际相对路径 | 运行时找不到模块 |
| `tsconfig.json` | TypeScript 类型检查和编辑器智能提示 | IDE 报红、无法跳转 |
| `metro.config.js` | Metro 打包构建时的模块查找 | 打包失败 |
| `jest.config.js` | 单元测试环境的模块映射 | 测试跑不起来 |

关键认知：**四个配置文件独立工作，谁管谁的阶段，不存在"配了一个其他自动生效"。**

### 2. Metro 与 Babel 的关系

Metro 和 Babel 不是"先后"关系，而是**包含**关系——Metro 是打包器，在打包过程中调用 Babel 来转译每个文件：

```
源码: import from '@/components/Foo'
    ↓
Metro 解析依赖 → metro.config.js 的 resolveRequest 找到 src/components/Foo
    ↓
Metro 调用 Babel 转译 → babel.config.js 的 module-resolver 替换路径
    ↓
Metro 打包所有模块成 bundle → 交给 Hermes 引擎运行
```

### 3. JS 引擎三国：Hermes vs JavaScriptCore vs V8

| | Hermes | JSC | V8 |
|------|------|------|------|
| 开发者 | Meta | Apple | Google |
| 编译策略 | AOT → 字节码 | 解释 + 多层 JIT | 解释 + 多层 JIT |
| 启动速度 | 最快 | 中等 | 最慢 |
| 峰值性能 | 最低 | 高 | 最高 |
| 内存占用 | 最小 | 中等 | 最大 |

**设计哲学差异**：V8 追求极致运行速度（多层 JIT 把热代码优化到接近 C++），但引擎重、启动慢。Hermes 完全为移动端启动体验设计——砍掉 JIT，把编译挪到构建阶段，用 AOT 生成字节码，换来最快启动和最小内存。牺牲的是峰值性能，但在移动端场景下，启动速度比峰值性能更重要。

### 4. AOT vs JIT 的核心分歧

| | AOT | JIT |
|------|------|------|
| 编译时机 | 构建时 | 运行时 |
| 代表 | Hermes、Go、Rust、Swift | V8、JSC、JVM |
| 启动速度 | 快（产物直接可执行） | 慢（要经历预热过程） |
| 峰值性能 | 低（无运行时类型信息） | 高（投机优化热代码） |
| 性能可预测性 | 高（第 1 次和第 10000 次一样） | 低（GC 和去优化可能卡顿） |

**JIT "热代码越跑越快" 的原理**（以 V8 为例）：
```
第 1 次调用 → Ignition 解释器逐行执行（最慢）
连续调用几十次 → Sparkplug 基线编译成初步机器码
继续调用上千次 → TurboFan 根据收集的类型信息做激进优化
```

TurboFan 的投机优化：如果 `add(a, b)` 被调 1000 次且每次 a、b 都是整数，就生成一个"假设都是整数"的专用版本，跳过类型检查直接做 CPU 整数加法。但如果某次传了字符串，假设失败，触发去优化（deoptimization），退回慢版本。这就是 JIT "性能不可预测" 的根源。

### 5. 为什么 Hermes 编译成字节码而不编译成机器码

| | 机器码 | 字节码 |
|------|------|------|
| 执行方式 | CPU 直接执行 | 解释器逐条翻译 |
| 生成速度 | 慢 | 快 |
| 体积 | 大（同样逻辑 3-5 倍） | 小 |
| 跨平台 | 需要为 arm64/x86 分别生成 | 一份通用 |

V8 选机器码是因为浏览器场景下编译成本可被使用时长摊平，且 JIT 有运行时类型信息能编译出高质量机器码。Hermes 选字节码是因为：移动端没有 JIT 的运行时类型信息辅助，AOT 编译成机器码优化质量远不如 JIT，体积还翻好几倍。字节码是性价比最高的方案——跳过最慢的源码解析步骤，用解释执行运行，启动速度接近机器码，体积和内存却小得多。

### 6. 分代垃圾回收（Generational GC）

基于经验观察：大多数对象活得很短，少数对象活得很长。

| | 新生代 | 老生代 |
|------|------|------|
| 对象特征 | 刚创建，很快死掉 | 经历多次 GC 仍存活 |
| 空间大小 | 几 MB | 大 |
| GC 频率 | 高（每次快） | 低（每次慢） |
| 算法 | Scavenge（半空间复制） | 标记-清除 |

**Scavenge 算法**：新生代分 From / To 两半。新对象在 From 分配，From 快满了触发 GC——把活着的对象复制到 To，From 整块清空（死对象无需逐个释放），角色互换。活着的对象少就复制得少，死对象零开销。经历两次复制还活着 → 晋升到老生代。

**为什么不用标记-清除处理新生代**：标记-清除需要遍历所有对象标记存活者，而 Scavenge 只处理活着的对象——新生代大部分对象都死了，复制活着的比遍历全部快得多。

### 7. 构建链路

**开发模式**：Metro 不落盘，在内存中实时打包，通过 HTTP 把 bundle 推给手机。

**生产打包**：
```bash
npx react-native bundle \
  --platform android --dev false \
  --entry-file index.js \
  --bundle-output android/app/src/main/assets/index.android.bundle
```

产物对比：

| | Web | React Native |
|------|------|------|
| JS 产物 | `dist/index.js` | `index.android.bundle` / `main.jsbundle` |
| 资源 | `dist/assets/` | 拷贝到平台资源目录 |
| 部署 | 部署 dist 到服务器 | 嵌入 .apk / .ipa 包内 |

如果使用 CodePush 热更新，bundle 文件可以独立分发，角色类似 Web 的 dist。

### 8. yarn start / yarn run android / yarn run ios 的区别

```
yarn start        → 只启动 Metro Dev Server（HTTP 8081 + WebSocket HMR）
yarn run android  → 编译原生代码(Gradle) + 安装 APK + 如果需要则启动 Metro
yarn run ios      → 编译原生代码(Xcode) + 安装 APP + 如果需要则启动 Metro
```

日常开发：首次必须跑 `yarn run android/ios`（编译安装原生壳）。之后只改 JS 代码只需 `yarn start`——App 已安装，自动连 Metro 拿最新 bundle。只有改原生代码（`android/` 或 `ios/` 下的文件）才需重新跑 `yarn run android/ios`。

## 收益

- 路径别名配不通时知道该查哪个配置文件
- 启动性能问题时理解引擎层面的根本原因（Hermes 选型、JIT 开销、GC 策略）
- 打包流程出问题时能沿着 Metro → Babel → Bundle 链路排查
- 理解为什么 React Native 的产物结构和 Web 完全不同
- 知道"只改 JS 代码重新跑 yarn start 就行"的底层原因

## 代价

- 概念较多（Hermes/JSC/V8、AOT/JIT、GC、Metro、Gradle、Xcode），一次性消化有负担
- 日常业务开发不需要全部掌握——但排坑时需要
- 引擎和编译策略随 React Native 版本变化（如 Hermes 逐渐成为默认），需要持续关注

## 常见误区

- 以为配了 `babel.config.js` 的别名就万事大吉——TypeScript、Metro、Jest 各自需要独立配置
- 以为 Metro 和 Babel 是先后执行的独立工具——Metro 内部调用 Babel，是包含关系
- 以为 AOT 总是比 JIT 快——AOT 启动快但峰值性能低，取决于场景
- 以为 Hermes 是"低配版 V8"——设计哲学完全不同，Hermes 是主动牺牲峰值性能换启动速度

## 关联节点

- [[前端]] — React Native 属于前端领域的移动端分支
- [[React 架构思维总览]] — React 的核心渲染机制是 React Native 的上层逻辑
- [[React 并发模式与调度机制]] — React 18 的并发特性在 React Native 中同样生效
- [[浏览器多进程架构]] — 与 Hermes 的单引擎模型形成对比
- [[浏览器事件循环与渲染机制]] — JS 引擎的事件循环在 React Native 中同样存在

## 适用场景

- 配置 React Native 项目的路径别名、环境变量等
- 排查 Metro 打包失败、模块找不到的问题
- 分析启动性能瓶颈
- 理解 Hermes 引擎的行为差异（如某些 JS 特性的支持情况）
- 评估 React Native 项目的热更新方案

## 不适用场景

- Expo 托管项目——Expo 封装了大部分底层配置，直接操作这些文件可能被覆盖
- 纯 Web React 项目——Metro/Hermes 是 RN 专属，Web 用 Webpack/Vite
- 原生模块开发——需要更深入 JSI / Native Modules 的知识
