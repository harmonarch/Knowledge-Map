---
type: concept
status: active
created: 2026-04-15
tags:
  - concept
  - serverless
  - backend
---

# Cloudflare Workers

## 一句话定义

Cloudflare 的 Serverless 计算平台，代码运行在全球 300+ 边缘节点上，用户在哪里代码就在哪里执行。

## 它解决什么问题

小开发者需要后端 API 但不想管服务器。Workers 让你写一段 JS/TS/Python/Rust，直接部署到全球边缘，延迟极低。

## 出现前提

你需要 API 接口、请求转发、鉴权中间件、定时任务、Webhook 处理等后端逻辑。

## 核心机制

基于 V8 Isolate（不是容器），冷启动极快。每次 HTTP 请求触发一个 `fetch` handler，处理完返回响应。通过 `wrangler.toml` 配置绑定 D1 / R2 / KV 等资源。

## 收益

- 免费 10 万次请求/天
- 全球边缘部署，延迟低
- 冷启动极快（V8 Isolate）
- 支持 Cron Triggers 定时任务

## 代价

- 一次请求最多 30 秒（免费版 10ms CPU 时间）
- 没有文件系统，需配合 R2 存文件
- 内存有限，不适合跑重型任务

## 常见误区

- 以为 Workers 是传统的长时运行服务 — 它是请求驱动的，不是常驻进程
- 以为 CPU 时间 = 墙钟时间 — 10ms CPU 时间实际上可以处理很多 I/O 密集型任务

## 关联节点
- [[Cloudflare]]
- [[Cloudflare R2]]
- [[Cloudflare D1]]
- [[Cloudflare KV]]
- [[Cloudflare AI Gateway]]

## 适用场景
- API 接口（对接 AI 服务、数据库查询）
- 请求转发、鉴权中间件
- 定时任务（Cron Triggers）
- Webhook 处理

## 一个真实例子

```javascript
// src/index.js
export default {
  async fetch(request) {
    const url = new URL(request.url);
    if (url.pathname === '/api/hello') {
      return Response.json({ message: 'Hello from Cloudflare Workers!' });
    }
    return new Response('Not Found', { status: 404 });
  }
};
```

```bash
wrangler deploy  # 上线，跑在全球边缘节点
```
