---
title: 改了 base_url 之后 Codex 就报 401？原因、排查思路和修复方案一次讲清
description: 很多 Codex 401 并不是“账号失效”，而是你把 base_url 改掉以后，请求发到了不接受当前认证方式的端点。本文讲清原因和修复方法。
pubDate: "2026-03-19T15:27:00Z"
updatedDate: "2026-03-19T15:27:00Z"
featured: false
tags:
  - Codex
  - OpenAI
  - base_url
  - "401"
  - Debugging
---

有一类 Codex 报错非常迷惑：

你明明刚刚还能用，改了一下 `base_url`，下一秒就开始报：

```text
401 Unauthorized
```

很多人第一反应是：

- 是不是账号掉登录了？
- 是不是 API key 过期了？
- 是不是 OpenAI 挂了？

但现实里，**这类 401 很多时候不是“凭证突然坏了”，而是你把请求发到了“不接受当前认证方式”的地址。**

这篇就专门讲这个问题：

1. 为什么改 `base_url` 会导致 Codex 401
2. 常见错误配置长什么样
3. 应该怎么修
4. 以后怎么避免再踩一次

---

## 先说结论

如果一句话总结：

> **`base_url` 决定了 Codex 把请求发给谁；认证方式决定了它带什么凭证。只要“请求目的地”和“凭证类型”不匹配，401 就会出现。**

换句话说，问题的本质不是“你改了一个字符串”，而是：

- 你改了请求发往的服务端
- 但认证方式、请求路径、Header、Key 类型，还是按原来那套在跑
- 于是服务端直接回你 `401 Unauthorized`

---

## 官方文档里其实已经埋了答案

OpenAI 官方 Codex 文档里有几个关键点：

### 1）Codex 支持两种 OpenAI 认证方式

官方认证文档明确写了，Codex 在使用 OpenAI 模型时支持两种登录方式：

- **Sign in with ChatGPT**：订阅式登录
- **Sign in with an API key**：按 API 用量计费

这意味着：

**你当前到底是“ChatGPT 登录”，还是“API key 登录”，非常重要。**

因为这直接决定了 Codex 会带什么凭证出去。

### 2）Codex 允许你改 provider 的 `base_url`

官方高级配置文档也明确写了：

- 你可以为 provider 配 `base_url`
- 如果只是想把内置 OpenAI provider 指向一个代理、路由器，或者数据驻留域名，可以直接设置 `openai_base_url`
- 你也可以在 `[model_providers.<id>]` 里定义自己的 provider

这说明一件事：

**改 `base_url` 本身没问题。**

真正的问题是：

> 你改完之后，新的那个地址，到底认不认你现在带过去的凭证？

---

## 为什么一改 `base_url` 就 401

最常见的原因，其实就三类。

## 原因一：你还在用 ChatGPT 登录，但目标地址已经不是原来那套认证体系

这是最常见的坑。

比如你原来用的是 Codex 默认的 OpenAI / ChatGPT 登录流程，一切正常。后来你把：

```toml
openai_base_url = "https://some-proxy.example.com/v1"
```

或者：

```toml
[model_providers.proxy]
base_url = "https://some-proxy.example.com/v1"
```

改成了一个代理、网关、OpenAI-compatible 路由器，甚至自建服务。

这时候问题就来了：

- Codex 还是按“OpenAI 登录态”带 token
- 但你新的 `base_url` 指向的服务，**未必接受这个 token**
- 它可能只认它自己的 API key
- 也可能要求额外 header
- 也可能根本不支持这套 ChatGPT OAuth / 会话认证

结果就是：**401。**

这类问题看起来像“Codex 登录失效”，本质其实是：

> 你把请求送到了另一个门口，但手里拿的还是旧门禁卡。

---

## 原因二：你把 provider 换成了自定义的，但认证方式没跟着换

官方文档里提到，自定义 provider 时有两种常见认证思路：

