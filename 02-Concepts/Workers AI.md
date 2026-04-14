---
type: concept
status: active
tags:
  - concept
  - ai
  - inference
---

# Workers AI

## 一句话定义

Cloudflare 的边缘推理服务，直接在 Workers 里调用开源 AI 模型，不需要自己部署 GPU。

## 它解决什么问题

想用 AI 模型但不想付 OpenAI 费用，或者有隐私敏感数据不想发到第三方。

## 出现前提

你需要文本生成、图像生成、文本嵌入、语音识别等 AI 能力，且对模型质量要求不极致。

## 核心机制

Cloudflare 在边缘节点部署了 GPU，通过 `env.AI.run()` 调用预置的开源模型。

支持的模型类型：
- 文本生成：Llama 3、Mistral、Gemma
- 图像生成：Stable Diffusion XL
- 文本嵌入：BGE、多语言嵌入模型
- 语音识别：Whisper
- 图像/文本分类：ResNet 等

## 收益

- 每天 10,000 神经元免费（约几百次小模型推理）
- 数据不出边缘节点，隐私友好
- 与 Workers 原生集成

## 代价

- 模型选择有限，只有 Cloudflare 预置的
- 推理质量不如 GPT-4 / Claude 等商业模型
- 大模型推理速度受边缘 GPU 限制

## 常见误区

- 以为能替代 OpenAI — 适合辅助任务（分类、摘要、嵌入），核心对话还是用商业模型
- 以为免费额度很少 — 对开发测试阶段够用

## 关联节点
- [[Cloudflare]]
- [[Cloudflare Workers]]
- [[Vectorize]]
- [[AI Agent]]

## 适用场景
- 原型阶段：不想付费，用免费开源模型测试想法
- 辅助任务：分类、摘要、嵌入等
- 隐私敏感场景：数据不出边缘节点

## 一个真实例子

```javascript
export default {
  async fetch(request, env) {
    const response = await env.AI.run('@cf/meta/llama-3-8b-instruct', {
      messages: [
        { role: 'user', content: '用一句话解释什么是 Cloudflare Workers' }
      ],
    });
    return Response.json(response);
  }
};

// 生成文本嵌入（用于 RAG）
const embeddings = await env.AI.run('@cf/baai/bge-base-en-v1.5', {
  text: ['这是要转换成向量的文本'],
});
```
