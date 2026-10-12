---
url: /posts/77546685d844efcf6e22d51233c316d1/
title: 写进 nitro 节的东西，记载全在别的页
date: 2027-04-18T09:00:00+08:00
updated: 2027-04-18T09:00:00+08:00
author: cmdragon

summary:
  摘要：`nitro` 在配置参考里不列清单，但**确实有几个键是 Nuxt 文档在别处教过怎么写的**：**`nitro.prerender`**——预渲染那页明写"顶层 `prerender` 选项对任何 `server.builder` 都生效；Nitro 专属的选项（官方举 **`concurrency`** 与 **`failOnError`**）要写在 `nitro.prerender`"；**`nitro.storage`**——服务端目录那页给的是"要加存储挂载点，用 `nitro.storage` 或 server plugins"，并附一整份 Redis 示例（`driver`/`port`/`host`/`username`/`password`/`db`/`tls`）与 `useStorage('redis')` 的三个方法。还有一条容易挑错的：**`routeRules` 有顶层节，`nitro.routeRules` 也有一个子项**，两处都只写 `object`。这一篇把"能照文档写的 nitro 键"和"文档没写、只能自己试的"分开。

categories:
  - 前端开发

tags:
  - nuxt4
  - nitro
  - 预渲染
  - 存储
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`nitro` 那一节不列清单，但你真要用某些 Nitro 能力时，Nuxt 文档其实在别的页写好了写法——只是不会有人在那里找到它。

## 一、nitro.prerender：两个例子官方点了名

预渲染那页在讲完顶层 `prerender` 之后，专门补了一句分工：**"The top-level `prerender` option applies whichever `server.builder` you use. Builder-specific options, such as Nitro's `concurrency` or `failOnError`, can be set in `nitro.prerender`."**

于是这条边界是清楚的：

| 你要设的东西 | 写在哪 |
| --- | --- |
| `routes`、`ignore`、`crawlLinks` 这类跨构建器的选项 | 顶层 `prerender` |
| Nitro 专属的 `concurrency`、`failOnError` | `nitro.prerender` |

官方点名的这两项**只出现在这一句里，没有任何示例**——所以能确证的只有"它们是 Nitro 侧的、写在这个位置"，参数类型与取值范围得去 Nitro 文档查（那页末尾留的正是 `nitro.build/config#prerender` 的 read-more）。

一个容易犯的错：把 `failOnError` 写进顶层 `prerender`。按上面那句的分工它属于 builder-specific，位置错了不会报错，只会静默不生效——**这正是"顶层透传"型配置最难查的地方**。

## 二、nitro.storage：一整份示例都在服务端那页

Server Storage 小节给的是定性加两条入口：**"Nitro provides a cross-platform storage layer."**、**"In order to configure additional storage mount points, you can use `nitro.storage`, or server plugins."**

随后是全套示例（Redis）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  nitro: {
    storage: {
      redis: {
        driver: 'redis',
        /* redis connector options */
        port: 6379, // Redis port
        host: '127.0.0.1', // Redis host
        username: '', // needs Redis >= 6
        password: '',
        db: 0, // Defaults to 0
        tls: {}, // tls/ssl
      },
    },
  },
})
```

然后在处理函数里按挂载名取：

```ts [server/api/storage/test.ts]
export default defineEventHandler(async (event) => {
  const keys = await useStorage('redis').getKeys()
  await useStorage('redis').setItem('foo', 'bar')
  await useStorage('redis').removeItem('foo')
  return {}
})
```

两点要挑出来。**其一，键名 `redis` 是挂载点名字**，`useStorage('redis')` 用同一个字符串取——**这份示例里的名字随你起，官方没给命名约束**。其二，**连接凭据被写在 `nuxt.config.ts` 里**：官方在这一节没给"从环境变量取值"的写法，另一条路它自己点明了——用 server plugin 加 runtime config。**把密码提交进仓库不是这份示例的用法**，`username`/`password` 在原示例里就是空串占位。

## 三、routeRules 有两处，都不写内容

| 位置 | 记载 |
| --- | --- |
| 顶层 `## routeRules` | "Global route options applied to matching server routes." ＋ **Experimental** 标注 ＋ See 指向 Nitro 的 route rules |
| `### nitro.routeRules` | 只有 `- **Type**: object` |

