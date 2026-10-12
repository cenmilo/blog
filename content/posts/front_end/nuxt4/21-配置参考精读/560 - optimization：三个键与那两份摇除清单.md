---
url: /posts/b638e722bcbf0bb0b562b11a6ab9bb7f/
title: optimization：三个键与那两份摇除清单
date: 2027-04-16T11:00:00+08:00
updated: 2027-04-16T11:00:00+08:00
author: cmdragon

summary:
  摘要：`optimization` 的定性是一句 **"Build time optimization configuration."**，下面**只有三个键**：`asyncTransforms`、`keyedComposables`、`treeShake`。`asyncTransforms` 传给 unctx 的转换器，默认把 `defineNuxtPlugin` 与 `defineNuxtRouteMiddleware` 当 async 函数、`objectDefinitions` 三组分别对应 `defineNuxtComponent`（`["asyncData","setup"]`）、`defineNuxtPlugin`（`["setup"]`）、`definePageMeta`（`["middleware","validate"]`）。`keyedComposables` 默认七项，各带 `argumentLength` 与 `source`，规则是**实参数少于 `argumentLength` 时把一段"魔法字符串"作为最后一个参数注入**、键在 SSR 与水合之间稳定、且**"你得自己处理这个多出来的键"**。`treeShake.composables` 才是"服务端不跑 `onMounted`"的实现出处：**server 那份默认摇掉十个 Vue 生命周期钩子**（`onMounted`、`onUpdated`、`onUnmounted`、`onBeforeMount`、`onBeforeUpdate`、`onBeforeUnmount`、`onRenderTracked`、`onRenderTriggered`、`onActivated`、`onDeactivated`）加 `#app` 的 `definePayloadReviver`/`definePageMeta`；**client 那份摇掉** `onRenderTracked`/`onRenderTriggered`/`onServerPrefetch` 与 `definePayloadReducer`/`definePageMeta`/`onPrehydrate`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 构建优化
  - tree-shaking
  - keyed-composables
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

文档反复说"服务端不会跑 `onMounted`"。这句话的技术出处不在渲染章节，而在一个只有三个键的配置节里。

`optimization` 就是那处。

## 一、asyncTransforms：谁被当成异步函数

`### asyncTransforms` 的说明是 "Options passed directly to the transformer from `unctx` that preserves async context after `await`."——**它管的是"`await` 之后上下文还在不在"这件事**（这正是 503 那类 E1001 报错的机制层）。

两个子项：

| 子项 | 类型 | 默认 |
| --- | --- | --- |
| `asyncFunctions` | `array` | `["defineNuxtPlugin", "defineNuxtRouteMiddleware"]` |
| `objectDefinitions` | — | 三组 |

`objectDefinitions` 下的三组逐字如下：

| 组 | 默认值 |
| --- | --- |
| `defineNuxtComponent` | `["asyncData", "setup"]` |
| `defineNuxtPlugin` | `["setup"]` |
| `definePageMeta` | `["middleware", "validate"]` |

读法：**这三个键是"对象形态定义的入口名"，数组值是这个对象里哪些属性该被当异步函数处理。** 例如 `definePageMeta` 的 `middleware` 与 `validate` 可以写 `await`，靠的就是这一行。

## 二、keyedComposables：注入一个键，代价你要自己承担

`### keyedComposables` 的说明值得整段读：

> Functions to inject a key for.
>
> As long as the number of arguments passed to the function is less than `argumentLength`, an additional magic string will be injected as the last argument. This key is stable between SSR and client-side hydration. You will need to take steps to handle this additional key.
> The key is unique based on the location of the function being invoked within the file.

四句规则：**实参个数少于 `argumentLength` 才注入**；注入的是**一段魔法字符串、位置是最后一个参数**；这个键**在 SSR 与客户端水合之间稳定**；**唯一性基于调用点在文件中的位置**。

中间那句 **"You will need to take steps to handle this additional key"** 是最实用的一条：这不只是构建器内部的事，你写接收可变参数的函数时会真的多收到一个参数。

默认七项，全部照录：

| `name` | `argumentLength` | `source` |
| --- | --- | --- |
| `callOnce` | 3 | `#app/composables/once` |
| `defineNuxtComponent` | 2 | `#app/composables/component` |
| `useState` | 2 | `#app/composables/state` |
| `useFetch` | 3 | `#app/composables/fetch` |
| `useAsyncData` | 3 | `#app/composables/asyncData` |
| `useLazyAsyncData` | 3 | `#app/composables/asyncData` |
| `useLazyFetch` | 3 | `#app/composables/fetch` |

