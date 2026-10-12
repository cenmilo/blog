---
url: /posts/bca9f3135ac626e8f12306dc8b307c44/
title: Kit 上下文两函数：useNuxt 会抛错、tryUseNuxt 返回 null
date: 2027-05-16T10:00:00+08:00
updated: 2027-05-16T10:00:00+08:00
author: cmdragon

summary:
  摘要：这两个函数只解决一件事：**在没把 `nuxt` 一路传参下来的地方，从上下文里把它取出来**。页面那条 `::note` 先把大多数场景排除掉——**在模块的 `setup` 里，Nuxt 已经作为第二个参数给你了，不需要调 `useNuxt()`**。真正需要它们的是"把模块拆成多个小文件"的时候。两者唯一的行为差别：**`useNuxt` 取不到时抛错，`tryUseNuxt` 取不到时返回 `null`**。`useNuxt` 的返回值那一节还顺带给出了实例的公开面，一共**五项：`options`、`hooks`、`hook`、`callHook`、`addHooks`**（后三项是 `nuxt.hooks` 上的简写）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 上下文
  - useNuxt
  - NuxtKit
  - 模块拆分
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块写到第三四个文件时会出现一个具体问题：辅助函数里要用配置，但 `setup` 传下来的 `nuxt` 只到第二层。这两个函数就是为这一层准备的。

## 为什么需要它

页面给的动机句是：**模块让你增强 Nuxt 的能力，是保持代码有条理、模块化的结构化方式；如果你打算把自己的模块拆成更小的部分，Nuxt 提供了 `useNuxt` 与 `tryUseNuxt`**——**它们让你方便地从上下文访问 Nuxt 实例，而不必把它作为参数一路传下去**。

紧接着那条 `::note` 划掉了最常见的误用：

> When you're working with the `setup` function in Nuxt modules, Nuxt is already provided as the second argument. This means you can access it directly without needing to call `useNuxt()`.

也就是说：**`setup (options, nuxt)` 里再写 `const nuxt = useNuxt()` 是多余的**。这两个函数的位置在 `setup` 之外的那些文件里。

## `useNuxt`：取不到就抛错

```ts
function useNuxt (): Nuxt
```

定性一句：**从上下文取得 Nuxt 实例；**如果 Nuxt 不可用，**它会抛出错误**。

```ts
import { useNuxt } from '@nuxt/kit'

const setupSomeFeature = () => {
  const nuxt = useNuxt()

  // You can now use the nuxt instance
  console.log(nuxt.options)
}
```

页面在 Return Value 一节把实例的公开面列成五项，这份清单其实可以直接当"模块里能对 Nuxt 做什么"的入口：

| 属性 | 类型 | 说明 |
| --- | --- | --- |
| `options` | `NuxtOptions` | **已解析的 Nuxt 配置** |
| `hooks` | `NuxtHookRegistry<NuxtHooks>` | Nuxt 的钩子系统，允许注册与监听生命周期事件 |
| `hook` | `(name, (...args) => Promise<void> \| void) => () => void` | **`nuxt.hooks.hook` 的简写**，为某个生命周期钩子注册单个回调 |
| `callHook` | `(name, ...args) => Promise<any>` | **`nuxt.hooks.callHook` 的简写**，手动触发一个钩子并运行所有已注册回调 |
| `addHooks` | `(configHooks: NuxtNestedHooks) => () => void` | **`nuxt.hooks.addHooks` 的简写**，一次注册多个钩子 |

三项"简写"是这一节的信息量所在：**`hook`、`callHook`、`addHooks` 都不是独立机制，而是 `nuxt.hooks` 上的转发**，所以能传给 `hooks` 的东西都能传给这三个。另外两个返回的是**取消注册的函数**（`() => void`），文档写了类型但没解释用途。

示例是"拆出去的辅助函数"标准形态——**只在特定构建器下做一件事**：

