---
title: 如何把 GPT 订阅通过 CLIProxyAPI 转发成 API 使用
description: 想把自己已有的 GPT 订阅接成 OpenAI 兼容 API？这篇文章讲清楚 CLIProxyAPI 的原理、安装、登录、转发配置和常见报错排查。
pubDate: "2026-03-19T15:58:00Z"
updatedDate: "2026-03-19T15:58:00Z"
featured: false
tags:
  - GPT
  - OpenAI
  - CLIProxyAPI
  - API
  - Codex
---

很多人手里已经有 GPT 订阅了，但一到要接程序、接客户端、接工作流的时候，就会卡在同一个问题上：

> ChatGPT / GPT 订阅我已经付费了，能不能把它转成 API 来用？

如果你想要的是一个 **OpenAI 兼容接口**，让现有工具可以像调用普通 API 一样去调 GPT，那么 `CLIProxyAPI` 是目前很常见的一条路。

它的核心思路并不复杂：

- 你先用 **自己的账号** 完成 OAuth 登录
- CLIProxyAPI 持有这份授权状态
- 然后在本地或服务器上暴露一个 **OpenAI 兼容接口**
- 你的应用、脚本、IDE 插件、自动化工具，再去请求这个接口

这样一来，很多原本只支持“填 `base_url + api_key`”的客户端，也能接到你已经授权好的 GPT 能力。

不过这里先说一句最重要的前提：

> 这篇文章讲的是 **基于你自己已授权账号** 的转发接法，不是“凭空把订阅变成官方 OpenAI API Key”。

它更像是：**把 OAuth 登录态包装成一个 API 代理层**。

---

## 先说结论

如果你只想快速知道应该怎么配，可以直接看这几行：

1. 安装 CLIProxyAPI
2. 用 OpenAI Codex / GPT 账号完成登录授权
3. 启动本地代理，例如 `http://127.0.0.1:8317`
4. 让你的客户端把：
   - `base_url` 指到 `http://127.0.0.1:8317/v1`
   - `api_key` 填成 **CLIProxyAPI 自己配置的 key**
   - `model` 填成代理暴露出来的模型名

也就是说，客户端并不是直接请求 OpenAI，而是请求 **CLIProxyAPI 暴露出的 OpenAI 兼容端点**。

---

## 这件事到底解决了什么问题

普通情况下，很多客户端只认这套配置：

- `base_url`
- `api_key`
- `model`

但 GPT 订阅本身通常不是这种接法。

你平时用 ChatGPT、Codex OAuth 登录，拿到的是一种“账号授权态”，而不是一串标准的 OpenAI API key。所以很多工具虽然支持 OpenAI 协议，却没法直接吃你的订阅登录态。

CLIProxyAPI 解决的就是这个缝：

- 上游：用 OAuth 登录 GPT / Codex
- 下游：吐出 OpenAI 兼容 API

所以它特别适合这些场景：

- 想把 GPT 订阅接给本地脚本
- 想给 OpenClaw、Continue、Cline、Cursor 一类工具走统一入口
- 想把多个账号做轮询或切换
- 想把模型调用统一挂到自己的代理层上

---

## 你要先理解一件事：这不是官方 API Key

这里很容易混淆。

很多人会把下面几样东西混成一个概念：

- ChatGPT / GPT 订阅
- OpenAI API Key
- Codex OAuth 登录
- OpenAI 兼容接口

它们其实不是一回事。

### 官方 API Key 是什么

这是 OpenAI 原生的开发者调用方式。你直接拿官方 API key，请求官方 API endpoint。

### CLIProxyAPI 做的是什么

CLIProxyAPI 并不是把你的订阅“转换成官方 key”。它做的是：

- 用你已有的 OAuth / CLI 登录态去访问上游能力
- 再在你自己的机器或服务器上，包装出一个 OpenAI-compatible API

所以正确理解应该是：

> **你得到的不是官方 API key，而是一个由 CLIProxyAPI 暴露出来的兼容层。**

这点想明白，后面很多配置就不会绕。

---

## 准备条件

开始之前，通常需要这些东西：

### 1）你自己的 GPT / OpenAI 账号授权

你至少要有一个可以正常完成 OAuth 登录的账号，并确保相关能力在你账号上可用。

### 2）一台运行 CLIProxyAPI 的机器

