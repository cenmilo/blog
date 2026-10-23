---
url: /posts/32bf88f5ae6aa1e0df3dca2cf29d9e79/
title: 810 - 错误边界（4）：error 事件，把边界里红过一次变成可读的数
date: 2026-10-31T12:00:00+08:00
updated: 2026-10-31T12:00:00+08:00
author: cmdragon

summary:
  摘要：边界接住一次抛错之后，页面上只有那一小块换了内容，控制台一条都不留。要把"红过"送进监控，官方给了三条口子：`@error` 事件、`useTemplateRef` 上读到的 `error` 与 `clearError()`、顶层 `vue:error` 钩子。这一站把三条同时接上，读它们各被叫到哪一次、按什么顺序，以及一个容易踩的时机差——在 `@error` 里读那位 `error` 还读到 null，隔零点几毫秒到钩子那一拍已经是错误对象，而错误槽也已经渲完了。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 错误边界
  - 错误处理
  - 会员区
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上是这一块红了，控制台里什么都没有

上一站结束时的样子：`http://localhost:3100/docs/getting-started?islandLab=boundary` 点`在边界里挂上那一位会炸的组件`之后，那块位置换成错误块，上面三行读数变成

```text
边界里红过：1 次
边界收到的错：Error: LAB_BOOM_807：渲染期抛错（没有边界包住它）
错误形状：{"是Error实例":true,"名字":"Error",…,"可枚举键":[]}
```

而打开 Console 面板，一条错误都没有。上一站读过这个形状，这一站要解决的是它带来的实际问题：**既然界面之外没人知道这里红过，那要往监控里送，该挂在哪一条口子上？**

官方一共给了三条。这一站把它们同时接在同一个边界上，读它们各被叫到哪一次。

## 三条口子分别写在哪儿

讲 `<NuxtErrorBoundary>` 那一节给前两条。Events 段：「`@error`: Event emitted when the default slot of the component throws an error」。再往下一段标题是「Accessing `error` and `clearError` in Script」，给的写法是 `useTemplateRef` 拿组件实例，然后读 `errorBoundary.value?.error`、调 `errorBoundary.value?.clearError()`。

第三条在讲错误处理那一节的 Vue Errors 段：「Nuxt provides a `vue:error` hook that will be called if any errors propagate up to the top level」，后面的提示块补了一句「`vue:error` 是基于 `onErrorCaptured` 那一位生命周期钩子的」。同段还写了第四条口子（`vueApp.config.errorHandler`，收所有 Vue 错误、包括已经被处理过的）——这一站没开它，原因放在最后一条。

## 仓库里加了什么

`app/pages/docs/[slug].vue` 那一档，边界上多挂一个 `ref`，页面里再注册顶层钩子：

```ts
const 边界引用 = useTemplateRef<{ error: unknown, clearError: () => void }>('errorBoundary')
const 门顺序 = ref<string[]>([])
const 钩子次数 = ref(0)
const 各口子那一刻 = ref('')
const 应用 = useNuxtApp()

function 记边界报错 (错误: unknown) {
  门顺序.value = [...门顺序.value, '@error']
  各口子那一刻.value += `@error 里读到：${读ref形状(边界引用.value?.error)}｜`
  事件时刻.value = performance.now()
}

function vue错误钩子 (错误: unknown) {
  钩子次数.value += 1
  门顺序.value = [...门顺序.value, 'vue:error']
  各口子那一刻.value += `vue:error 里读到：${读ref形状(边界引用.value?.error)}｜`
}

if (import.meta.client) {
  应用.hook('vue:error', vue错误钩子)
  onBeforeUnmount(() => 应用.hooks.removeHook('vue:error', vue错误钩子))
}
```

两处写法值得停一下。**钩子要在离开这一页时摘掉**：`nuxtApp.hook()` 注册的是全站共享的那一位，不摘就会把后面别的页面红过的次数算到这一档头上。**读 `useTemplateRef` 那一位要自己判形状**：组件 `defineExpose` 出来的是内部那个 `shallowRef`，页面里读的时候类型上写的是包装、运行时读到的是解包后的值——所以 `读ref形状()` 那种"先问一句它到底是什么"的写法在这里不是多余。

判据还是 `test/e2e/error-boundary.spec.ts`，这一站加三条（档十、档十一、档十二），整档十二条一起跑 `12 passed`。

