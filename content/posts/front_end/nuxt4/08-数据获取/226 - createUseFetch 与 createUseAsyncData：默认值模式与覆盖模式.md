---
url: /posts/fe19a0767c96fc9a8b74ccb391b6de3a/
title: 226 - createUseFetch 与 createUseAsyncData：默认值模式与覆盖模式
date: 2026-12-26T10:00:00+08:00
updated: 2026-12-26T10:00:00+08:00
author: cmdragon

summary:
  摘要：这两工厂函数解决的是同一件事——**别再每个页面重复写 baseURL 和鉴权头**——官方定位句分别是 "A factory function to create a custom useFetch composable with pre-defined default options." 与 createUseAsyncData 的同款句子，并各补一句 "The resulting composable is fully typed and works exactly like `useFetch`, but with your defaults baked in."。有两处必须先知道的约束：它们**都是编译器宏**，而且**必须是以 export 声明的形式出现在 `composables/`（或任何被编译器扫描的目录）里**——"It must be used as an **exported** declaration in the `composables/` directory (or any directory scanned by the Nuxt compiler). Nuxt automatically injects de-duplication keys at build time."。**传对象与传函数是两种语义**，官方分成两节写：Default Mode 里 "the factory options act as **defaults**. Callers can override any option"；Override Mode 里 "the factory options **override** the caller's options. The function receives the caller's options as its argument"，并给了用途句 "This is useful for enforcing settings like authentication headers or a specific base URL that should not be changed by the caller."。`createUseFetch` 还多一条第三重载：传 `routes` 给第三方接口做类型化（"Only the type of `routes` is read, so pass `{} as Routes`"、路径**按写下的样子匹配、不要加 baseURL 前缀**、运行时拼出来的路径解析为 `unknown`、要拒绝未匹配路径就同时开 `routeTypedFetch` 与 `strictRouteTypes`）。最后一条我认为最容易踩：想在新组合式里用 `useNuxtApp()`，**必须用函数形态**，官方的 important 解释了原因——要在调用点的 setup 上下文里取实例，而不是在模块作用域。

categories:
  - 前端开发

tags:
  - nuxt4
  - createUseFetch
  - 工厂函数
  - 类型化
  - 取数

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我在每个页面重复写 `baseURL` 第五次之后才去查这个。它属于"官方早就给了出口，只是页面藏在很后面"的那类。

## 两个宏，约束一样

定位句两节同形：

> A factory function to create a custom useFetch composable with pre-defined default options.
> A factory function to create a custom useAsyncData composable with pre-defined default options.

再各加一句：

> The resulting composable is fully typed and works exactly like `useFetch`, but with your defaults baked in.

**约束那条我要单独抄**，因为它决定了"为什么我写在别处的不生效"：

> `createUseFetch` is a compiler macro. It must be used as an **exported** declaration in the `composables/` directory (or any directory scanned by the Nuxt compiler). Nuxt automatically injects de-duplication keys at build time.

拆开是四件事：

1. 它是**宏**，不是普通函数（和 163 讲的宏同一族）。
2. 要写成 **export 的声明**，`export const useAPI = createUseFetch({...})`。
3. 文件要落在 `composables/`（或任何被扫描目录）里——**这正好接上 151 讲的扫描范围**。
4. 去重 key 由 Nuxt **在构建期注入**，所以我不必为它命名 key（但 215 那张共享表仍然适用：想跨组件共享还是要显式给）。

## 默认用法两节各一例

```ts [app/composables/useAPI.ts]
export const useAPI = createUseFetch({
  baseURL: 'https://api.nuxt.com',
})
```

```vue [app/pages/modules.vue]
<script setup lang="ts">
const { data: modules } = await useAPI('/modules')
</script>
```

```ts [app/composables/useCachedData.ts]
export const useCachedData = createUseAsyncData({
  getCachedData (key, nuxtApp) {
    return nuxtApp.payload.data[key] ?? nuxtApp.static.data[key]
  },
})
```

第二个例子有意思的地方在于：它把 213 那段"默认缓存实现"**手写了一遍当默认值**——官方选这个例子，说明这确实是工厂最常见的用途之一（统一缓存策略）。