**这七项就是"为什么 `useState('x')` 少写一个参数也没事"的答案**：`useState` 的 `argumentLength` 是 2，你只传 1 个时它被补上第 2、3 个参数位（其中一个是注入的键）。

官方还留了一条 read-more 指向模块文档的 keyed functions 一节（`guide/modules/recipes-basics#add-keyed-functions`）——**要自己加一个 keyed 函数，去那里，不是在这一节改。**

## 三、treeShake：两份摇除清单

`### treeShake` 一句 "Tree shake code from specific builds."，其下 `#### composables` 是 "Tree shake composables from the server or client builds."，两个方向各一份默认清单。

**client 默认（摇掉这些）：**

```json
{
  "vue": ["onRenderTracked", "onRenderTriggered", "onServerPrefetch"],
  "#app": ["definePayloadReducer", "definePageMeta", "onPrehydrate"]
}
```

**server 默认（摇掉这些）：**

```json
{
  "vue": ["onMounted", "onUpdated", "onUnmounted", "onBeforeMount", "onBeforeUpdate", "onBeforeUnmount", "onRenderTracked", "onRenderTriggered", "onActivated", "onDeactivated"],
  "#app": ["definePayloadReviver", "definePageMeta"]
}
```

两张表并排读，四件事就落地了：

- **"服务端不执行 `onMounted` 一类钩子"不是运行时判断，而是构建期就不带这些代码**——server 那十个钩子全在摇除清单里。
- **对称方向也对**：客户端不需要 `onServerPrefetch`，它出现在 client 清单里。
- **`definePageMeta` 两端都被摇**（第 503 节讲的 E1007"宏被 transform 掉"与此呼应）。
- **payload 的两半各归一侧**：客户端摇 `definePayloadReducer`、服务端摇 `definePayloadReviver`。

官方示例给的是"改这份清单"的写法：

```ts
export default defineNuxtConfig({
  optimization: {
    treeShake: {
      composables: {
        client: { vue: ['onMounted'] },
        server: { vue: ['onServerPrefetch'] },
      },
    },
  },
})
```

**这行示例值得警惕：按 `client`/`server` 分组给的是 `{ 包名: [名字...] }` 结构，而示例里在 client 摇 `onMounted`、在 server 摇 `onServerPrefetch`——正好与默认清单的方向相反。** 也就是说：**这个示例演的是"怎么写"，不是"该这么写"**；照抄会把两端最需要的钩子摇掉（客户端没了 `onMounted`、服务端多摇一个本来就不在 server 清单里的钩子）。

**摇除之后会怎样（是构建产物变化还是静默报错），本节没写**；要改就构建一次，看产物与运行表现，而不是只看配置通过。

## 自查

- `optimization` 定性只有 "Build time optimization configuration."，三个键：`asyncTransforms`、`keyedComposables`、`treeShake`。
- `asyncFunctions` 默认两项；`objectDefinitions` 三组说明"哪些属性算异步函数"（`asyncData`/`setup`、`setup`、`middleware`/`validate`）。
- `keyedComposables` 七项各带 `argumentLength` 与 `source`；注入条件是实参更少、注入位置是最后一个参数、键在 SSR 与水合之间稳定、**多出来的键要你自己处理**；加自定义 keyed 函数去看模块那节。
- `treeShake.composables` 两份清单就是"服务端不跑 `onMounted`"的技术出处；`definePageMeta` 两端都被摇。
- 官方示例的方向与默认清单相反，只演示写法不推荐做法；改后要构建验证。

## 下一篇

下一篇 561 单读 `buildId` 作为配置项能做什么：定义只有一句（may contain 项目状态哈希）、默认那串 UUID 是样例、能不能自己设以及设了会怎样文档都没写。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## optimization` 全节：定性句；`### asyncTransforms` 的 unctx 说明句、`asyncFunctions` 两项默认与 `objectDefinitions` 三组默认；`### keyedComposables` 的四句规则、七项默认 JSON 全文、指向模块文档 keyed functions 的 read-more；`### treeShake` 与 `#### composables` 的两句说明、client 与 server 两份默认 JSON 全文、那条 `client: {vue:['onMounted']}` 示例）
- https://nuxt.com/docs/4.x/errors/e1007 与 https://nuxt.com/docs/4.x/errors/e1001（宏被构建 transform 掉、`await` 后丢上下文，与本节两个键的机制对应）
