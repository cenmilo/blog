---
url: /posts/89c1fcd66e4d2e0a3695d9de85d143ca/
title: 650 - Kit 页面三函数：extendPages 必须原地改、extendRouteRules 与 addRouteMiddleware
date: 2027-05-16T11:00:00+08:00
updated: 2027-05-16T11:00:00+08:00
author: cmdragon

summary:
  摘要：pages 这一节的**三个函数分别改三样东西**：`extendPages` 改路由表、`extendRouteRules` 给某个路由挂 Nitro 的路由规则、`addRouteMiddleware` 注册路由中间件。最关键的一句在 `extendPages` 的参数说明里：**你应该直接修改传给你的那个 pages 数组，改动一份复制出来的数组不会反映到配置里**。路由条目本身有**七个字段**且**全部可选**（`name`、`path`、`file`、`meta`、`alias`、`redirect`、`children`），其中 `path` 不给时由 Nuxt 从文件位置推断。`extendRouteRules` 的选项只有一个 `override`（默认 `false`），`addRouteMiddleware` 的选项有两个：`override`（**同名则替换**）与 `prepend`（排到已有中间件之前）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 路由扩展
  - 路由规则
  - 路由中间件
  - NuxtKit
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

路由是由文件结构生成这件事，模块要插手就得在生成之后改。这一节三个函数对应三个插手点：**路由表、路由规则、路由中间件**。

## `extendPages`：改路由表

先说清默认行为：**在 Nuxt 里，路由是根据 `app/pages` 目录下的文件结构自动生成的**。然后给出手动定制的理由，页面列了三种：**为一个 Nuxt 没生成的动态页面加路由、移除已有路由、修改某个路由的配置**。

```ts
function extendPages (callback: (pages: NuxtPage[]) => void): void
```

参数说明里那句是这一节最容易造成"我明明改了却没生效"的话：

> Note: You should modify the provided pages array directly, as changes made to a copied array will not be reflected in the configuration.

**必须是原地修改**：`push`、`unshift`、`splice`、改元素属性都行；**`map` 出一个新数组再赋值给别的变量则无效**，因为回调的返回类型是 `void`，Nuxt 不看返回值。

官方示例（**注册一个 Nuxt 不会生成的预览页**）：

```ts
import { createResolver, defineNuxtModule, extendPages } from '@nuxt/kit'

export default defineNuxtModule({
  setup (options) {
    const { resolve } = createResolver(import.meta.url)

    extendPages((pages) => {
      pages.unshift({
        name: 'prismic-preview',
        path: '/preview',
        file: resolve('runtime/preview.vue'),
      })
    })
  },
})
```

用 `unshift` 而不是 `push`，是把这条路由放到列表前面。

路由条目的七个字段，**没有一个标必填**：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `name` | `string` | `false` | 路由名，**用于编程式导航与识别路由** |
| `path` | `string` | `false` | 路由 URL 路径；**不设时 Nuxt 从文件位置推断** |
| `file` | `string` | `false` | 作为该路由组件的 Vue 文件路径 |
| `meta` | `NuxtPageMeta` | `false` | 自定义元数据，**可在布局、中间件或导航守卫里使用** |
| `alias` | `string[] \| string` | `false` | 一个或多个别名路径，**用于支持多个 URL** |
| `redirect` | `RouteLocationRaw` | `false` | 重定向规则，**支持命名路由、对象或字符串路径** |
| `children` | `NuxtPage[]` | `false` | 该路由下的嵌套子路由，用于布局或视图嵌套 |

## 给 `meta` 加类型是增广接口

`meta` 想要自定义键并有类型，做法是**增广 `@nuxt/schema` 里的 `NuxtPageMeta`**：

```ts [index.d.ts]
declare module '@nuxt/schema' {
  interface NuxtPageMeta {
    requiresAuth?: boolean
  }
}

export {}
```

```ts [modules/auth.ts]
import { defineNuxtModule, extendPages } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    extendPages((pages) => {
      for (const page of pages) {
        // requiresAuth is now typed as boolean | undefined
        if (page.meta?.requiresAuth) {
          // ...
        }
      }
    })
  },
})
```

页面把这条路径写成了两个入口通用：**当你使用 `extendPages` 或使用 `pages:extend` 钩子时都可以这样加类型**。注意示例里读的是 `page.meta?.requiresAuth`——**`meta` 本身可能不存在**，这也是它被列为可选字段的后果。

## `extendRouteRules`：给路由挂 Nitro 规则

背景句：**Nuxt 由 Nitro 服务端引擎驱动；在 Nitro 里你可以把高层逻辑直接写进配置，这对重定向、代理、缓存、给路由追加响应头这类动作很有用**。**它的工作方式是把路由模式与具体的路由设置关联起来**。

