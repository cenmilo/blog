---
url: /posts/370d9c90ee50c60cfa77bd63940e96ca/
title: 532 - 配置里的环境变量覆盖：NUXT_ 前缀与命名映射
date: 2027-04-07T10:00:00+08:00
updated: 2027-04-07T10:00:00+08:00
author: cmdragon

summary:
  摘要：`runtimeConfig` 的值会被"匹配的环境变量"在运行时自动替换，而匹配要同时满足两条：**自定义键必须先在 `nuxt.config` 里定义过**（官方给的理由是防止任意环境变量被暴露给应用代码），**变量必须是大写、以 `NUXT_` 开头、用 `_` 分隔键与大小写变化**——`apiSecret` 对 `NUXT_API_SECRET`、`public.apiBase` 对 `NUXT_PUBLIC_API_BASE`。有一处官方明说的例外：`NUXT_APP_SECRET` **不需要在配置里声明键**，因为 `runtimeConfig.appSecret` 是 Nuxt 自带的（默认空字符串、仅服务端可见）。另外 `nitro.envPrefix` 的默认值就是 `"NUXT_"`，`app` 这个命名空间被官方保留、叫你别拿它放自己的值。值本身还会被 destr 转型，纯数字形状的密钥要包引号。

categories:
  - 前端开发

tags:
  - nuxt4
  - 环境变量
  - runtime-config
  - 命名映射
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

在平台上设了 `API_BASE=https://api.example.com`，代码里 `useRuntimeConfig().public.apiBase` 还是配置里那个默认值。

两个原因都可能：名字不对，或者那个键压根没在配置里声明过。这一篇讲这两条规则。

## 两条硬要求

Runtime Config 那一节先给定性：**"Runtime config values are automatically replaced by matching environment variables at runtime."** 然后是两个要求，原文：

1. Custom runtime config variables must be defined in your `nuxt.config`. **This ensures that arbitrary environment variables are not exposed to your application code.**
2. Only a specially-named environment variable can override a runtime config property. That is, **an uppercase environment variable starting with `NUXT_` which uses `_` to separate keys and case changes.**

第 1 条的理由值得记住——这不是繁琐，是**边界**：没声明的键不会因为环境里有同名变量就渗进应用。所以"设了没生效"的第一嫌疑是配置里没这个键，第二才是变量名。

## 命名映射：官方给的那对示例

映射规则本身只有第 2 条那句话，具体对应靠示例。官方给的 `.env` 与配置成对出现：

