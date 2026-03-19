---
title: 如何在 OpenClaw 上使用 GPT 订阅：别把 ChatGPT 订阅和 API Key 搞混
description: 想把 OpenAI 的 GPT 能力接到 OpenClaw 里？这篇文章讲清楚 ChatGPT 订阅、OpenAI API Key、Codex OAuth 以及推荐接入方式。
pubDate: 2026-03-19
updatedDate: 2026-03-19
featured: false
tags:
  - OpenClaw
  - OpenAI
  - ChatGPT
  - GPT
  - Codex
---

很多人第一次接 OpenClaw 时，都会卡在同一个问题上：

> 我已经买了 ChatGPT Plus / Business / Pro，为什么在 OpenClaw 里还不能直接用 `openai/gpt-*`？

答案很简单：**ChatGPT 订阅** 和 **OpenAI API** 不是一回事。

如果你想在 OpenClaw 上稳定使用 OpenAI 的 GPT 能力，实际上有 **两条路线**：

1. **API Key 路线**：适合想用标准 OpenAI provider 的人
2. **ChatGPT OAuth / Codex 路线**：适合想用 OpenAI Codex（ChatGPT OAuth）的人

这篇文章就把这件事一次讲清楚。

## 先说结论

如果你只想先知道该怎么选，直接看这里：

- 想用 `openai/gpt-5.4`、`openai/gpt-5.4-pro` 这类 **标准 OpenAI 模型名** → **用 OpenAI API Key**
- 想走 **ChatGPT 订阅授权**，不折腾 API Key → **用 `openai-codex` provider 的 OAuth 登录**
- **不要把 ChatGPT Plus/Business 当成 API 额度本身**，这两个体系是分开的

OpenClaw 官方文档里写得很明确：

- `openai` provider 的认证方式是 **`OPENAI_API_KEY`**
- `openai-codex` provider 的认证方式是 **OAuth（ChatGPT）**

也就是说：

- **API Key 对应 `openai`**
- **ChatGPT OAuth 对应 `openai-codex`**

这点搞清楚，后面基本就顺了。

---

## 路线一：用 OpenAI API Key（最直接、最标准）

如果你追求的是最标准、最清晰、最容易排查问题的方式，那就直接用 **OpenAI API Key**。

### 适合谁

适合这些场景：

- 你希望在 OpenClaw 里直接使用 `openai/gpt-*` 模型
- 你想清楚地按 API 计费
- 你不想把“ChatGPT 网页订阅”和“程序调用”混在一起
- 你要做更稳定的自动化、代理、工作流编排

### 官方对应关系

OpenClaw 文档中对 OpenAI provider 的说明是：

- Provider：`openai`
- Auth：`OPENAI_API_KEY`
- 示例模型：`openai/gpt-5.4`、`openai/gpt-5.4-pro`

### 最常见的接法

初始化时直接走向导：

```bash
openclaw onboard --auth-choice openai-api-key
```

如果只是想看当前模型状态，也可以用：

```bash
openclaw models status
```

### 典型配置思路

比如把默认模型切成 OpenAI：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.4"
      }
    }
  }
}
```

### 这条路线的优点

- 最符合“程序调用模型”的习惯
- 模型命名清晰，和文档一致
- 问题最好查
- 对多 Agent、多工作流也更自然

### 这条路线最容易踩的坑

最大的坑就是：

> **ChatGPT Plus / Business / Pro 订阅，不等于 OpenAI API 余额。**

你就算是 ChatGPT 付费用户，如果没有 API Key，OpenClaw 里的 `openai/...` 也不会凭空能用。

这是很多人第一次接 OpenClaw 时最容易误会的地方。

---

## 路线二：用 ChatGPT 订阅登录 OpenClaw（OpenAI Codex OAuth）

如果你更想复用你在 ChatGPT / OpenAI 侧已有的订阅授权，而不是单独配 API Key，那么 OpenClaw 官方支持另一条路线：

**OpenAI Codex OAuth**。

### 官方怎么定义这条路线

OpenClaw 文档中写得非常明确：

- Provider：`openai-codex`
- Auth：**OAuth（ChatGPT）**
- 示例模型：`openai-codex/gpt-5.4`
- 并且官方明确说明：**OpenAI Codex OAuth 支持在 OpenClaw 这类外部工具中使用**

这意味着，如果你想写一篇“如何在 OpenClaw 上使用 GPT 订阅”的教程，最应该强调的其实就是：

> 在 OpenClaw 里，**“订阅式登录”这件事，官方对应的是 `openai-codex`，不是普通 `openai` provider。**

### 推荐命令

最常见的登录方式：

```bash
openclaw onboard --auth-choice openai-codex
```

或者直接登录 provider：

```bash
openclaw models auth login --provider openai-codex
```

登录完成后，可以检查状态：

```bash
openclaw models status
```

### 登录过程会发生什么

根据 OpenClaw 的 OAuth 文档，这条链路本质上是一个标准的 **PKCE OAuth 流程**：

1. 生成 PKCE verifier / challenge 和随机 state
2. 打开 OpenAI 授权页面
3. 尝试监听本地回调 `http://127.0.0.1:1455/auth/callback`
4. 如果本地回调拿不到，就手动粘贴 redirect URL 或 code
5. 再去交换 token
6. 把 access / refresh / expires / accountId 存起来

