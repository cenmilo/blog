---
url: /posts/d57e1544c0abfdf004adfaa7b3b8bc6a/
title: 406 - runtimeConfig 的类型增强
date: 2027-02-24T10:00:00+08:00
updated: 2027-02-24T10:00:00+08:00
author: cmdragon

summary:
  摘要：`useRuntimeConfig().apiSecret` 有没有类型，取决于两条路。官方首选是自动的："Nuxt tries to automatically generate a typescript interface from provided runtime config using unjs/untyped."——你在 `nuxt.config` 里写的字面量就是类型来源。第二条是手动增强，官方给的模板是 `declare module 'nuxt/schema'` 里分别写 `interface RuntimeConfig`（私有层）与 `interface PublicRuntimeConfig`（public 层），并且强调末尾那句 `export {}`："It is always important to ensure you import/export something when augmenting a type"。还有一条身份差别：**`nuxt/schema` 是给最终项目用的便利入口，模块作者要增强 `@nuxt/schema`**。类型之外别忘了读法边界：客户端只有 `public` 与 `app` 两层、服务端整个可读但只读。

categories:
  - 前端开发

tags:
  - nuxt4
  - runtimeConfig
  - 类型增强
  - untyped
  - useRuntimeConfig
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`runtimeConfig` 是最容易"类型对不上"的一处：值能被环境变量覆盖、层还分私有和 public，写代码时 `config.apiSecret` 到底有没有提示，取决于你有没有理解这两条路径。

## 路径一：默认由 untyped 自动推

官方 `## Typing Runtime Config` 第一句：

> Nuxt tries to automatically generate a typescript interface from provided runtime config using [unjs/untyped].

也就是**什么都不用写**——你在 `nuxt.config` 里给的这段字面量本身就是类型来源：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: '123',
    public: {
      apiBase: '/api',
    },
  },
})
```

自动推出来的接口就是 `apiSecret: string` 加上 `public.apiBase: string`。前提是**键先在 nuxt.config 里声明过**（342 那条：没声明的键，环境变量也不会凭空造出类型）。

## 路径二：手动增强两个接口

官方紧接着说 "But it is also possible to type your runtime config manually"，给的模板逐字如下：

```ts [index.d.ts]
declare module 'nuxt/schema' {
  interface RuntimeConfig {
    apiSecret: string
  }
  interface PublicRuntimeConfig {
    apiBase: string
  }
}
// It is always important to ensure you import/export something when augmenting a type
export {}
```

三个要点：

1. **两个接口分别对应两层**：`RuntimeConfig` 是顶层（私有、只服务端可读那层），`PublicRuntimeConfig` 管 `public` 层。
2. 增强写在 `.d.ts` 里，用 `declare module 'nuxt/schema'`。
3. **末尾那句 `export {}` 是必需的**——官方注释直说"始终要确保为某个类型做增强时 import/export 一点东西"，少了它整个文件不是模块，增强不生效。

## 身份差别：用 `nuxt/schema` 还是 `@nuxt/schema`

官方 `::note` 划了这条线，也是 396 强调过、最容易写错的一处：

> `nuxt/schema` is provided as a convenience for end-users to access the version of the schema used by Nuxt in their project. Module authors should instead augment `@nuxt/schema`.

判断方法：你在**写应用**就 `declare module 'nuxt/schema'`；你在**写要发布的模块**就 `declare module '@nuxt/schema'`。写错包名不会报语法错，但增强对你项目里的类型不起作用。

## 增强文件要放对上下文

395/396 那条位置规则同样适用：项目被切成 app/node/server/shared 四个类型上下文，"TypeScript will not recognize augmentations placed outside these directories"。想让服务端路由和 app 都看到这个增强，位置要选能覆盖两侧的目录（`shared/` 下的 `.d.ts` 覆盖 app 代码、共享代码与服务端路由，见 055）。

## 别忘了类型之外的读法边界

类型有提示≠哪端都读得到。官方 `::important` 的行为差别（341）：

- 客户端：只有 `runtimeConfig.public` 与 `runtimeConfig.app` 可用，对象**既可写又有响应式**。
- 服务端：整个 runtime config 可读，但**只读**，"to avoid context sharing"。

服务端路由里要拿到被环境变量覆盖后的值，得把 `event` 传进去（官方 `::note`：给 `event` 是可选的，但推荐，"to get the runtime config overwritten by environment variables at runtime for server routes"）：

```ts [server/api/test.ts]
export default defineEventHandler(async (event) => {
  const { apiSecret } = useRuntimeConfig(event)
})
```

还有一条安全边界官方用 `::caution` 标出：**别把 runtime config 的键通过渲染或塞进 `useState` 暴露到客户端**。

## `app` 是保留命名空间

想往 `runtimeConfig.app` 下加自己的键，会撞上保留命名空间。官方在 runtime-config 页写得很硬：

> Runtime config keys prefixed with `app` are reserved for Nuxt (currently `runtimeConfig.app` and `runtimeConfig.appSecret`). Do not use them for your own values.

这正是错误码 B5003 讲的事（410 有逐条拆解）：自定义键挪到 `runtimeConfig.public`（会暴露给客户端）或顶层自定义命名空间（只服务端）。

## 自查

- 默认不用写：untyped 从 `nuxt.config` 里的字面量自动生成接口（前提键已声明）。
- 手动增强就两个接口：`RuntimeConfig`（私有层）、`PublicRuntimeConfig`（public 层），写在 `declare module 'nuxt/schema'` 里且**必须有 `export {}`**。
- 模块作者要增强 `@nuxt/schema`，不是 `nuxt/schema`。
- 增强文件要放对类型上下文目录（app/server/shared）。
- 类型有提示不代表读得到：客户端只见 public/app、服务端整个但只读；服务端路由要传 `event` 才拿运行时覆盖值。
- `runtimeConfig.app` 是保留命名空间（B5003）。

## 下一篇

下一篇 407 转到模块作者侧：`defineNuxtModule` 的 `meta.configKey` 如何让模块选项变成 `nuxt.config` 里的一个键，以及 `meta.compatibility` 与模块依赖约束在启动时报错的行为。

## 本文依据

- `## Typing Runtime Config`：untyped 自动生成接口、手动增强模板（`declare module 'nuxt/schema'` + `RuntimeConfig`/`PublicRuntimeConfig` + `export {}` 注释）、`::note`（`nuxt/schema` 便利 vs 模块作者增强 `@nuxt/schema`）；客户端只有 public/app 可写可响应式、服务端整个只读防上下文共享的 `::important`、服务端路由传 `event` 的 `::note`、`::caution` 别暴露到客户端、`app` 前缀为 Nuxt 保留（`guide/going-further/runtime-config`）；`useRuntimeConfig(event)` 与 `$config.public` 见 341；保留命名空间与 B5003 见 410；类型上下文目录规则见 395/396。