```ini [.env]
NUXT_API_SECRET=api_secret_token
NUXT_PUBLIC_API_BASE=https://nuxtjs.org
```

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '', // can be overridden by NUXT_API_SECRET environment variable
    public: {
      apiBase: '', // can be overridden by NUXT_PUBLIC_API_BASE environment variable
    },
  },
})
```

从这两段能读出的映射形状：

| 配置里的键 | 环境变量 | 规律 |
| --- | --- | --- |
| `apiSecret` | `NUXT_API_SECRET` | camelCase 的两个词在变量里变成 `API_SECRET`，前面加 `NUXT_` |
| `public.apiBase` | `NUXT_PUBLIC_API_BASE` | **嵌套段各占一段**：`public` → `PUBLIC`、`apiBase` → `API_BASE` |

**注意文档没有给出完整算法**：连续大写缩写（比如 `apiURL`）、含数字的键名会被映射成什么，这一节没写。撞上时别猜，用下面那条读实际值的方法验证。

## 例外：NUXT_APP_SECRET 不用声明

`app` 命名空间里有个官方自带的例外。Application Secret 那节先划保留：**"Runtime config keys prefixed with `app` are reserved for Nuxt (currently `runtimeConfig.app` and `runtimeConfig.appSecret`). Do not use them for your own values."**

紧接着给的是那句关键描述：**"The value defaults to an empty string, is only available on the server, and you can set it with `NUXT_APP_SECRET` without declaring the key in `nuxt.config`."**

三个信息：**默认空字符串**、**只有服务端能读到**、**不需要你在配置里声明键**（因为它本来就不是"自定义"键）。这一条与上面第 1 条不冲突——第 1 条管的是自定义键。

同一个作用域里还有两个官方点名的可覆盖项，都在 `runtimeConfig.app` 下：`app.baseURL` 与 `app.cdnURL`，对应 `NUXT_APP_BASE_URL` 与 `NUXT_APP_CDN_URL`，两者都写在启动命令前（属于运行时注入）。

**所以命名习惯上有一处陷阱**：`NUXT_APP_*` 这一段是保留给 Nuxt 内部值的。想给自己的配置加环境变量，别起名成 `runtimeConfig.app.xxx`，否则按 `NUXT_B5003` 那一节的说法，你的键会与框架自己的值撞在同一命名空间里。

## 前缀本身可以改，但默认就是 NUXT_

`nitro.envPrefix` 这个默认值出现在配置参考里 `runtimeConfig` 的默认 JSON 中：

```json
{
  "public": {},
  "app": { "buildId": "...", "baseURL": "/", "buildAssetsDir": "/_nuxt/", "cdnURL": "" },
  "nitro": { "envPrefix": "NUXT_" }
}
```

也就是说 `NUXT_` 这个前缀是 Nitro 侧的一个配置项，默认值就是它。**这一节没有说明改这个前缀是否同步影响 `runtimeConfig.app.*` 那几个官方变量**，所以真要改前缀，先按下一节的方法把两边的实际行为测出来。

## 值会被转型，密钥最容易出事

两个地方写了同一类风险。Runtime Config 那一节的 `::warning`：环境变量值会用 destr 自动转成对应的 JS 类型，**`NUXT_MY_VAR=4848e0` 会变成数字 `4848`**；要保持字符串，值里必须真的含双引号——`.env` 里写 `NUXT_MY_VAR='"4848e0"'`，直接设变量时（shell、Dockerfile、平台面板）要确认引号是值的一部分、没被 shell 剥掉。

Application Secret 那节把同一件事讲得更具体：**"Nitro parses environment overrides, so a secret made only of digits or one that looks like JSON arrives as a number, boolean or object rather than a string."** 官方给的两条应对是包引号（`NUXT_APP_SECRET='"12345..."'`），或者干脆把 `runtimeConfig.appSecret` 写在 `nuxt.config` 里。

还有一条只在构建期生效的坑：**把默认值指向"不同名的环境变量"会在运行时坏掉**——原文举的是设 `myVar` 为 `process.env.OTHER_VARIABLE`，"will only work during build-time and will break on runtime"，官方的建议是让变量名与 `runtimeConfig` 的结构相匹配。

## 密钥没设会怎样

这段与部署直接相关：开发环境下如果 `appSecret` 没设，**Nuxt 会生成一个随机密钥存在 `.nuxt/app-secret`**，并且第一次用到派生密钥时打一条警告；但 **"Builds never generate one: set `NUXT_APP_SECRET` in every environment you deploy to."**

配套的两句限制：**换了这个值会作废所有依赖它的会话或签名**（官方要求每个环境不同、但在同一环境的多次部署与多实例之间保持稳定）；以及 **Nuxt 本身不要求 `NUXT_APP_SECRET`，只有用到它的模块或服务端特性才需要**。

## 自查

- 覆盖要同时满足两条：自定义键先在 `nuxt.config` 里声明（这是防止任意环境变量渗入的边界），变量名是大写、`NUXT_` 开头、用 `_` 分隔键与大小写变化。
- 映射形状靠示例记：`apiSecret` ⇔ `NUXT_API_SECRET`、`public.apiBase` ⇔ `NUXT_PUBLIC_API_BASE`；**连续大写与数字键的算法文档未给**。
- `NUXT_APP_SECRET` 是官方明说的例外（不用声明键），`app` 命名空间被保留、别放自己的值；`app.baseURL`/`app.cdnURL` 也走启动时的 `NUXT_APP_*`。
- `NUXT_` 来自 `nitro.envPrefix` 默认值；改前缀对官方那几个变量的影响文档未说明。
- 值会被 destr 转型：纯数字或像 JSON 的密钥要包引号，或改在配置里写；指向"不同名变量"的默认值只在构建期有效。
- 构建从不自动生成 appSecret——每个部署环境都要自己设，且它的值一旦改动会作废相关会话与签名。

## 下一篇

下一篇 533 讲那批"已经不存在"的配置文件：`nitro.config`、`vite.config`、`postcss.config`、`webpack.config` 都不再被单独读取，四条映射与 b5004 那条迁移提示怎么配合。

## 本文依据

- https://nuxt.com/docs/4.x/guide/going-further/runtime-config（运行时被匹配变量替换的定性句、两条 key requirements 与理由、`NUXT_API_SECRET`/`NUXT_PUBLIC_API_BASE` 成对示例、destr 转型的 `::warning` 与 `'"4848e0"'`、不同名变量只在构建期有效的 `::warning`、Application Secret 小节：`app` 前缀保留句、默认空字符串/仅服务端/`NUXT_APP_SECRET` 无需声明键、32 随机字节与跨部署稳定与改值作废会话签名、纯数字或 JSON 形状密钥的两种应对、dev 生成到 `.nuxt/app-secret` 而 builds 从不生成、Nuxt 本身不要求该变量）
- https://nuxt.com/docs/4.x/api/nuxt-config（`runtimeConfig` 默认 JSON 含 `nitro.envPrefix: "NUXT_"` 与 `app` 四键；`### baseURL` 与 `### cdnURL` 的 `NUXT_APP_BASE_URL`/`NUXT_APP_CDN_URL` 启动注入写法）
- https://nuxt.com/docs/4.x/errors/b5003（`runtimeConfig.app` 下自定义键会与框架内部值相撞，去处是 `public` 或顶层）
- https://nuxt.com/docs/4.x/guide/going-further/server-imports（`deriveSecret` 与 session helpers 默认用应用密钥签署，用于说明谁在依赖 `appSecret`）