### 方案 A：让它继续走 OpenAI 认证

```toml
[model_providers.proxy]
name = "OpenAI Proxy"
base_url = "https://proxy.example.com/v1"
requires_openai_auth = true
```

这里的关键是 `requires_openai_auth = true`。

官方文档明确说了：当你这样配置时，Codex 会继续使用 OpenAI 的认证方式；而且此时会忽略 `env_key`。

### 方案 B：让它走这个 provider 自己的 key

```toml
[model_providers.proxy]
name = "My Router"
base_url = "https://proxy.example.com/v1"
env_key = "MY_PROXY_API_KEY"
```

这时候就不是 OpenAI 登录了，而是让 Codex 去读你自己的环境变量密钥。

问题往往出在：

- 你明明想走 OpenAI 认证，却没写 `requires_openai_auth = true`
- 或者你明明换成了第三方路由器，却还在期待 ChatGPT 登录态能直接通

这两种都会让服务端收到“它不认识”的认证信息，最后返回 401。

---

## 原因三：`base_url` 写得像对，其实路径不对

还有一种情况，也很常见：

**域名对了，路径错了。**

比如：

- 少了 `/v1`
- Azure OpenAI 路径没按要求写
- 代理要求的是某个特定前缀
- 你填的是网页地址，不是 API 地址

这类错误有时会报 404、405，但也有一些网关会统一回 401 或者泛化成鉴权失败。

所以别看到 401 就只盯着 token，**URL 本身也要检查。**

---

## 一个特别容易踩的隐蔽坑：项目级 `.codex/config.toml` 覆盖了你的全局配置

官方文档还提到一件很关键的事：

Codex 不只会读 `~/.codex/config.toml`，它还会读项目里的：

```text
.codex/config.toml
```

而且是**按项目层级逐层覆盖**，越靠近当前工作目录的配置优先级越高。

这意味着你可能以为：

> 我明明已经把全局配置改回来了，为什么还是 401？

实际上很可能是因为：

- 项目目录里还有一份 `.codex/config.toml`
- 里面继续覆盖了 `base_url`
- 于是你以为你在用 A，实际上 Codex 还在用 B

这个坑非常常见，而且非常阴。

---

## 怎么确认到底是不是 `base_url` 导致的

如果你怀疑是这个问题，最实用的排查顺序是下面这样。

## 第一步：先确认你现在用的是哪种认证方式

官方文档里已经说明了 Codex 常见的两种认证：

- ChatGPT 登录
- API key 登录

你先把这个问题搞清楚：

- 你现在到底是 `codex login` 过来的 ChatGPT 登录？
- 还是本来就配置了 API key？

这一步不清楚，后面就全是盲查。

---

## 第二步：检查到底是谁在覆盖 `base_url`

你至少要同时看这两处：

```text
~/.codex/config.toml
.codex/config.toml
```

重点查这些字段：

- `openai_base_url`
- `model_provider`
- `[model_providers.xxx]`
- `base_url`
- `requires_openai_auth`
- `env_key`

如果你改了全局，但项目里还有覆盖，最终生效的可能根本不是你看到的那份。

---

## 第三步：确认目标服务到底认哪种认证

这是排障里最关键的一步。

你要确认你现在指向的那个 `base_url`：

- 是官方 OpenAI 接口？
- 是 OpenAI 数据驻留域名？
- 是公司内部代理？
- 是第三方 OpenAI-compatible router？
- 是 Azure OpenAI？

然后问自己：

> 这个服务，到底认什么凭证？

如果它只认自己的 API key，而你还在给它发 ChatGPT 登录态，那 401 完全正常。

如果它支持 OpenAI 认证转发，那你就应该按官方方式配置 `requires_openai_auth = true`，而不是乱混。

---

## 修复方案一：你其实就是想用官方 OpenAI / 官方 Codex

如果你根本不需要代理，不需要 router，不需要自定义 endpoint，那么最简单的修法就是：

