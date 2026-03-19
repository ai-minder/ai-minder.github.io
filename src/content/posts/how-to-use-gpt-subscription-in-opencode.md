---
title: 如何在 OpenCode 上使用 GPT 订阅：别把 ChatGPT 订阅和 API Key 搞混
description: 想在 OpenCode 里直接用自己现成的 GPT 订阅？这篇文章讲清楚可行路径、安装方式、登录流程、适用范围和常见误区。
pubDate: 2026-03-19
updatedDate: 2026-03-19
featured: false
tags:
  - OpenCode
  - OpenAI
  - ChatGPT
  - GPT
  - Codex
---

很多人第一次装 OpenCode，都会问同一个问题：

> 我已经买了 ChatGPT Plus / Pro / Business，能不能直接在 OpenCode 里用？

答案是：**能走一条“订阅授权”路线，但你得先把概念分清楚。**

最容易混淆的地方是这两个东西：

- **ChatGPT 订阅**
- **OpenAI API Key**

它们不是一回事。

如果你在 OpenCode 里想复用自己已有的 GPT 订阅，通常走的不是“填 OpenAI API key”这条线，而是 **ChatGPT OAuth / Codex 授权** 这条线。

这篇文章就把这事一次讲清楚。

## 先说结论

如果你只想快速知道该怎么搞，直接看这几条：

- 想在 OpenCode 里用现成的 GPT 订阅，不是去填普通 OpenAI API key
- 一条常见做法是安装 **OpenAI Codex OAuth 插件**
- 然后执行：

```bash
npx -y opencode-openai-codex-auth@latest
opencode auth login
```

- 完成浏览器登录授权后，再在 OpenCode 里选对应的 OpenAI / GPT 模型

更直白点说：

> **订阅用户常走的是 OAuth 登录路线，不是 API Key 路线。**

## 为什么很多人一上来就配错

因为大多数人脑子里只有一种“接模型”的方式：

- base URL
- API key
- model

这套对官方 OpenAI API 没问题，但对“我已经买了 ChatGPT 订阅，想直接复用账号授权”这件事，就不完全对了。

OpenCode 官方 Providers 文档的核心思路是：

- 普通 provider 主要通过 `/connect` 去添加 API key
- 某些 provider 也支持通过 OAuth 或订阅授权方式接入

而现在想在 OpenCode 里复用 OpenAI / GPT 订阅，业界常见路径就是：

- **OpenCode + OpenAI Codex OAuth 插件**
- 浏览器登录你的 ChatGPT 账号
- 让 OpenCode 通过授权态去使用对应模型

所以第一层认知一定要立住：

> **“我有 ChatGPT 订阅” ≠ “我已经有 OpenAI API Key”**

## OpenCode 官方是怎么设计 provider 的

从 OpenCode 官方 Providers 文档看，它本身是一个 **多 provider、模型无关** 的 AI coding agent。

OpenCode 支持：

- 通过 `/connect` 添加 provider 凭据
- 把凭据存到本地认证文件里
- 再通过 `/models` 选择模型

也就是说，OpenCode 本身并不死绑某一家。它更像一个统一壳子，底下接不同 provider。

对于普通 OpenAI API 用户，思路通常是：

- 去 OpenAI 平台拿 API key
- 在 OpenCode 里添加 provider key
- 选模型开用

但如果你想吃的是 **ChatGPT 订阅本身的授权**，那通常就要用额外的 OAuth 插件来打通。

## 目前最常见的做法：装 OpenAI Codex OAuth 插件

现在比较常见的一条路，是这个插件：

- `numman-ali/opencode-openai-codex-auth`

它的定位写得很直白：

- 给 OpenCode 提供 **ChatGPT Plus / Pro** 的 OAuth 认证接入
- 安装后可以通过 `opencode auth login` 走网页登录授权
- 再在 OpenCode 里使用对应的 GPT / Codex 模型

插件 README 给出的最短路径基本就是：

```bash
npx -y opencode-openai-codex-auth@latest
opencode auth login
```

然后你就会被拉去浏览器登录 OpenAI / ChatGPT 账号。

授权成功后，再执行类似：

```bash
opencode run "write hello world to test.txt" --model=openai/gpt-5.2 --variant=medium
```

注意，具体模型名会随着插件版本和 OpenCode 版本变化，README 里列出的常见族包括：

- `gpt-5.2`
- `gpt-5.2-codex`
- `gpt-5.1-codex`
- `gpt-5.1-codex-mini`

所以最稳的做法不是死背模型名，而是：

- 先装插件
- 先完成登录
- 再看当前插件实际支持的模型和 variant

## 实际登录流程是怎样的

如果你以前用过 OAuth 工具，这个流程其实不难。

大概就是：

