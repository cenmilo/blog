---
url: /posts/b8da883859492ea0e2ead1cabccdb583/
title: 245 - baseURL 的三个来源：请求选项、runtimeConfig 与 app 命名空间
date: 2027-01-01T11:00:00+08:00
updated: 2027-01-01T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：同一个词 `baseURL` 在文档里出现在三个不同层，混用会出现"本地好、部署挂"。① **单次请求的选项**：`useFetch` 选项表那行只有 "Base URL for the request."（类型 `MaybeRefOrGetter<string>`、无默认），做组合式时可以烤进去（`createUseFetch({ baseURL: 'https://api.nuxt.com' })`），要强制不被调用方改就传函数（官方注释 "baseURL is always enforced, regardless of what the caller passes"）；用 `routes` 声明第三方接口时有一条硬规则：**"The declared paths are matched as written. Do not prefix them with the `baseURL`."** ② **runtimeConfig**：官方示例是 `public: { apiBase: process.env.NUXT_PUBLIC_API_BASE || '/api' }`，用的时候 `$fetch('/test', { baseURL: config.public.apiBase, headers: { Authorization: \`Bearer ${config.apiSecret}\` } })`，并配一句两侧可见性："since `apiBase` is defined within the `public` namespace, it is universally accessible on both server and client-side, while **`apiSecret` is only accessible on the server-side**"。③ **`app` 命名空间**："Nuxt uses `app` namespace in runtime-config with keys including `baseURL` and `cdnURL`. You can customize their values at runtime by setting environment variables." 且 **"This is a reserved namespace. You should not introduce additional keys inside `app`."**——`app.baseURL` 默认 `'/'`、可用 `NUXT_APP_BASE_URL` 在运行时改；`app.cdnURL` 对应 `.output/public` 静态资源。另外三条常被忽略的边界：`$fetch` 打内部 API 路由时**根本不经过网络**（SSR 下直接调函数，"saving an additional API call"），所以内部请求多数不需要 baseURL；`.env` 里的值只在开发/构建期被读，生产要交给平台环境变量；**静态资源那一侧 `app.head` 里写死的 `/favicon.ico` 不会按 `cdnURL` 解析**。

categories:
  - 前端开发

tags:
  - nuxt4
  - baseURL
  - runtimeConfig
  - 部署
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


症状通常是这三句之一："打到了自己的域名上"、"生产环境请求了本地地址"、"挂在子路径下时资源 404"。它们分别属于下面三层。

## 层一：单次请求的 baseURL

`useFetch` 选项表里那行非常短：

| 选项 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `baseURL` | `MaybeRefOrGetter<string>` | - | "Base URL for the request." |

它也可以烤进工厂里，两种模式对应两种意图（都在 226 讲过，这里只摆 baseURL 的视角）：

```ts [默认：调用方可覆盖]
export const useAPI = createUseFetch({
  baseURL: 'https://api.nuxt.com',
  lazy: true,
})

const { data } = await useAPI('/modules', { baseURL: 'https://other-api.com' }) // 覆盖掉
```

```ts [强制：调用方改不动]
// baseURL is always enforced, regardless of what the caller passes
export const useAPI = createUseFetch(callerOptions => ({
  baseURL: 'https://api.nuxt.com',
}))
```

还有一条只出现在类型化那节、但踩的人不少：

> The declared paths are matched **as written**. Do not prefix them with the `baseURL`.

也就是说 `routes` 里写 `/pets`，`baseURL` 是 `https://api.example.com`，**不要写成 `https://api.example.com/pets`**——声明只按字面匹配。

## 层二：runtimeConfig 的 public 与非 public

