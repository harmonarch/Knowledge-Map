---
type: concept
status: active
created: 2026-04-15
tags:
  - concept
  - database
  - sqlite
---

# Cloudflare D1

## 一句话定义

Cloudflare 的 Serverless SQL 数据库，底层是 SQLite，可以直接在 Workers 里查询，不需要连接池管理。

## 它解决什么问题

小型 Web 应用需要关系型数据库，但不想管数据库服务器、连接池、冷启动等问题。

## 出现前提

你需要存储用户数据、内容、配置等结构化数据，且规模在中小型范围内。

## 核心机制

底层 SQLite，通过 Wrangler CLI 创建和迁移。在 Worker 中通过 `env.DB.prepare()` 执行 SQL。

## 收益

- 500 万行读取 / 天免费
- 10 万行写入 / 天免费
- 5 GB 存储免费
- 与 Workers 原生集成，无冷启动
- SQL 语法，学习成本低

## 代价

- SQLite 的并发写入限制
- 不适合高并发写入场景
- 复杂查询性能不如 PostgreSQL

## 常见误区

- 以为 D1 能替代所有数据库 — 它适合中小应用和 MVP，大规模应用还是需要 PlanetScale / Supabase
- 以为 SQLite = 玩具 — SQLite 在读多写少场景下性能很强

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]

## 适用场景
- 小型 Web 应用的数据存储
- 用户数据、配置、内容管理
- 原型开发和 MVP 阶段

## 一个真实例子

```bash
wrangler d1 create my-database
wrangler d1 execute my-database --file ./schema.sql
```

```javascript
export default {
  async fetch(request, env) {
    const { results } = await env.DB.prepare(
      'SELECT * FROM users WHERE email = ?'
    ).bind('user@example.com').all();
    return Response.json(results);
  }
};
```