可以是：

- 本机
- 家里服务器
- 云服务器
- Docker 容器

如果只是自己本地用，**先从本机启动** 是最简单的。

### 3）你要接入的客户端

比如：

- OpenClaw
- Continue
- Cline
- Cursor 的 OpenAI-compatible 模式
- 你自己的脚本或后端

---

## 第一步：安装 CLIProxyAPI

最常见的是直接按项目文档安装，或者用 Docker 跑。

如果你是 Linux / macOS，本地快速起步通常会比较方便。

项目 README 给出的思路里，最关键的不是“装在哪”，而是两个点：

1. 你要有一个 `config.yaml`
2. 你要有一个存授权状态的目录，比如：

```yaml
port: 8317
auth-dir: "~/.cli-proxy-api"
debug: true
api-keys:
  - "sk-cliproxyapi-default-key-change-me"
```

这个配置的意思很直白：

- 代理监听在 `8317`
- 登录 token 存在 `~/.cli-proxy-api`
- 开启调试日志
- 对下游客户端要求一个代理自己的 API key

这里要特别注意：

> `api-keys` 这里配置的是 **CLIProxyAPI 自己的访问密钥**，不是 OpenAI 官方 key。

你后面给客户端填的 `api_key`，通常就是它。

---

## 第二步：完成 OpenAI Codex / GPT 登录

CLIProxyAPI 的核心能力之一，就是支持通过 OAuth 登录上游账号。

对 GPT 这条线来说，常见做法是走 **OpenAI Codex 登录**。

通常流程是：

1. 启动 CLIProxyAPI 的登录流程
2. 浏览器打开授权页面
3. 你登录自己的 OpenAI / GPT 账号
4. CLIProxyAPI 把授权结果保存到 `auth-dir`

很多人第一次会卡在这里，原因通常不是工具坏了，而是：

- 浏览器没成功拉起
- 回调没接住
- 本地端口被占了
- 登录态过期了

如果登录成功，后面你就不用再把“订阅”直接塞给客户端了，因为真正和上游打交道的是 CLIProxyAPI。

---

## 第三步：启动代理服务

登录完成后，就可以正式启动服务。

启动之后，你通常会得到一个本地地址，例如：

```text
http://127.0.0.1:8317
```

这时候它对下游暴露的是 OpenAI-compatible API，所以大多数支持自定义 `base_url` 的客户端，都可以直接接入。

一个常见习惯是把它写成：

```text
http://127.0.0.1:8317/v1
```

因为很多工具默认就是按 OpenAI 的 `/v1/...` 路径来拼接口。

---

## 第四步：先用 curl 验证通不通

在接 IDE、接工作流之前，最稳的做法永远是先手测一遍。

如果你的代理支持 OpenAI Responses 接口，可以先这样测：

```bash
curl http://127.0.0.1:8317/v1/responses \
  -H "Authorization: Bearer sk-cliproxyapi-default-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": "用一句话解释 CLIProxyAPI 是做什么的。"
  }'
```

如果你接的是只会调 Chat Completions 的旧客户端，也可以用：

```bash
curl http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer sk-cliproxyapi-default-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "messages": [
      {"role": "user", "content": "你好，做个自我介绍。"}
    ]
  }'
```

如果这一层能通，后面接任何客户端基本都只是“填参数”的问题。

---

## 第五步：在客户端里怎么填

几乎所有 OpenAI-compatible 客户端，最终都逃不过三项：

### 1）Base URL

填成你的代理地址，例如：

```text
http://127.0.0.1:8317/v1
```

如果你是部署在云端，就换成你自己的域名：

```text
https://llm.example.com/v1
```

### 2）API Key

填成 **CLIProxyAPI 自己的 key**，例如：

```text
sk-cliproxyapi-default-key-change-me
```

不是 OpenAI 官方 key，也不是你浏览器里的 token。

### 3）Model

填成 CLIProxyAPI 当前支持或映射出来的模型名。

很多示例会直接用：

```text
gpt-5
```

但实际以你当前代理配置、模型别名和上游支持情况为准。如果某个客户端报“模型不存在”，优先去看代理当前暴露的模型映射，而不是怀疑自己 key 错了。

---

## 在 OpenAI SDK 里怎么接

如果你自己写代码，思路其实最简单。

