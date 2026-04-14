---
type: concept
status: active
tags:
  - concept
  - ai
  - vector-database
  - rag
---

# Vectorize

## 一句话定义

Cloudflare 的向量数据库，存储文本/图片被 AI 模型转换成的语义向量，用于相似度搜索，做 RAG 必备。

## 它解决什么问题

RAG（检索增强生成）需要一个向量数据库来存储和检索语义相近的文档片段，让 LLM 的回答有依据。

## 出现前提

你在做 RAG 应用：用户提问 → 找到相关文档 → 把文档和问题一起发给 LLM。

## 核心机制

1. 文档喂给嵌入模型 → 得到向量
2. 向量存到 Vectorize
3. 用户提问 → 问题转成向量 → 在 Vectorize 里找最相近的文档片段
4. 相关文档 + 用户问题一起发给 LLM → 得到有依据的回答

## 收益

- 30,000 个向量免费
- 每月 3000 万次查询免费
- 支持 1536 维（够用 OpenAI embedding）
- 与 Workers / Workers AI 原生集成

## 代价

- 免费额度的向量数量有限，大规模知识库可能不够
- 功能不如 Pinecone / Weaviate 丰富

## 常见误区

- 以为向量数据库很复杂 — 核心就是 insert 和 query 两个操作
- 以为必须用 OpenAI 的 embedding — Workers AI 自带免费的嵌入模型

## 关联节点
- [[Cloudflare]]
- [[Workers AI]]
- [[Cloudflare Workers]]
- [[AI Agent]]

## 适用场景
- RAG 应用（知识库问答）
- 语义搜索
- 推荐系统

## 一个真实例子

```javascript
// 插入向量
await env.VECTORIZE.insert([{
  id: 'doc-001',
  values: [0.12, 0.55, 0.88, ...],
  metadata: { text: '文档原文内容', source: 'article-1.pdf' }
}]);

// 相似度搜索
const results = await env.VECTORIZE.query(queryVector, {
  topK: 5,
  returnMetadata: true,
});
```
