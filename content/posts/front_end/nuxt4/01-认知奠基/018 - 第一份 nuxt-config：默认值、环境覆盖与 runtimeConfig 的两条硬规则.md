---
url: /posts/85b8a88c532f2e4f783d4ce24816e0a0/
title: 第一份 nuxt.config：默认值、环境覆盖与 runtimeConfig 的两条硬规则
date: 2026-10-18T09:00:00+08:00
updated: 2026-10-18T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：nuxt.config.ts 位于项目根目录，用来覆盖或扩展应用行为，默认配置已能覆盖大多数场景；最小写法是导出 defineNuxtConfig 包一个对象，而这个 helper 全局可用不需要 import。配置本身还能按环境写：$production、$development、$env.staging，用 --envName 选择（例如 nuxt build --envName staging）。runtimeConfig 侧有两条必须记住的规则：一、自定义的 runtime config 变量必须在 nuxt.config 里声明，官方说这是为了确保任意环境变量不会被暴露给应用代码；二、只有特定命名的环境变量能覆盖它，也就是以 NUXT_ 开头的大写变量，用下划线分隔键与大小写变化。另外三条官方明写的坑：把默认值设成另一个名字的变量（如 myVar: process.env.OTHER_VARIABLE）只在构建期有效、运行时会坏；环境变量值会被 destr 自动转成 JS 类型（NUXT_MY_VAR=4848e0 会变成数字），要保字符串必须让值本身带双引号；Nuxt CLI 在 dev/build/generate 时读 .env，但运行已构建的服务端时不会读你的 .env。

categories:
  - 前端开发

tags:
  - nuxt4
  - 配置
  - 环境变量
  - runtimeconfig
  - 部署

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


005 讲划了"约定 vs 配置"的界，这一讲真动手。重点不在语法，而在**一份配置在三条链上取值不同**：本地 dev、构建时、以及跑起来的服务器。密钥泄露和"线上读到空值"这两类事故，都出在这三条链上。

## 文件本身：位置与最小形状

> 位于 Nuxt 项目根目录的 `nuxt.config.ts` 文件，可以用来覆盖或扩展应用的行为。
>
> 一个最小的配置文件导出一个 `defineNuxtConfig` 函数，里面装着你的配置。`defineNuxtConfig` 这个 helper 是全局可用的，不需要导入。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  // My Nuxt config
})
```

官方还补了它的用途范围：

> 这个文件会在文档里被频繁提到，例如添加自定义脚本、注册模块，或改变渲染模式。

也就是说它同时管三件事：**代码怎么构建（模块、别名、transpile）、应用怎么运行（`ssr`、`routeRules`）、以及配置怎么注入（`runtimeConfig`）**。第三件是本篇主角。

## 第一层环境覆盖：`$production` / `$env`

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  $production: {
    routeRules: {
      '/**': { isr: true },
    },
  },
  $development: {
    //
  },
  $env: {
    staging: {
      //
    },
  },
})
```

> 要在运行 Nuxt CLI 命令时选择某个环境，只需把名字传给 `--envName` 标志，例如：`nuxt build --envName staging`。

用法判据：**差异是"配置整块"级别的**（生产要 CDN 缓存、开发要动态），用这层；**差异是"某个值"级别的**（密钥、地址），用下面的 `runtimeConfig` + 环境变量。

## runtimeConfig：默认只在服务端可见