以 Node.js 为例，核心就是把 `baseURL` 指向 CLIProxyAPI：

```js
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: 'sk-cliproxyapi-default-key-change-me',
  baseURL: 'http://127.0.0.1:8317/v1'
});

const resp = await client.responses.create({
  model: 'gpt-5',
  input: '给我一句话解释什么是 CLIProxyAPI。'
});

console.log(resp.output_text);
```

如果你的客户端库比较旧，也可以走 `chat.completions.create` 那一套，本质一样。

---

## 如果你想部署到远程服务器

很多人本地调通之后，下一步就是想把它挂到 VPS 或家里的服务器上。

这当然可以，但有几个点一定要提前想明白。

## 1）管理端和代理端不要裸奔

如果你把服务直接暴露到公网，至少要考虑：

- 代理访问 key
- 反向代理
- HTTPS
- 访问控制
- 日志里是否会泄露敏感信息

特别是管理接口，最好只开放给内网或 localhost，不要为了图方便全网裸开。

## 2）代理 key 要单独管理

不要把默认 key 原样公开用。

最少也该做这些事：

- 换成随机强密码
- 放到环境变量或安全配置里
- 不要硬编码进公开仓库

## 3）你要接受这不是官方托管 API

这意味着：

- 稳定性取决于你的代理环境
- 并发和限流取决于你的账号状态、网络和代理实现
- 掉登录、token 过期、上游变更，都可能影响可用性

所以它非常适合个人、团队内部、实验环境、自建工作流；但如果你要做对外正式商用接口，架构和风控就要想得更重一些。

---

## 最常见的几个坑

这种接法最容易出问题的地方，其实不是“不会安装”，而是概念混淆。

## 坑一：把 OpenAI key 和 CLIProxyAPI key 填反

这是最常见的。

很多人会在客户端里填：

- `base_url = http://127.0.0.1:8317/v1`
- `api_key = sk-openai-xxxx`

然后发现 401。

因为这时候校验你请求的，不是 OpenAI，而是 **CLIProxyAPI 本身**。你当然要填它自己的 key。

---

## 坑二：地址对了，但少了 `/v1`

不少客户端会自己拼路径，但也有不少不会。

你如果只填：

```text
http://127.0.0.1:8317
```

有的客户端最终请求的可能就不是你预期的 API 路径。

最稳的做法通常还是直接填：

```text
http://127.0.0.1:8317/v1
```

---

## 坑三：上游登录失效了，你却一直怀疑客户端

如果 CLIProxyAPI 持有的 OAuth 状态过期，或者上游授权已经掉了，下游客户端看到的往往只是：

- 401
- unauthorized
- invalid token
- upstream auth failed

这时候别一直在 IDE 里改 `api_key`，而应该先确认：

- CLIProxyAPI 当前登录状态是不是还有效
- token 有没有过期
- 最近有没有改过代理配置或账号

---

## 坑四：把它当作“官方 API 平替”去理解

它当然能让很多 OpenAI-compatible 工具跑起来，但你最好别把它理解成“拿订阅等于拿到了官方 API 账户能力”。

这两者在本质上不是一个东西：

- 认证边界不同
- 额度来源不同
- 稳定性边界不同
- 风险和维护方式也不同

正确心智应该是：

> **CLIProxyAPI 是一个兼容层，不是官方 API key 发行器。**

---

## 什么时候适合这种方案

如果你符合下面这些场景，这条路就很顺：

- 你已经有 GPT / Codex 订阅
- 你主要想接现有客户端和自动化工具
- 你接受“自己维护一个代理层”
- 你想统一多个工具的接入方式

但如果你要的是：

- 官方 SLA
- 标准开发者计费
- 非常清晰的企业采购和法务边界
- 对外提供正式商业接口

那你还是应该优先考虑官方 API 方案。

---

## 最后一句话总结

如果一句话说完这篇文章，那就是：

> **CLIProxyAPI 的本质，是把你自己的 GPT / Codex OAuth 登录态，包装成一个 OpenAI 兼容 API，让现有客户端可以继续按 `base_url + api_key + model` 的方式接入。**

它不是把订阅变成官方 API key，而是给“只会说 API 语言的客户端”加了一层翻译器。

只要你把这点想清楚，配置时就不会再把 key、base_url、上游登录态这几件事搅在一起。