对普通用户来说，你不用死记这些细节，只要知道一件事：

**这条路不是填 API Key，而是走网页登录授权。**

### 什么时候更适合这条路线

- 你已经在 OpenAI / ChatGPT 侧有可用订阅与账号
- 你想走 OAuth，而不是自己管理 API Key
- 你更倾向于“登录账号”而不是“配置开发者密钥”

---

## 两条路线到底怎么选？

可以用一句很实用的话来判断：

### 选 API Key，如果你要的是“模型调用能力”

你想要的是：

- 标准 `openai/...` provider
- 清晰的模型切换
- 程序化、工程化、自动化
- 易排错、易迁移

那就选 **OpenAI API Key**。

### 选 ChatGPT OAuth，如果你要的是“订阅账号授权接入”

你想要的是：

- 不想单独管理 API Key
- 更接近“我已经买了 ChatGPT，能不能直接接入 OpenClaw”
- 愿意使用 `openai-codex/...` 这个 provider 路线

那就选 **OpenAI Codex OAuth**。

---

## 一个非常重要的认知：订阅和 provider 是绑定关系

很多教程的问题，不是步骤不对，而是把概念写混了。

在 OpenClaw 里，正确理解应该是：

| 你手里的东西 | 在 OpenClaw 里对应什么 |
|---|---|
| OpenAI API Key | `openai` provider |
| ChatGPT OAuth / Codex 登录 | `openai-codex` provider |

所以如果你：

- 用的是 ChatGPT 授权
- 却又想直接选 `openai/gpt-5.4`

那就很容易一脸懵：

> 为什么登录了还是不能用？

因为你登录的入口和你选的 provider，根本不是同一条线。

---

## 推荐的新手上手顺序

如果你是第一次折腾，建议按下面顺序来：

### 方案 A：想最稳

1. 先走 API Key
2. 跑 `openclaw onboard --auth-choice openai-api-key`
3. 把默认模型切到 `openai/gpt-5.4`
4. 用 `openclaw models status` 确认状态

这套最不容易混乱。

### 方案 B：你就是想用订阅登录

1. 跑 `openclaw onboard --auth-choice openai-codex`
2. 完成网页授权
3. 检查 `openclaw models status`
4. 选用 `openai-codex/gpt-5.4` 这类模型

这套更贴近“订阅接入”的心智模型。

---

## 多账号 / 多环境怎么处理

OpenClaw 的 OAuth 文档里还专门提到一个很实用的建议：

如果你有“个人账号”和“工作账号”，**最推荐的方式是分 agent 管理**。

比如：

```bash
openclaw agents add work
openclaw agents add personal
```

这么做的好处是：

- 账号隔离
- 凭证隔离
- 工作区隔离
- 不容易把个人和公司的模型路由混在一起

对于长期使用 OpenClaw 的人，这个习惯非常值钱。

---

## 常见误区

### 误区一：买了 ChatGPT Plus，就能直接当 API 用

不行。

ChatGPT 订阅和 OpenAI API 是两套不同的接入方式。想用 `openai/...`，还是需要 **OpenAI API Key**。

### 误区二：用了 ChatGPT OAuth，就应该选 `openai/...`

也不对。

ChatGPT OAuth 在 OpenClaw 官方文档里对应的是 **`openai-codex` provider**。

### 误区三：我只要登录一次，以后所有 agent 都自动共用

不一定。

OpenClaw 的认证信息是 **按 agent 存储** 的。你如果做了多 agent 隔离，就要按各自配置来理解，不要想当然地以为全局共用。

---

## 最后一句话总结

如果一句话把这篇文章说完，那就是：

> **在 OpenClaw 里，用 GPT 订阅并不是“不配置就能直接用”，而是要选对接入路径：API Key 走 `openai`，ChatGPT 订阅 OAuth 走 `openai-codex`。**

只要你把这件事想明白，OpenClaw 接 OpenAI 就不会再绕。

后面无论你是做个人助手、频道机器人、还是多 agent 工作流，都会顺很多。