### 直接把 `base_url` 改回默认

比如把这类配置删掉：

```toml
openai_base_url = "https://some-proxy.example.com/v1"
```

或者删掉你临时试验的自定义 provider。

如果你之前已经把配置改乱了，最干脆的做法是：

1. 去掉自定义 `base_url`
2. 保留官方默认 provider
3. 重新登录一次

例如重新走：

```bash
codex login
```

如果你怀疑本地缓存已经脏了，再重新登录一次通常更稳。

---

## 修复方案二：你要接代理，但这个代理支持 OpenAI 认证

如果你的目标是：

- 仍然使用 OpenAI 模型
- 但通过公司代理、LLM 网关、数据驻留地址转发

那就应该按官方文档的思路来配。

### 方式 A：用 `openai_base_url`

如果你只是想把**内置 OpenAI provider** 指到另一个 OpenAI 兼容入口，官方建议直接用：

```toml
openai_base_url = "https://us.api.openai.com/v1"
```

这个思路适合“仍然是 OpenAI 体系，只是 endpoint 变了”的情况。

### 方式 B：自定义 provider + `requires_openai_auth = true`

如果你是通过代理访问 OpenAI 模型，官方也给了这种写法：

```toml
model_provider = "proxy"

[model_providers.proxy]
name = "OpenAI Proxy"
base_url = "https://proxy.example.com/v1"
requires_openai_auth = true
```

这套配置的重点是：

- 代理必须真的支持 OpenAI 认证透传
- 你不能期待“任何 OpenAI-compatible 服务”都支持 ChatGPT / OpenAI 登录态

如果代理本身不支持，那你照抄这段也还是会 401。

---

## 修复方案三：你要接第三方 router / 网关，那就老老实实用它自己的 key

如果你的 `base_url` 指向的是：

- 第三方 API router
- 公司内部网关
- OpenAI-compatible 中转服务
- 自己的模型代理层

那很多时候正确修法不是“继续折腾 ChatGPT 登录”，而是：

### 换成 provider 自己的 key

```toml
model_provider = "router"

[model_providers.router]
name = "My Router"
base_url = "https://router.example.com/v1"
env_key = "ROUTER_API_KEY"
```

然后在环境变量里提供对应密钥。

这套思路最稳，因为认证边界是清楚的：

- 你请求谁
- 就用谁的 key

不要指望所有兼容 OpenAI 接口格式的服务，也兼容 OpenAI 的身份体系。

它们是两回事。

---

## 修复方案四：检查是不是项目配置在偷改你

如果你明明已经“改回来了”，结果问题还在，那就优先检查：

```text
项目目录/.codex/config.toml
```

把里面的 `base_url` / `model_provider` / `openai_base_url` / provider 配置翻一遍。

很多“怎么改都不生效”的问题，最后都不是 Codex 有鬼，而是：

> 你改的是全局配置，但项目配置把它又盖回去了。

---

## 一个实用的判断口诀

以后你只要遇到“改完 `base_url` 就 401”，脑子里直接过这四句：

1. **我现在带出去的是 ChatGPT 登录，还是 API key？**
2. **我现在请求发给的是谁？**
3. **这个服务认不认这种凭证？**
4. **有没有项目级 `.codex/config.toml` 在覆盖我？**

这四个问题答清楚，基本都能定位到根因。

---

## 最后一句话总结

真正的结论其实不复杂：

> **`base_url` 改掉以后，401 往往不是“Codex 突然坏了”，而是“认证方式和目标端点不再匹配了”。**

要修，就不要只盯着“重新登录”。

你应该优先确认：

- 是不是还在用对的 provider
- 是不是还在用对的认证方式
- 代理/网关是不是真的支持 OpenAI auth
- 有没有项目级配置把你的修改覆盖掉

把这几个点理顺，`401 Unauthorized` 这种问题通常都能很快收掉。