官方示例把这三件事放在一个配置块里，很值得整块抄：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    // Private keys are only available on the server
    apiSecret: '123',

    // Public keys that are exposed to the client
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || '/api',
    },
  },
})
```

配套那句归属规则："Variables that need to be accessible on the server are added directly inside `runtimeConfig`. Variables that need to be accessible on both the client and the server are defined in `runtimeConfig.public`."

用法（官方在服务端事件处理里演示）：

```ts [server/api/test.ts]
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig(event)

  // Access public variables
  const result = await $fetch(`/test`, {
    baseURL: config.public.apiBase,
    headers: {
      // Access a private variable (only available on the server)
      Authorization: `Bearer ${config.apiSecret}`,
    },
  })
  return result
})
```

**这段里两个细节我要划出来**：一是 `useRuntimeConfig(event)` 在服务端**要传 `event`**（073 那条限制）；二是官方总结句——"`apiBase` is defined within the `public` namespace, it is universally accessible on both server and client-side, while `apiSecret` **is only accessible on the server-side**."

### 环境变量只在两侧时机不同

`NUXT_` 前缀覆盖这条我按原话记三条：

- 覆盖写法：`NUXT_PUBLIC_API_BASE = "https://api.localhost:5555"`、`NUXT_API_SECRET = "123"` 放在 `.env` 里。
- **`.env` 的读取时机**："Any environment variables set within `.env` file are accessed using `process.env` in the Nuxt app during **development** and **build/generate**."
- **生产不用 `.env`**："In **production runtime**, you should use platform environment variables and `.env` is not used."

所以"我在 `.env` 改了 baseURL，本地有效线上无效"是文档明写的行为，不是意外。

（073 那篇讲 `.env` 时还记过一条：删掉变量不会取消已经设过的值——生产只认 `NUXT_` 前缀，同一族规则。）

## 层三：app 命名空间是保留的

原文两段：

> Nuxt uses `app` namespace in runtime-config with keys including `baseURL` and `cdnURL`. You can customize their values at runtime by setting environment variables.
>
> This is a reserved namespace. You should not introduce additional keys inside `app`.

`app.baseURL` 那小节给了默认与运行时覆盖：

> By default, the `baseURL` is set to `'/'`.
> However, the `baseURL` can be updated at runtime by setting the `NUXT_APP_BASE_URL` as an environment variable.

访问方式是通用的（插件里）：

```ts [/plugins/my-plugin.ts]
export default defineNuxtPlugin((NuxtApp) => {
  const config = useRuntimeConfig()

  // Access baseURL universally
  const baseURL = config.app.baseURL
})
```

而 `app.cdnURL` 管的是另一件事："You can use a custom CDN for serving static assets inside `.output/public` using the `NUXT_APP_CDN_URL` environment variable."

**这条就是 201/222 那个坑的另一面**：静态资源会按 `cdnURL` 解析、回落到 `app.baseURL`，但**写在 `app.head` 里的字面路径（如 `/favicon.ico`）不会被解析**——要指到 CDN 上就得自己用 `useRuntimeConfig().app` 拼。

三层的关系我用一句我自己的话总结（**文档没有这样归纳过**）：层一决定"这一次请求打到哪"，层二决定"这份配置谁能看见"，层三决定"应用本身挂在哪、静态资源从哪发"。

## 内部请求其实常常不需要 baseURL

这一条值得放在最后，因为它能省掉一半的 baseURL 配置。211 与 238 两处都写着同一件事：

> During server-side rendering, calling `$fetch` to fetch your internal [API routes] will directly call the relevant function (emulating the request), **saving an additional API call**.

> `$fetch` is a configured instance of ofetch which supports adding the base URL of your Nuxt server as well as direct function calls during SSR (avoiding HTTP roundtrips).

所以对 `server/api/` 那些路由，**写相对路径就行**：浏览器里同源发出，服务端直接调函数。真要 baseURL 的场景是**外部 API**（层一或层二）与**子路径部署**（层三）。

## 一处文档空白，别猜

**`useFetch('/api/x')` 这种相对路径会不会自动带上 `app.baseURL` 前缀？** 222 已经记过这条：选项表只说 "Base URL for the request."，配置说 `app.baseURL` 是应用的基础路径，**两者关系没有任何一句描述**。所以我的判据是实测：

1. `NUXT_APP_BASE_URL=/prefix/ node .output/server/index.mjs` 起生产服务（官方给过这条命令）。
2. 打开一个会请求 `/api/...` 的页面，看网络面板里实际请求的是 `/api/...` 还是 `/prefix/api/...`。
3. 同时 `curl` 服务端日志确认那次是不是根本没走网络（内部直调）。

## 自查

1. `routes` 声明里的路径能不能带 baseURL 前缀？（答：不能，官方原话 "Do not prefix them with the `baseURL`."，且按字面匹配。）
2. `apiBase` 与 `apiSecret` 谁能被浏览器看到？（答：`apiBase` 在 `public` 命名空间里、两侧可见；`apiSecret` 只有服务端可访问。）
3. 子路径部署时改哪个值，能不能自己往那个命名空间加键？（答：`app.baseURL`，运行时用 `NUXT_APP_BASE_URL`；`app` 是保留命名空间，不应加自己的键。）

## 下一篇

246 - 刷新与导航：refreshNuxtData、key 与保活组件的竞态

## 本文依据

- `api/composables/use-fetch`：选项表 `baseURL` 行与 `query`/`params`/`cache` 相邻各行；`createUseFetch` 的 tip（"pre-defined defaults (like `baseURL` or auth headers）"）。
- `api/composables/create-use-fetch`：`baseURL: 'https://api.nuxt.com'` 两处示例、Default vs Override Mode 两段与其注释句、`routes` 一节 "Do not prefix them with the `baseURL`." 与匹配规则。
- `api/composables/use-runtime-config`：runtimeConfig 配置块（`apiSecret` 与 `public.apiBase` 的注释）、public/非 public 归属句、`server/api/test.ts` 示例与两侧可见性总结句、环境变量段（`NUXT_PUBLIC_API_BASE`/`NUXT_API_SECRET` 示例、`.env` 只在 development 与 build/generate、生产不用 `.env`）、`app` 命名空间两段（含 reserved namespace 的 note）、`app.baseURL` 三句与插件示例、`app.cdnURL` 与 `NUXT_APP_CDN_URL`。
- `api/utils/dollarfetch`、`guide/recipes/custom-usefetch`：SSR 打内部 API 直接调用函数、省一次请求的两句（211/238 已逐句引）。
- `api/nuxt-config`（app.baseURL）：说明句、默认 `"/"`、`NUXT_APP_BASE_URL` 与启动命令示例（222 已逐句引）。
- `getting-started/seo-meta`：`cdnURL` 解析与"字面路径不解析"整段 note（186/201 已逐句引）。