**两处都不列可用属性。** 九个属性（`redirect`、`ssr`、`cors`、`headers`、`swr`、`isr`、`prerender`、`noScripts`、`appMiddleware`）的清单只写在渲染概念页。**所以"该写顶层还是 `nitro.` 下"这个问题，文档没有正面回答**——能参照的是 `directory-structure/server` 把 `routeRules` 当**顶层键**用（示例 `routeRules: { '/**': { isr: true } }` 出现在环境覆盖那节），部署页与预渲染页的示例也全是顶层写法。

**结论只能是：官方所有示例都写在顶层，`nitro.routeRules` 这个子项没有任何示例。** 要动它，先自己构建验证一次。

## 四、其余的 nitro 键：Nuxt 侧没有记载

`nitro` 下还能写什么，本系列能安全说出的只有：**除上面这些被点名的键之外，Nuxt 文档没有再教过任何 `nitro.*` 选项**（`typescript.builder` 那种是 `typescript` 节的事，`nitro.runtimeConfig` 已在上一篇）。要用别的 Nitro 能力，路径是：先查 Nitro 配置文档 → 写进 `nitro` → 构建验证 → 升级 Nuxt 次版本时复查（那句 `::warning` 说的就是这件事）。

## 自查

- `nitro.prerender` 承载 Nitro 专属的预渲染选项，官方点名 `concurrency` 与 `failOnError`；跨构建器的选项（`routes`/`ignore`/`crawlLinks`）在顶层 `prerender`；**位置写错不会报错，只会不生效**。
- `nitro.storage` 是加存储挂载点的两条官方入口之一（另一条是 server plugins），Redis 示例给了完整连接器字段；**凭据别照示例写进配置**，官方另指 runtime config 那条路。
- `routeRules` 在顶层与 `nitro` 下各有一处，两处都不列属性；官方全部示例都用顶层写法，`nitro.routeRules` 无示例。
- 九个属性的语义只在渲染概念页；把它当"这节的清单"会查不到。
- 其余 `nitro.*` 键 Nuxt 文档不教，只能查 Nitro 并构建验证。

## 下一篇

下一篇 565 读 `server` 节：它只有一个子项 `builder`，默认 `@nuxt/nitro-server`；`"nitro"` 与 `"vite"` 是两个简写，而后者的行为差别很大——`ssr: false` 出静态 SPA，带 SSR 时产物入口导出 web 标准 `{ fetch }`，且**没有服务端运行时，因此服务端路由、除 `prerender` 外的路由规则、内联样式与服务端组件都不可用**。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/prerendering（顶层 `prerender` 与 `nitro.prerender` 的分工句及 `concurrency`/`failOnError` 点名、指向 nitro.build/config#prerender 的 read-more）
- https://nuxt.com/docs/4.x/directory-structure/server（Server Storage 小节：cross-platform storage 定性、`nitro.storage` 与 server plugins 两条入口、Redis 示例全文与 `useStorage('redis')` 三个方法、另一条 runtime config 路径；Nitro Config 小节的 `nitro` 透传句与影响生产部署的 `::warning`；环境覆盖示例里 `routeRules` 写在顶层）
- https://nuxt.com/docs/4.x/api/nuxt-config（`### nitro.routeRules` 仅 `object`、`## routeRules` 顶层节的三行与 Experimental 标注）
- https://nuxt.com/docs/4.x/guide/concepts/rendering（Route Rules 九个属性清单，用于说明语义所在处）