```ts [setupTranspilation.ts]
import { useNuxt } from '@nuxt/kit'

export const setupTranspilation = () => {
  const nuxt = useNuxt()

  if (nuxt.options.builder === '@nuxt/webpack-builder') {
    nuxt.options.build.transpile ||= []
    nuxt.options.build.transpile.push('xstate')
  }
}
```

```ts [module.ts]
import { setupTranspilation } from './setupTranspilation'

export default defineNuxtModule({
  setup () {
    setupTranspilation()
  },
})
```

三行细节都值得照抄：**`setup` 里不再接收参数**（辅助函数自己从上下文取）、**按 `nuxt.options.builder` 判断当前构建器**、**push 之前先 `||= []` 兜住可能不存在的数组**。

## `tryUseNuxt`：取不到返回 null

```ts
function tryUseNuxt (): Nuxt | null
```

**同一个动作的宽容版本**：可用时返回实例，不可用时返回 `null`。返回值一节说明它返回的对象就是 `useNuxt` 那节描述的那份。

```ts
import { tryUseNuxt } from '@nuxt/kit'

function setupSomething () {
  const nuxt = tryUseNuxt()

  if (nuxt) {
    // You can now use the nuxt instance
    console.log(nuxt.options)
  } else {
    console.log('Nuxt is not available')
  }
}
```

它的示例比 `useNuxt` 那个多了一层信息——**给自己加的配置键上类型**：

```ts [requireSiteConfig.ts]
declare module '@nuxt/schema' {
  interface NuxtOptions {
    siteConfig: SiteConfig
  }
}
// ---cut---
import { tryUseNuxt } from '@nuxt/kit'

interface SiteConfig {
  title?: string
}

export const requireSiteConfig = (): SiteConfig => {
  const nuxt = tryUseNuxt()
  if (!nuxt) {
    return {}
  }
  return nuxt.options.siteConfig
}
```

```ts [module.ts]
import { defineNuxtModule, useNuxt } from '@nuxt/kit'
import { requireSiteConfig } from './requireSiteConfig'

export default defineNuxtModule({
  setup (_, nuxt) {
    const config = requireSiteConfig()
    nuxt.options.app.head.title = config.title
  },
})
```

这段里有三处各自独立的做法：**通过 `declare module '@nuxt/schema'` 增广 `NuxtOptions` 来给自己新增的顶层键补类型**（和服务端兼容性那一章增广 `RuntimeConfig` 是同一族写法）；**取不到实例时返回空对象而不是抛错**，让辅助函数在 Nuxt 之外也能被调用；**`setup` 这一侧用 `_` 占住第一个参数**，因为只用到实例。

注意这两份示例并不自洽：`module.ts` 里 import 了 `useNuxt` 却没有用它。**照抄时要删掉这个未使用的导入**，否则类型检查会报。

## 选哪个

| 情况 | 选 | 理由 |
| --- | --- | --- |
| 在 `setup` 里 | 都不用 | 第二个参数已经给了 |
| 模块内部的辅助文件，且它只在构建流程里被调用 | `useNuxt` | **取不到说明调用位置错了，抛错正好把它暴露出来** |
| 一个可能被 Nuxt 上下文之外复用的工具函数 | `tryUseNuxt` | 需要一条"没有实例时怎么办"的分支 |

判断依据其实是**抛错对你是不是有效信号**：在模块流程里，没有实例＝编程错误，越早炸越好；在共享工具里，没有实例＝正常情况之一。

这一页没写的：**"从上下文取"的机制是什么**（没有提到 async context 或作用域如何建立），以及**在 `hooks` map 注册的回调里 `useNuxt()` 是否可用**——这两点决定了很多拆法的可行性，但页面没有给出答案，只能实测。

另外记得 Kit 的总前提：**这两个函数只给模块用，运行时那五种位置（组件、组合式函数、页面、插件、服务端路由）都不该导入它们**。

下一篇讲 pages 那三个函数，其中 `extendPages` 有一条"必须原地改数组"的硬要求。

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/context
