---
url: /posts/ee71d9711ed0c2cf39c2621c90a46645/
title: 231 - onPrehydrate：水合前内联进 HTML 的那段脚本
date: 2026-12-28T09:00:00+08:00
updated: 2026-12-28T09:00:00+08:00
author: cmdragon

summary:
  摘要：这是 199 那张成因表的一个官方补救口：**在水合发生之前，先在浏览器里改一次 DOM**，让服务端给的 HTML 与 Vue 接手时的期望一致。定位句是 "Use onPrehydrate to run a callback on the client immediately before Nuxt hydrates the page."（v3.12+），而那句 note 直接把两个真实模块举为例证：「This is an advanced utility and should be used with care. For example, `nuxt-time` and `@nuxtjs/color-mode` manipulate the DOM to avoid hydration mismatches.」**最容易误解的是 Usage 那一段的两侧分工**："Call `onPrehydrate` in the setup function of a Vue component (e.g., in `<script setup>`) or in a plugin. **The call itself only has an effect when made on the server and is stripped from your client build.** The callback you pass, however, is serialized and inlined into the HTML, so it runs in the browser immediately before Nuxt hydrates."——写在服务端、跑在浏览器。参数表里那三条限制是硬约束：**不能引用外部变量**、**不能依赖 Nuxt 或 Vue 上下文**（"Runs before Nuxt runtime initializes, so it should not rely on Nuxt or Vue context."）；第二条重载给了 `key`，返回那个 prehydrate id 字符串，用途官方限定为 "advanced scenarios like multiple root nodes"。底层形状它也写明了：回调在构建期被字符串化并压缩，**作为 `<script>` 内联在服务端 HTML 接近 `</body>` 处**；回调带 `el` 形参时，组件根元素会被打上 `data-prehydrate-id` 属性供脚本定位。

categories:
  - 前端开发

tags:
  - nuxt4
  - onPrehydrate
  - 水合
  - DOM
  - 主题

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说它解决的问题长什么样：深色模式网站首屏闪一下浅色，主题存在浏览器本地，服务端不知道用户偏好，只能先按默认值渲染 HTML；浏览器接手时才发现该是深色，于是改类名——**这一改就和水合要匹配的 HTML 冲突了**（199 那张表里的第一类成因）。

## 一句话定位与它的自我警告

> Use onPrehydrate to run a callback on the client immediately before Nuxt hydrates the page.

版本要求是一条 important：**"This composable is available in Nuxt v3.12+."**

note 我认为比用法更重要，因为它同时给了"要小心"和"谁在用"：

> This is an advanced utility and should be used with care. For example, [`nuxt-time`] and [`@nuxtjs/color-mode`] manipulate the DOM to avoid hydration mismatches.

被点名的两个都是官方生态里的包：`nuxt-time`（099 讲过的那个时间组件）与官方颜色模式模块。**它们的共同点是"服务端只能给占位内容，真实值只有浏览器知道"**——这正是需要在挂载前后动手改 DOM 的场景。

## 两侧分工：写在服务端，跑在浏览器

Usage 整段是这一篇的核心，我逐句拆：

> Call `onPrehydrate` in the setup function of a Vue component (e.g., in `<script setup>`) or in a plugin. The call itself only has an effect when made on the server and is stripped from your client build. The callback you pass, however, is serialized and inlined into the HTML, so it runs in the **browser** immediately before Nuxt hydrates. This means it can access browser globals like `window` and the DOM.

| 句子 | 含义 |
| --- | --- |
| 调用位置 | 组件 `setup`（含 `<script setup>`）或插件 |
| "**only has an effect when made on the server**" | 在客户端调用它没有意义，注册动作发生在服务端渲染时 |
| "**stripped from your client build**" | 这段调用不会进客户端包（和 165 讲 `<DevOnly>`/`<ClientOnly>` 的"不进构建"是同一类描述） |
| 回调"serialized and inlined into the HTML" | 你的函数被序列化成字符串塞进 HTML，**不是随 JS 包加载** |
| 执行时机 | 浏览器里、Nuxt 水合之前 |
| 因此能碰什么 | `window`、DOM（普通 `<script setup>` 顶层不能碰的那些） |

这个"写在服务端、跑在浏览器"的双面性，是我第一次读这页时最容易跳过的部分——**它看起来像一个普通 composable，其实是一个"往 HTML 里注一段脚本"的服务端注册口**。

## 参数表里那三条限制是硬约束

```ts [Signature]
export function onPrehydrate (callback: (el: HTMLElement) => void): void
export function onPrehydrate (callback: string | ((el: HTMLElement) => void), key?: string): undefined | string
```