```ts
function extendRouteRules (route: string, rule: NitroRouteConfig, options?: ExtendRouteRulesOptions): void
```

参数只有两句描述：**`route` 是要匹配的路由模式；`rule` 是要应用到匹配路由的路由规则配置**。第三个参数是一张一行的表：

| 名称 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `override` | `boolean` | `false` | 覆盖路由规则配置，默认为 false |

示例把三个函数串在一起——**加一条新路由，再把旧路径重定向过去，并给新路由上缓存**：

```ts
extendPages((pages) => {
  pages.unshift({
    name: 'preview-new',
    path: '/preview-new',
    file: resolve('runtime/preview.vue'),
  })
})

extendRouteRules('/preview', {
  redirect: {
    to: '/preview-new',
    statusCode: 302,
  },
})

extendRouteRules('/preview-new', {
  cache: {
    maxAge: 60 * 60 * 24 * 7,
  },
})
```

**`statusCode: 302` 是显式写的**（不是默认），**`maxAge` 用秒数表达式**，`60 * 60 * 24 * 7` 就是七天。

那一节在 `rule` 下面挂了两条提示，一处指向 Nitro 文档的路由规则章节，一处指向**混合渲染那一章的 Route Rules 小节**——也就是说**能写进 `rule` 的键清单不在这一节**，要按那两处去查。

## `addRouteMiddleware`：注册中间件

定性：**注册路由中间件，让它们对所有路由可用，或只对某些路由可用**。下面紧跟一句容易被跳过的区分：

> Route middlewares can also be defined in plugins via the `addRouteMiddleware` composable.

**同名但不是一个东西**：这一节的是 Kit 函数（构建期），插件里那个是 composable（运行时）。两者指向不同小节，文档给了链接但**没有说同一份中间件用两种方式注册会怎样**。

```ts
function addRouteMiddleware (input: NuxtMiddleware | NuxtMiddleware[], options?: AddRouteMiddlewareOptions): void
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `name` | `string` | `true` | 中间件名 |
| `path` | `string` | `true` | 中间件文件路径 |
| `global` | `boolean` | `false` | 为 `true` 时**把中间件应用到所有路由** |

`input` 可以是对象或数组。第二个参数两个选项：

| 属性 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `override` | `boolean` | `false` | **为 `true` 时替换同名的中间件** |
| `prepend` | `boolean` | `false` | **为 `true` 时把中间件排在已有中间件之前** |

`override` 在这里的语义比路由规则那处更具体：**它管的是"同名怎么办"**——不打开时，同名中间件的行为这一节没有说明。

官方示例是**一个全局登录中间件**，两份文件一起给：

```ts [module.ts]
import { addRouteMiddleware, createResolver, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    const { resolve } = createResolver(import.meta.url)

    addRouteMiddleware({
      name: 'auth',
      path: resolve('runtime/auth'),
      global: true,
    }, { prepend: true })
  },
})
```

```ts [runtime/auth.ts]
export default defineNuxtRouteMiddleware((to, from) => {
  // isAuthenticated() is an example method verifying if a user is authenticated
  if (to.path !== '/login' && isAuthenticated() === false) {
    return navigateTo('/login')
  }
})
```

**`path` 指向的是不带扩展名的文件路径**（`runtime/auth` 而不是 `runtime/auth.ts`）。中间件本体用 `defineNuxtRouteMiddleware` 定义，**跳转用 `return navigateTo('/login')`**——这两点与运行时那一侧的写法完全一致，模块注册只是把文件位置换到 runtime 目录。

## 三个函数各管什么，一张图记住

| 你要的效果 | 用哪个 | 传什么 |
| --- | --- | --- |
| 多一条/少一条路由 | `extendPages` | 回调里原地改数组 |
| 某条路径要重定向、缓存、加响应头 | `extendRouteRules` | 路由模式 ＋ 规则对象（＋ `override`） |
| 每次导航都要过一个检查 | `addRouteMiddleware` | `name`＋`path`（＋`global`），必要时 `prepend` |

顺序上有个隐含关系：示例里**先 `extendPages` 加路由、再对新旧两条路径挂规则**，但**文档没有规定这三者之间的调用先后是否影响结果**（除 `prepend`、`order` 这类显式排序项）。

这一节未记载的还有：`pages:extend` 钩子与 `extendPages` 的执行时机差异、`extendRouteRules` 在多模块给同一路由挂规则时的合并规则、`override: false` 时同名中间件到底谁生效。

下一篇讲 layout 与 head 那两个函数，其中一个要靠 `write: true` 绕开虚拟 `.vue` 文件的限制。

## 下一篇

下一篇 651：Kit 布局与 head 两函数：addLayout 的 write 绕法与 setGlobalHead 的十一项

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/pages
