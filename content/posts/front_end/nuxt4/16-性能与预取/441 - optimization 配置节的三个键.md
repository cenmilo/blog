---
url: /posts/6a2c39a45c0cfbe502ad514eb5d1a2d5/
title: 441 - optimization 配置节的三个键
date: 2027-03-08T09:00:00+08:00
updated: 2027-03-08T09:00:00+08:00
author: cmdragon

summary:
  摘要：`optimization` 这一节只有一句定性——"Build time optimization configuration"，也就是构建期优化，下面三个键没有一个是你以为的水合开关。`asyncTransforms` 把选项直接传给 `unctx` 的转换器，用于在 `await` 之后保留异步上下文，默认登记 `defineNuxtPlugin`、`defineNuxtRouteMiddleware` 两个函数和三组对象定义。`keyedComposables` 给七个函数注入一个末位 magic string，官方明说"你需要自己处理这个多出来的参数"。`treeShake.composables` 的默认清单才是与水合规则有关的那一半：服务端包里摇掉十个 Vue 生命周期钩子，客户端包里摇掉 `onPrehydrate` 等三项。

categories:
  - 前端开发

tags:
  - nuxt4
  - optimization
  - treeShake
  - 编译期
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

想找"少水合一点"的开关，你在配置页里翻到 `optimization`。点进去三个键名一个都猜不出用途：`asyncTransforms`、`keyedComposables`、`treeShake`。

方向确实错了一半：**这一节是构建期的事，跟"什么时候水合"没关系**。但它值得读，因为第三个键的默认值里藏着一件你早就被要求知道的事的实现。

这一节的定性只有一句：

> Build time optimization configuration.

## asyncTransforms：过了 await 还要留着上下文

官方给的定义：

> Options passed directly to the transformer from `unctx` that preserves async context after `await`.

这句话的人话版：**异步函数里 `await` 完之后，当前上下文还在**，所以你在 `defineNuxtPlugin` 的异步逻辑里 await 之后再调组合式函数，不会拿不到东西。做这件事的是 `unctx` 的转换器，而这一节配的是**直接传给那个转换器的选项**。

两个子键。第一个是 `asyncFunctions`，类型 `array`，默认值就两个名字：

```json
[
  "defineNuxtPlugin",
  "defineNuxtRouteMiddleware"
]
```

第二个是 `objectDefinitions`。它的形状是"函数名 → 要处理的属性名列表"，默认三组：

| 函数 | 默认列表 |
| --- | --- |
| `defineNuxtComponent` | `["asyncData", "setup"]` |
| `defineNuxtPlugin` | `["setup"]` |
| `definePageMeta` | `["middleware", "validate"]` |

读法是这样：`definePageMeta` 里的 `middleware` 和 `validate` 这两个属性，是会被这个转换器处理的对象成员。官方在这里只给了名字和默认值，**没有解释每个键改了会怎样**，也没有给使用建议。

## keyedComposables：多出来的那个参数

这段说明是四个键里写得最完整的，逐句都在讲规则：

> Functions to inject a key for.
> As long as the number of arguments passed to the function is less than `argumentLength`, an additional magic string will be injected as the last argument. This key is stable between SSR and client-side hydration. You will need to take steps to handle this additional key.
> The key is unique based on the location of the function being invoked within the file.

四件事：

- 传参个数**少于** `argumentLength` 时，才会在**最后一个参数位**注入一个 magic string。
- 这个 key 在服务端渲染与客户端水合之间是**稳定的**（同一处调用两边算出同一个 key）。
- **你要自己处理这个多出来的参数**——官方原话就是这么一句提醒，没有代劳方案。
- key 的唯一性来自**函数在文件里被调用的位置**。

默认登记了七个函数，每项三个字段（`name`、`argumentLength`、`source`）：

| name | argumentLength | source |
| --- | --- | --- |
| `callOnce` | 3 | `#app/composables/once` |
| `defineNuxtComponent` | 2 | `#app/composables/component` |
| `useState` | 2 | `#app/composables/state` |
| `useFetch` | 3 | `#app/composables/fetch` |
| `useAsyncData` | 3 | `#app/composables/asyncData` |
| `useLazyAsyncData` | 3 | `#app/composables/asyncData` |
| `useLazyFetch` | 3 | `#app/composables/fetch` |

这张表对你写代码的实际含义只有一个：`useFetch('/api/x')` 只传了一个参数，按规则它会被补上第二个和第三个位子的东西——所以**别按"函数原本签名有几个参数"去推断运行时收到了几个**。要包一层自己的函数转发给这些组合式函数时尤其注意，这一条就是"You will need to take steps to handle this additional key"指的场景。

## treeShake：默认清单解释了那条老规则

`treeShake` 下只有一个 `composables`，说明是 "Tree shake composables from the server or client builds."——**从服务端或客户端的包里摇掉指定的组合式函数**。官方的例子中把 `onMounted` 从客户端摇掉（`client: { vue: ['onMounted'] }`）、把 `onServerPrefetch` 从服务端摇掉。

默认值比例子信息量大得多，两份清单列全：

| 来源 | 从客户端包摇掉 | 从服务端包摇掉 |
| --- | --- | --- |
| `vue` | `onRenderTracked`、`onRenderTriggered`、`onServerPrefetch` | `onMounted`、`onUpdated`、`onUnmounted`、`onBeforeMount`、`onBeforeUpdate`、`onBeforeUnmount`、`onRenderTracked`、`onRenderTriggered`、`onActivated`、`onDeactivated` |
| `#app` | `definePayloadReducer`、`definePageMeta`、`onPrehydrate` | `definePayloadReviver`、`definePageMeta` |

三件事可以从这两列里直接读出来：

**服务端那一列是十个生命周期钩子**，全是浏览器侧的挂载/更新/卸载钩子。这就是"服务端不会执行 `onMounted` 这类钩子"这条规则的**实现层解释**——不是运行时给你跳过，而是这些名字在服务端包里根本不出现。

**`onPrehydrate` 和 `definePayloadReducer` 只在客户端被摇掉**，`definePayloadReviver` 只在服务端被摇掉。写入侧和读取侧的分工就体现在这两行的不对称上（`onPrehydrate` 只在水合前那一刻执行，见 438 那篇讲的 E1006 就是它没被转换时报的错）。

**`definePageMeta` 两列都有**，也就是说它在客户端和服务端都被摇掉。这一条看着反直觉，但它属于默认值，改动前先确认你的项目里有没有真的依赖它。

## 自查

- 这一节三个键全是构建期配置，没有水合时机开关；控制水合时机的是 `hydrate-*` 那类组件属性（378 那篇）。
- `asyncTransforms` 默认两个函数 + 三组对象定义，官方未解释改动后果。
- `keyedComposables` 默认七个函数，规则是"参数少于 `argumentLength` 就补一个末位 magic string"，且**要你自己处理**；key 按调用位置生成、SSR 与水合之间稳定。
- `treeShake.composables` 默认清单里，服务端摇掉十个 Vue 生命周期钩子，客户端摇掉 `onRenderTracked`、`onRenderTriggered`、`onServerPrefetch` 加 `#app` 的三项。

## 下一篇

下一篇 442 进入真正能改加载行为的预取：`<NuxtLink>` 默认按可见性预取，官方同时写明了它的代价——两种触发器都开会重复取同一份资源，而浏览器忙、断网或只有 2g 时它会自己跳过。