官方给的形状（005 讲见过，这里补齐读法）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    // 只在服务端可用的私有键
    apiSecret: '123',
    // public 里的键也会暴露给客户端
    public: {
      apiBase: '/api',
    },
  },
})
```

> 当你把 `apiBase` 加入 `runtimeConfig.public` 时，Nuxt 会把它加进每个页面的 payload。我们可以在服务端与浏览器两侧通用地访问 `apiBase`。

读的时候注意三个点：

- `public` 里的东西**会进 payload**，也就是**浏览器能看到**。它不是"隐藏得更深的配置"，是"公开的常量"。
- 顶层键只在服务端。
- 模板里也有入口：官方给的是"公共运行时配置可以在 Vue 模板里用 `$config.public` 访问"。

两侧行为还有一处不对称，官方专门用 important 框写了：

> 行为在客户端与服务端之间是不同的：
> - 在客户端，只有 `runtimeConfig.public` 与 `runtimeConfig.app`（由 Nuxt 内部使用）里的键可用，并且这个对象**既可写又是响应式的**。
> - 在服务端，整个运行时配置都可用，但它是**只读的**，以避免上下文共享。

"服务端只读"这个设计的理由是安全：一份进程要服务多个请求，改它就是把 A 用户的值带给 B 用户。

## 两条硬规则：能不能被环境变量覆盖

官方把规则写成"两个关键要求"：

> 运行时配置的值**会在运行时被匹配的自动环境变量替换**。有两个关键要求：
> 1. 自定义的 runtime config 变量必须在你**`nuxt.config` 里声明**。这是为了确保任意环境变量不会暴露给你的应用代码。
> 2. 只有一个**特定命名**的环境变量能覆盖一个运行时配置属性。也就是说，一个以 `NUXT_` 开头的大写环境变量，它用 `_` 来分隔键与大小写变化。

对照官方那个例子，命名规则一下就清楚了：

```ini [.env]
NUXT_API_SECRET=api_secret_token
NUXT_PUBLIC_API_BASE=https://nuxtjs.org
```

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '', // 可被 NUXT_API_SECRET 环境变量覆盖
    public: {
      apiBase: '', // 可被 NUXT_PUBLIC_API_BASE 环境变量覆盖
    },
  },
})
```

`apiBase` 在 `public` 下 → 变量名要带 `_PUBLIC_`。**没在 `nuxt.config` 里声明的键，环境变量再怎么写也读不到**——这条设计是防泄露，但也是最常踩的"我明明配了环境变量"的原因。

## 三条官方明写的坑

**一、用别的名字的变量做默认值，只在构建期有效。**

> 把 `runtimeConfig` 值的默认值设为**另一个名字的**环境变量（例如把 `myVar` 设为 `process.env.OTHER_VARIABLE`）**只在构建期起作用，在运行时会坏**。建议你使用与你 `runtimeConfig` 对象结构相匹配的环境变量名。

也就是说 `myVar: process.env.OTHER_VARIABLE` 这种写法在 dev 里"看起来是对的"，部署到平台后就固定成了构建时那个值。**正解**：`myVar: ''` + 配 `NUXT_MY_VAR`。

**二、环境变量值会被自动转型。**

> 环境变量的值会**被 `destr` 自动转换成它的 JavaScript 类型**。例如 `NUXT_MY_VAR=4848e0` 会变成数字 `4848`。要让一个值保持字符串，环境变量值本身必须包含字面的双引号：在 `.env` 文件里写 `NUXT_MY_VAR='"4848e0"'`；在直接设置变量时（shell、Dockerfile 或托管平台面板），要确保引号是值的一部分、没有被 shell 剥掉。

**纯数字的密钥或 PIN 会被解析成数字**——这类 bug 只在部分环境出现，极难查。官方甚至专门给了一个视频提示说这是"开发者使用 runtimeConfig 时最容易犯的错误"。

**三、`.env` 在运行时不被读取。**

> Nuxt CLI 内置了对你在开发、构建与 generate 时读取 `.env` 文件的支持。**但当你运行已构建好的服务器时，你的 `.env` 文件不会被读取。**

这条直接决定部署做法：**平台的环境变量面板（或容器 env）才是运行时的来源**，把 `.env` 一起上传是没用的。

## 序列化：不能塞函数

> 你的运行时配置在传给 Nitro 之前会被序列化。这意味着任何无法被序列化再反序列化的东西（例如函数、Set、Map 等），都不应设置在 `nuxt.config` 里。
>
> 不要把不可序列化的对象或函数从 `nuxt.config` 传进你的应用；应把这段代码放进一个 Nuxt 或 Nitro 插件或中间件里。

这条和 005 讲的"配置只是数据"是一体两面：**`nuxt.config` 是构建期求值的 JS，但它的值要经过序列化才能到达运行时**。所以逻辑属于插件，不属于配置。