## 读数，原样打印

```text
[边界·三条口子] {"顺序":" 三条口子被叫到的顺序：@error → vue:error","钩子":" vue:error 钩子被叫到：1 次（最近一条：Error: LAB_BOOM_807：渲染期抛错（没有边界包住它））","那一刻":" 各口子那一刻从 ref 读到：@error 里读到：null｜vue:error 里读到：Error 对象（…）｜","微任务里":" 把同一句读法挪到微任务里：Error 对象（…）","注册者":" 这一位钩子上的注册者：1 个注册者｜那一刻错误槽在不在：1｜距 @error 0.70 毫秒","边界计数":" 边界里红过：1 次 "}

[边界·用ref清] {"计数":" 边界里红过：2 次 ","顺序":" 三条口子被叫到的顺序：@error → vue:error → ref.clearError() → @error → vue:error","钩子次数":" vue:error 钩子被叫到：2 次 ","错误槽":1,"探针":0}

[边界·外面] {"处理器跑到":" 处理器被跑到几次：1","有边界位吗":0,"钩子":" 这一趟 vue:error 被叫到：1 次 ","错误页位":0,"控制台条数":1}

[边界·普通对象] {"红过":" 第二个边界里红过：1 次 ","那一位ref":" 各口子那一刻从 ref 读到：vue:error 里读到：null｜","整页错误页":0}
```

## 一次抛错把前两条按顺序叫到，第三条是全站共用的

`@error` 先响、`vue:error` 后响，顺序串读出来就是 `@error → vue:error`。这与包内那三行的顺序一致（`node_modules/nuxt/dist/app/components/nuxt-error-boundary.vue:33-38`：先 `emit("error", err)`，再 `callHook("vue:error", …)`，最后才 `error.value = err`）。每红一次，两条各被叫到一遍——档十一里从脚本调用 `clearError()` 又换来一次新的抛错，顺序串接着长，钩子从 1 次变 2 次。

**这里有个时机差，是最容易写错的一处。** 同一句读法 `边界引用.value?.error`，在 `@error` 里读到的是 `null`，到 `vue:error` 那一拍读到的是错误对象；而那一刻距 `@error` 只隔了 0.70 毫秒，页面上的错误槽已经渲出来了。所以：

- 想在 `@error` 处理器里"顺便读一下边界现在存着什么"，读到的是空的——**要用的东西就在参数里**（`@error` 交出来的就是那次抛错的那个值），别去读状态；
- 顶层钩子那一拍已经晚于状态更新，读得到，但它不是只给你用的（下面那条对照会说明为什么）。

顺带纠正一处我自己的预判：本来以为"钩子被推到赋值之后"是因为排在前面的注册者回了 Promise（钩子是按注册顺序串行叫的，`hookable/dist/index.mjs:31-38` 里只要前面回了 thenable，后面的就被推走）。读数打回来的是 `1 个注册者`——这个解释被数据否掉了。中间那一拍到底是什么，这一站没钉到底，写在末尾的未闭合里。

## 脚本里那条 `clearError()` 与槽里那颗按钮是同一条路

`[边界·用ref清]` 那一行读数与上一站槽里那颗按钮的结果一模一样：`计数` 从 1 次变 2 次、错误槽重新出现（`错误槽:1`、`探针:0`）。原因写在包内那一位 `defineExpose({ error, clearError })`（`:48-51`）——暴露出来的就是内部同一个函数，不是第二个"也能清一下"的东西。所以两条路都会踩上上一站那条规矩：**原因没改掉，清一次就是再红一次**。

另一条对照更值钱。这一档右边还有第二个边界（上一站抛普通对象那一位）。它红的时候，第一个边界那一位 `error` 读回来还是 `null`：`[边界·普通对象]` 那行 `那一位ref` 就是这个形状。⇒ `useTemplateRef` 读的是**这一个实例自己的状态**，不是"全站最近一次错误"。页面上有几块各自包了边界，就得各拿各的引用，别指望一位通读。

## 边界外那一次：只剩顶层这一条口子

`[边界·外面]` 走的是 807 那档（`?islandLab=err`，事件处理器里抛普通错）。这一趟读回来四件事：`有边界位吗: 0`（这一档页面上根本没有边界，`@error` 那条口子不存在）、`处理器跑到: 1`、`vue:error 被叫到：1 次`、`控制台条数: 1`。