选项范围官方各给一句：`createUseFetch` "accepts all the same options as `useFetch`, including `baseURL`, `headers`, `query`, `onRequest`, `onResponse`, `server`, `lazy`, `transform`, `getCachedData`, and more"；`createUseAsyncData` 那条列的是 `server`、`lazy`、`immediate`、`default`、`transform`、`pick`、`getCachedData`、`deep`、`dedupe`、`timeout`、`watch`。

## 传对象与传函数是两种语义

这是这一篇最该记住的一段，官方分成两个小节：

| 模式 | 官方原话 | 效果 |
| --- | --- | --- |
| Default Mode（传对象） | "When you pass a plain object, the factory options act as **defaults**. Callers can override any option." | 调用方能改，工厂值只是缺省 |
| Override Mode（传函数） | "When you pass a function, the factory options **override** the caller's options. The function receives the caller's options as its argument, so you can read them to compute your overrides." | **调用方改不动**，工厂说了算 |

官方给的两个例子对比很干净：

```ts [默认模式：调用方可覆盖]
export const useAPI = createUseFetch({
  baseURL: 'https://api.nuxt.com',
  lazy: true,
})

const { data } = await useAPI('/modules')                                  // 用默认 baseURL
const { data } = await useAPI('/modules', { baseURL: 'https://other-api.com' }) // 覆盖掉
```

```ts [覆盖模式：强制]
// baseURL is always enforced, regardless of what the caller passes
export const useAPI = createUseFetch(callerOptions => ({
  baseURL: 'https://api.nuxt.com',
}))
```

`createUseAsyncData` 那一节的对应例子只强制一个布尔，读起来更直白："`deep` is always enforced as false"。

官方对 Override Mode 的用途句是："This is useful for enforcing settings like authentication headers or a specific base URL that should not be changed by the caller." ——**"不该被调用方改"就是选它的唯一理由**。我一开始反过来用（传对象当"强制"），结果某个页面自己传了 baseURL，鉴权头还在但打到别的域名上，查了半天。

## 想在新组合式里用 useNuxtApp：只能用函数形态

这一条官方专门挂了一个 important，理由讲得很具体：

> The **function signature** (override mode) is required here so that `useNuxtApp()` is called in the setup context (at the composable call site) rather than in the module scope, where no Nuxt instance is available.

配合示例：

```ts [app/composables/useAPI.ts]
export const useAPI = createUseFetch(callerOptions => ({
  $fetch: useNuxtApp().$api as typeof $fetch,
  ...callerOptions,
}))
```

读法是：函数体在**每次调用时**执行，所以那一次能拿到 Nuxt 实例；传对象的话表达式在模块加载时求值，**那时候没有实例**。这和 130 讲过的"上下文"问题是同一个根，只是这里的解法是"换成函数形态"。

顺带这一例还展示了 `$fetch` 选项的真实用法（220 表里那行 "Custom $fetch implementation." 在这里落地）。

## createUseFetch 独有的第三条：给第三方接口做类型

从 Nuxt 4.6 起 `createUseFetch` 可以传 `routes` 描述一个外部 API：

> From Nuxt 4.6, pass `routes` to describe the API a composable calls. Its responses, query parameters, and request bodies are typed from that route set, including when `experimental.routeTypedFetch` is off.

形状（官方示例节选）：

```ts [app/composables/usePetStore.ts]
import type { DynamicParam, Endpoint } from 'nuxt/app'

interface Pet { id: number, name: string }

interface PetStoreRoutes {
  '/pets': {
    [Endpoint]: {
      GET: { response: Pet[], query: { limit?: number } }
      POST: { response: Pet, body: { name: string } }
    }
    [DynamicParam]: {
      [Endpoint]: { GET: { response: Pet } }
    }
  }
}

export const usePetStore = createUseFetch({
  baseURL: 'https://api.example.com',
  routes: {} as PetStoreRoutes,
})
```

四个限制官方写得非常具体，我逐条抄，因为这几条最容易踩：

> Only the type of `routes` is read, so pass `{} as Routes`; the value is dropped before the request is made. The declared paths are matched **as written**. Do not prefix them with the `baseURL`. A path built at runtime resolves to `unknown`.

- 值被丢掉，**只读类型**，所以写 `{} as Routes`。
- 声明的路径**按字面匹配**，且**不要带 baseURL 前缀**（示例里 baseURL 是 `https://api.example.com`，路径写 `/pets`）。
- **运行时拼出来的路径**（模板串）类型解析为 `unknown`——这条我原本以为会自动匹配动态段，官方明确不会。