| 参数 | 必填 | 官方描述（关键部分） |
| --- | --- | --- |
| `callback` | 是 | "A function (or stringified function) to run before Nuxt hydrates. It will be stringified and inlined in the HTML. **Should not have external dependencies or reference variables outside the callback.** Runs before Nuxt runtime initializes, **so it should not rely on Nuxt or Vue context.**" |
| `key` | 否 | "(Advanced) A unique key to identify the prehydrate script, useful for advanced scenarios like **multiple root nodes**." |

三条限制我按症状理解：

1. **不能引用外部变量**：函数要被字符串化，闭包捕获不了任何东西。写在外面再引用必然拿不到值。
2. **不能依赖 Nuxt/Vue 上下文**：它跑的时候框架还没初始化，`useRoute()`、`useState()`、组件实例全都不存在。
3. `key` 只服务于"多个根节点"那一类高级场合（返回值那一节说明它能拿到 id，用来自己设置 `data-prehydrate-id`）。

返回值两行：

- "Returns `undefined` when called with only a callback function."
- "Returns a string (the prehydrate id) when called with a callback and a key, which can be used to set or access the `data-prehydrate-id` attribute for advanced use cases."

## 官方示例：三个用法层层递进

```vue [app/app.vue]
<script setup lang="ts">
onPrehydrate(() => {
  // Runs in the browser, right before Nuxt hydrates
  console.log(window)
})

// Access the root element
onPrehydrate((el) => {
  console.log(el.outerHTML)
  // <div data-v-inspector="app.vue:15:3" data-prehydrate-id=":b3qlvSiBeH:"> Hi there </div>
})

// Advanced: access/set `data-prehydrate-id` yourself
const prehydrateId = onPrehydrate((el) => {})
</script>

<template>
  <div>
    Hi there
  </div>
</template>
```

第二个例子那条注释里的 HTML 很有价值：它把服务端渲染出来的真实属性都印出来了（`data-v-inspector` 与 `data-prehydrate-id`）。**这是我不用起项目也能看到这层机制长什么样的唯一入口。**

## 落到产物里是什么样

这一段我认为是这一页最该读的部分，因为它解释了"为什么它能在水合前跑到"：

> Under the hood, the callback is stringified and minified at build time, then inlined as a `<script>` tag in the server-rendered HTML, **just before the closing `</body>` tag**. For the example above, the rendered HTML includes something like:

```html
<div data-prehydrate-id=":b3qlvSiBeH:"> Hi there </div>
<script>(()=>{console.log(window)})()</script>
<script>document.querySelectorAll('[data-prehydrate-id*=":b3qlvSiBeH:"]').forEach(el=>{console.log(el.outerHTML)})</script>
```

> When the callback accepts an `el` parameter, the component's root element is tagged with a `data-prehydrate-id` attribute so the inlined script can find it.

三条信息：

1. **构建期就字符串化 + 压缩**，所以运行期没有框架参与。
2. **落点是 `</body>` 之前**——和 193 讲的 JSON-LD/noscript/body 位置标签是同一区域。这个位置的共同点是"每个客户端都能读到"（193 那句 "where every client can read them"）。
3. 带 `el` 形参时才会打 `data-prehydrate-id`，脚本靠 `querySelectorAll` 找元素。所以**"我写了 `(el) => ...` 但元素上没有那个属性"是不该发生的**——真发生了就去看你的回调是不是没接这个参数。

## 用与不用，我的判断

官方自己把话说在前面了（"advanced utility and should be used with care"），所以我的顺序是：

1. 主题/偏好这类"两侧不一致"，先试 199 给的常规解法：`useCookie`（官方原话 "This works on both server and client"）——**让服务端一开始就知道值，比事后改 DOM 干净**。
2. 值只有浏览器能拿到（`localStorage`、媒体查询细节、时间差），才考虑这段脚本。
3. 要改 DOM 又怕水合错配，也可以退回 `<ClientOnly>`（198/096），代价是首屏没那块内容。
4. `onPrehydrate` 的适用面是"**内容已经渲染出来了，只是属性/类名需要在水合前对齐**"——那两个被点名的模块正好都是这一类。

## 自查

1. 这个调用在服务端还是客户端"生效"？回调又在哪里执行？（答：调用只在服务端有效果、且从客户端构建中被剥掉；回调被内联进 HTML，在浏览器水合前执行。）
2. 回调为什么不能引用外部变量、不能用 `useRoute()`？（答：函数会被字符串化后内联，闭包带不过去；它跑在 Nuxt 运行时初始化之前。）
3. 官方点名的两个使用者是谁、共同点是什么？（答：`nuxt-time` 与 `@nuxtjs/color-mode`，都在水合前改 DOM 以避免不匹配。）

## 下一篇

232 - 取数在生命周期里的位置：服务端没有挂载钩子这件事