1. 安装 OpenCode 的 OpenAI Codex 认证插件
2. 执行 `opencode auth login`
3. 终端拉起浏览器
4. 你登录自己的 ChatGPT / OpenAI 账号
5. 授权完成后，OpenCode 拿到对应登录态
6. 后续在 OpenCode 里选模型使用

对普通用户来说，你不需要关心太多底层细节。真正重要的是：

> **你登录的是账号授权，不是手填 API key。**

## Plus / Pro / Business 到底能不能用

这里得说得谨慎一点，别瞎鸡巴承诺。

从目前公开资料看：

- 插件 README 明确主打的是 **ChatGPT Plus / Pro** OAuth 认证
- 插件定位是 **personal development use**
- README 也明确提醒：
  - 适合个人开发使用
  - 如果是生产环境或多用户应用，应该优先走 **OpenAI Platform API**

这意味着什么？

### Plus / Pro

这是目前最明确的支持目标。也就是说，如果你是 Plus / Pro 用户，走这条 OAuth 路线是最合理的理解方式。

### Business

Business 本质上还是 ChatGPT 订阅体系里的一种，但和个人 Plus / Pro 在组织、权限、数据边界上不完全一样。

是否能在你的具体 OpenCode + 插件环境里顺利复用，取决于：

- 你的账号权限状态
- 插件当前支持情况
- OpenCode 当前版本
- OpenAI 那边的授权行为有没有变化

所以比较稳妥的说法是：

> **如果你是个人用户，优先按 Plus / Pro 的 OAuth 路线理解。**
>
> **如果你是 Business 用户，别想当然，先实测当前插件和授权流程是否能正常通过。**

## 什么时候你应该改走 API Key 路线

虽然很多人更想“复用订阅”，但不是所有场景都适合这么干。

如果你要的是这些东西：

- 更稳定、可预期的程序化调用
- 团队多人共享
- 明确的计费与额度边界
- 生产环境使用
- 更清晰的报错和可观测性

那你更应该考虑：

- 直接走 **OpenAI Platform API**
- 在 OpenCode 里按标准 provider 方式配置 API key

也就是说：

- **个人自己爽用** → 订阅 OAuth 路线很香
- **团队 / 生产 / 自动化严肃场景** → API key 路线通常更稳

## 常见误区

## 误区一：我买了 ChatGPT Plus，就等于我有 OpenAI API 能力

不是。

ChatGPT 订阅和 OpenAI API 是两套东西。你在 OpenCode 里能不能用，取决于你走的是哪条接入路径。

## 误区二：OpenCode 里所有 OpenAI 能力都只靠 `/connect` 填 key

也不对。

OpenCode 官方 provider 体系确实大量依赖 `/connect` 和 API key，但要复用 ChatGPT 订阅，常见接法是 **OAuth 插件 + 浏览器登录**。

## 误区三：订阅登录一定比 API key 更适合所有人

扯淡。

订阅登录的好处是省事、贴近“我已有账号就直接用”。

但一旦你要的是：

- 多人共用
- 稳定工作流
- 自动化调用
- 服务端部署

API key 往往还是更干净。

## 误区四：模型名永远不变

也别想太美。

这种插件最容易变的就是：

- 模型名
- variant 名称
- 登录流程
- 兼容的 OpenCode 版本

所以最稳的办法永远是：

- 看当前插件 README
- 看当前 OpenCode 版本说明
- 本地跑一遍最小测试命令

## 一个实用的上手顺序

如果你是第一次折腾，建议按这个顺序来：

### 路线 A：你就是想复用自己的订阅

1. 安装插件

```bash
npx -y opencode-openai-codex-auth@latest
```

2. 登录授权

```bash
opencode auth login
```

3. 看插件 / 当前环境支持哪些模型

4. 先跑一个最小测试：

```bash
opencode run "write hello world to test.txt" --model=openai/gpt-5.2 --variant=medium
```

5. 如果不通，先排查：
   - 插件版本
   - OpenCode 版本
   - 浏览器授权是否成功
   - 模型名是不是过期了

### 路线 B：你更看重稳定和可控

那就别折腾订阅复用了，直接走官方 API key provider。

你会省掉很多奇怪的兼容问题。

## 最后一句话总结

如果一句话把这篇文章说完，那就是：

> **在 OpenCode 上使用 GPT 订阅，核心不是去找 OpenAI API key，而是通过 ChatGPT OAuth / Codex 授权路线把你已有的订阅账号接进来。**

个人开发场景，这条路挺顺。

但如果你要把它当成团队级、生产级、长期稳定的接入方式，那还是建议优先考虑标准 API 路线。别为了省一步配置，后面把自己坑进一堆兼容破事里。

## 参考来源

- OpenCode 官方 Providers 文档
- `numman-ali/opencode-openai-codex-auth` README