把两档并排放，就是这一站的结论：

| 抛错的位置 | `@error` | `vue:error` | 控制台 |
| --- | --- | --- | --- |
| 边界里（默认槽） | 响，早于状态更新 | 响，晚于状态更新 | 不响（上一站读过 0 条） |
| 边界外 | 根本没有这一条 | 响 | 响一行 |

**要做全站上报，只能挂 `vue:error`**：它是两种位置都路过的那一条（边界自己那一句 `callHook` 加根组件 `nuxt-root.vue:71-72` 那一句，两处都叫它）。`@error` 只看得见自己那一块，用它做监控要在每个边界上各接一遍；用它做"这块红过、给用户换个说法"才正好。

## 这一站故意没开的那一条口子

讲错误处理那一节还写了 `vueApp.config.errorHandler`，并强调它「会收到所有 Vue 错误，即使已经被处理过」。这一站没接，因为它是**替换**：一旦你写上自己的处理器，Vue 默认那一路就不再走，而 807、808 两站读到的"控制台里那一行错误"正是默认那一路发出来的。拿它做上报是对的，但要连着把默认行为补回去，否则这条线上前面几站的读数会一起变味——那是这一站没法顺手做的事。

## 排错表

| 想知道的 | 读哪一位 | 只读一处会错在哪 |
| --- | --- | --- |
| 三条口子各响了几次 | 顺序串 ＋ 钩子次数 | 只看计数会分不清是哪一条口子给的 |
| 能不能在 `@error` 里读边界状态 | 那一刻那一位读到 null 还是对象 | 不读会写出"参数没给就用状态"的代码 |
| 脚本里的清错和按钮一样吗 | 计数有没有再翻一次 | 只看错误槽还在，会以为没生效 |
| 多位边界会不会串 | 第二个边界红时第一个那一位 | 不看会以为读的是全站最近一次错误 |
| 上报该挂哪条 | 边界外那一趟里两条口子各响没响 | 只挂 `@error` 会漏掉所有没包边界的错 |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/error-boundary.spec.ts
```

预期 `12 passed`。手点版：进 `?islandLab=boundary`，开 Console 面板，点`在边界里挂上那一位会炸的组件`，读那四行新加的输出——顺序、钩子次数、那一刻从 ref 读到什么、注册者与毫秒差；再点`用 ref 上的 clearError() 清一次`，看顺序串往后长两截。然后进 `?islandLab=err` 点`事件里普通抛错`：这回顺序那行根本没有，钩子那行响一次，而 Console 里多出一整行——这就是两条口子各自能看见的范围。

## 下一篇

错误边界（5）：服务端渲染那一路它接不住，换路由会自动清错——同一位探针在 SSR 与 CSR 两条路上各抛一次，读回的是整屏错误页还是 `#error` 槽；再验那句"If you navigate to another route, the error will be cleared automatically"。

## 本文依据

- 官方：讲 `<NuxtErrorBoundary>` 那一节（Events 段的 `@error`、Slots 段的 `#error`、Accessing `error` and `clearError` in Script 那段的 `useTemplateRef` 写法）；讲错误处理那一节的 Vue Errors 段（`vue:error` 那两句、提示块那句"基于 `onErrorCaptured`"、`vueApp.config.errorHandler` 那句）
- 包内源码：`node_modules/nuxt/dist/app/components/nuxt-error-boundary.vue`（`:33-38` 三条口子的发起顺序、`:48-51` `defineExpose({ error, clearError })`）、`node_modules/nuxt/dist/app/components/nuxt-root.vue:71-72`（根组件那一句 `callHook('vue:error')`）、`node_modules/hookable/dist/index.mjs:31-38`（钩子按注册顺序串行叫、前面回了 thenable 就把后面推走）
- 仓库：`app/pages/docs/[slug].vue`（`ref="errorBoundary"`、`门顺序`、`钩子次数`、`各口子那一刻`、那颗读 ref 与那颗用 ref 清、`err` 档新增的钩子次数行）、`test/e2e/error-boundary.spec.ts`（新增 3 条，整档 12 条）
- 上一批：[809 - 错误边界（3）：error 槽里那两样东西，显示与回去](/posts/de847f25f637a01d9f4f6ce7449e27d8/)