## appSecret：一个不用声明的例外

`runtimeConfig` 有一处例外值得单独记，因为它关系到会话安全：

> 以 `app` 为前缀的运行时配置键被保留给 Nuxt（目前是 `runtimeConfig.app` 与 `runtimeConfig.appSecret`）。不要把它们用于你自己的值。
>
> Nuxt 提供一个私有的 `runtimeConfig.appSecret` 作为应用的根密钥。模块与端侧特性会用 `deriveSecret` 从它派生各自的密钥，会话辅助函数默认用它来密封。**它的值默认为空字符串，只在服务端可用，并且你可以用 `NUXT_APP_SECRET` 设置它而不需要在 `nuxt.config` 里声明这个键。**

生成方式官方给了命令：

```bash [Terminal]
openssl rand -base64 32
```

以及两条要求：

> 为每个环境使用不同的值，但在该环境的各次部署与各服务端实例之间保持它稳定。改变这个值会使任何依赖它的会话或签名失效。

> 开发时，只有当 `appSecret` 未设置时，Nuxt 才会生成一个随机密钥并存在 `.nuxt/app-secret` 里……**构建时永远不会生成：请在你部署到的每个环境设置 `NUXT_APP_SECRET`。**

这一条正好落在本讲的点上：**"没声明也能被环境变量设置"是官方唯一的例外**，其余键都必须声明。

## 自查

- [ ] 你的每个自定义 runtime config 键，都在 `nuxt.config` 里声明了吗（包括默认空值）？
- [ ] 变量名是不是按 `NUXT_` + 大写 + 下划线分层的规则来的（`public.apiBase` → `NUXT_PUBLIC_API_BASE`）？
- [ ] 有没有写 `myVar: process.env.OTHER` 这种只在构建期有效的默认值？
- [ ] 纯数字或形似 JSON 的密钥值，有没有按官方办法给值本身加引号？
- [ ] 部署时是否用平台环境变量而不是随包上传 `.env`？
- [ ] `nuxt.config` 里有没有函数/Map/Set 之类不可序列化的值？
- [ ] 每个部署环境都设了 `NUXT_APP_SECRET`（生产不会自动生成）？

## 下一篇

019 - 一次访问的完整顺序：服务端七步、浏览器五步与每步能介入的钩子

## 本文依据

- `getting-started/configuration`（nuxt.config.ts 位于根目录用于覆盖或扩展应用行为、最小写法与 defineNuxtConfig 全局可用、"这个文件常用于添加自定义脚本、注册模块或改变渲染模式"、$production/$development/$env 与 --envName 及 c12 机制、runtimeConfig 默认仅服务端且 public/app 暴露客户端、可被环境变量覆盖、app.config.ts 不能被环境变量覆盖）
- `guide/going-further/runtime-config`（apiSecret/public.apiBase 完整示例与"加入 public 会进每个页面 payload"、模板里 $config.public、客户端只有 public/app 且可写且响应式而服务端全部可用但只读以避免上下文共享、两条关键要求（必须声明 + NUXT_ 前缀大写用下划线分隔键与大小写）、.env 示例与对应注释、默认值取其它名字变量只在构建期有效并建议匹配结构、destr 自动类型转换与 4848e0 示例及 .env/shell 里的双引号写法、Nuxt CLI 在 dev/build/generate 读 .env 而运行已构建服务端时不读、序列化限制（函数/Set/Map 不应设置、改放插件或中间件）、app 前缀保留与 appSecret 的默认为空字符串/仅服务端/可用 NUXT_APP_SECRET 而不必声明、openssl rand -base64 32、每环境不同且部署内保持稳定否则会话与签名失效、Nitro 会解析环境覆盖使纯数字或 JSON 样值变成非字符串、开发时未设置才生成随机密钥存 .nuxt/app-secret 且构建不生成故每个部署环境都要设置）
- `getting-started/installation`、`api/commands/init`（--nightly 等 CLI 侧环境相关项属后续篇目）