未匹配的路径与方法默认**接受并解析为 `unknown`**；要让它报错，同时开两个实验开关：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    routeTypedFetch: true,
    strictRouteTypes: true,
  },
})
```

官方给了判例：开了之后 `usePetStore('/pats')` 与 `usePetStore('/pets', { method: 'put' })` 都是类型错误（**注意 `/pats` 那个拼写错误正是这套类型想抓的东西**）。

两条补充我也照录，因为它们防止我把它想大：

> The routes a client declares are its own. They are not added to your app's route set, so plain `$fetch` and `useFetch` are unaffected. This client types requests against its declared routes, not your own server's routes.

> The interface above is the shape `fetchdts` uses. A module can generate one from an API description, such as an OpenAPI document, with `compileRoutes` from `fetchdts/compiler`, and pass the emitted interface to `routes`.

也就是说：**给自己项目的 `server/api/` 类型化不是它的用途**（219 讲过 `useFetch` 的 URL 类型提示本来就来自你的服务端路由）。

## 签名三条一起看

```ts
function createUseFetch (
  options?: Partial<UseFetchOptions> & { addons?: UseFetchAddon[] },
): typeof useFetch

function createUseFetch (
  options: (callerOptions: UseFetchOptions) => Partial<UseFetchOptions>,
): typeof useFetch

function createUseFetch<Routes> (
  options: Partial<UseFetchOptions> & { routes: Routes, addons?: UseFetchAddon[] },
): DeclaredUseFetch<Routes>
```

返回类型是 `typeof useFetch`（或带 routes 时的 `DeclaredUseFetch<Routes>`）——**所以新组合式的参数与返回值和原来一模一样**，官方那句 "The resulting `useAPI` composable has the same signature and return type as `useFetch`, with all options available for the caller to use or override." 就是这个意思。`createUseAsyncData` 只有前两条重载（没有 routes）。

addons 那部分留给下一篇，两节的说明句是同一句："Addons are reusable units of behavior defined with `defineUseFetchAddon` / `defineUseAsyncDataAddon`. They can declare custom call-site options, run middleware around the handler, extend the returned object, and attach custom logic to the composable."

## 自查

1. 传对象和传函数的语义差别是什么？（答：对象是默认值、调用方可覆盖；函数是强制覆盖，且能读到调用方选项。）
2. 为什么工厂里要用 `useNuxtApp()` 就必须用函数形态？（答：对象形态在模块作用域求值，那里没有 Nuxt 实例；函数在调用点的 setup 上下文里执行。）
3. `routes` 类型化有四条什么限制？（答：只读类型要写 `{} as Routes`；路径按字面匹配且不写 baseURL 前缀；运行时拼的路径为 `unknown`；默认不拒绝未匹配，要拒绝得同时开 `routeTypedFetch` 与 `strictRouteTypes`。）

## 下一篇

227 - 给取数函数加插件：addon 的自定义选项、中间件与扩展返回值

## 本文依据

- `api/composables/create-use-fetch`：定位句与 "fully typed / works exactly like useFetch / defaults baked in" 两句；note（compiler macro、exported declaration、composables 目录、构建期注入去重 key）；Usage 两例（`useAPI` 与调用页）；Type 三条重载；Options 段的键清单；Typing a Third-Party API 全节（4.6 起、`DynamicParam`/`Endpoint` 示例、三条使用结果、四条限制原文、两个实验开关与两个类型错误判例、两条 note/tip）；Default vs Override Mode 两节原文与三块示例；Combining with a Custom `$fetch` 段与 important（函数形态才能拿 `useNuxtApp()`）；Addons 段与 `refreshOnFocus` 示例。
- `api/composables/create-use-async-data`：定位句与同款两句；同一条 macro note；`useCachedData` 的 `getCachedData` 示例；Type 两条重载；Options 段列出的 11 个键；Default Mode（`lazy: true` + `server: false` 与调用方覆盖 `server: true`）；Override Mode（`deep` is always enforced as false）；Addons 段与 `useCustomAsyncData` 示例。
- `api/composables/use-fetch`：`$fetch` 选项行 "Custom $fetch implementation."（220 已引）；URL 类型提示来自服务端路由那句（219 已引）。
