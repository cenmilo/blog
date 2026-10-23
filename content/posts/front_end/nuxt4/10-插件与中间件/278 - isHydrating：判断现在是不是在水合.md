---
url: /posts/3daa5b0cb66155dd5445948ef2a8ecc2/
title: 278 - isHydrating：判断现在是不是在水合
date: 2027-01-12T11:00:00+08:00
updated: 2027-01-12T11:00:00+08:00
author: cmdragon

summary:
  摘要：水合（hydration）是浏览器接手服务端 HTML、Vue 挂上交互的那一段。nuxtApp.isHydrating 是一个布尔，专门告诉你客户端此刻是否正处于这个阶段——官方给的用法是在 onErrorCaptured 里区分"水合期的错误"和"水合完成后的错误"。这一篇讲清 isHydrating 是什么、那个官方示例的判据，再顺带把 hydration 那一节的五类成因与对策收进来：因为 isHydrating 常和水合不匹配问题一起出现，判断处于水合期本身不能替你修好不匹配，真正要改的是让服务端和客户端渲染出一致的内容。

categories:
  - 前端开发

tags:
  - nuxt4
  - isHydrating
  - 水合
  - 生命周期
  - nuxtApp
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先把术语讲平：**水合（hydration）= 浏览器拿到服务器发来的那段 HTML 后，Vue 接手它、把事件监听和组件状态挂上去，让静态 HTML 变成可交互应用的过程。** `nuxtApp.isHydrating` 就是这段过程进行中时为 `true` 的一个布尔开关。

## isHydrating 是什么

use-nuxt-app 页给的定义只有一句：

> Use `nuxtApp.isHydrating` (boolean) to check if the Nuxt app is hydrating on the client side.

三个信息藏在里面：它是布尔；它回答的是"当前是不是正在水合"；而且这个判断的意义在**客户端**（"on the client side"）——服务端不存在水合，水合是浏览器那一侧接手时才发生的事。

官方给的使用例子是错误边界里的一个判据：

```ts [app/components/nuxt-error-boundary.ts]
export default defineComponent({
  setup (_props, { slots, emit }) {
    const nuxtApp = useNuxtApp()
    onErrorCaptured((err) => {
      if (import.meta.client && !nuxtApp.isHydrating) {
        // ...
      }
    })
  },
})
```

读这段的要点：它用 `onErrorCaptured` 捕获子树错误，然后用 `import.meta.client && !nuxtApp.isHydrating` 把处理范围限定成"**在客户端、且已经水合完**"。潜台词是——**水合期间冒出来的错误，和水合完成后交互期间冒出来的错误，往往要区别对待**：前者多半是服务端/客户端渲染不一致（水合不匹配）的信号，后者才是运行期真正的业务错误。官方这个内置组件（错误边界本身）就是这么分流的。

这其实呼应了生命周期那页那句"onBeforeMount、onMounted 等钩子在 SSR 不执行"——水合期是一段时间窗，`isHydrating` 给了你在代码里探这段时间窗的手段。

## 但 isHydrating 修不了水合不匹配

这是这一篇真正想让新手记住的一点。**判断"现在是不是水合期"≠ 解决"水合出错"**。`isHydrating` 是诊断/分流用的，不是修问题的开关。真正的水合不匹配，要按 hydration 那页逐类去改。

官方那句为什么不能忽略这些警告：

> Hydration mismatches are not just warnings - they are indicators of serious problems that can break your application.

后果分两类，原文列了：性能上"Increased time to interactive ... force Vue to re-render the entire component tree"、"Users may see content flashing or unexpected layout shifts"；功能上"Event listeners may not attach properly"、"Application state can become out of sync"、"Search engines may index different content than what users actually see"。

它给的五类成因与官方对策，逐条：

| 成因 | 反例（会水合不匹配） | 官方对策 |
| --- | --- | --- |
| 服务端用了浏览器 API | `localStorage.getItem('theme')`（服务端没有 localStorage） | 换 `useCookie('theme', { default: () => 'light' })` |
| 数据两端不一致 | 模板里直接 `{{ Math.random() }}` | `const state = useState('random', () => Math.random())` |
| 按客户端状态条件渲染 | `v-if="window?.innerWidth > 768"` | 用媒体查询类名，或走客户端渲染 |
| 有副作用的第三方库 | `import.meta.client` 里 `SomeBrowserLibrary.init()` | "Initialise libraries after hydration has completed"，即放进 `onMounted` |
| 依时间变化的内容 | `new Date().getHours() < 12 ? 'Good morning' : ...` | 用 `<NuxtTime>`，或 `ClientOnly` + `onMounted` 设值 |

注意第三、四类：官方案例里连 "if (import.meta.client) { ... } 里初始化浏览器库" 都算反例——正确落点是 `onMounted`（水合完成之后）。这跟 `isHydrating` 正好对上：**要在水合完成之后才做的事，判据可以是 `onMounted`；要在水合期里区分错误，判据才是 `!isHydrating`。**

官方 In Summary 的四条守则（SSR-friendly 组合式、ClientOnly 包浏览器内容、两端数据一致、避免 setup 里的副作用挪进 `onMounted`）就是把上表收成动作。

## isHydrating 的典型用武之地

把两端场景连起来，`isHydrating` 真正用得上的地方是"同一段逻辑，水合期要跳过、之后要执行"：

- 错误处理分流：像官方错误边界那样，`!isHydrating` 才当运行期错误处理。
- 初始页中间件里跳过重复执行：233 那篇提过初始页的客户端中间件会在水合时再跑一趟，判断 `isHydrating && payload.serverRendered`（或结合 `isHydrating`）能跳过那次无意义的重跑。
- 别在水合期做的副作用（比如某些埋点、DOM 测量）：与其 `setTimeout` 乱猜，不如明确等水合完成——但更常见的正解仍是 `onMounted`。

## 自查

- `isHydrating` 是客户端布尔，回答"此刻是否在接手 HTML 挂交互"；别指望它自动修好水合不匹配。
- 有水合警告先按成因改（localStorage→useCookie、随机/时间/客户端条件→useState/onMounted/ClientOnly/NuxtTime），这是根治。
- 水合后才做的事，`onMounted` 比手动判 `isHydrating` 更直接；`isHydrating` 适合"在错误捕获/中间件里区分水合期与否"。
- 内置的 `<NuxtErrorBoundary>` 本身就是按 `!isHydrating` 分流的，可当参考实现。

## 下一篇

下一篇 279 讲 `nuxtApp.payload`：服务端算出来、发给浏览器的那份数据到底长什么样，`data` 和 `state` 两个键分别装什么，以及自定义序列化（reducer/reviver）。
