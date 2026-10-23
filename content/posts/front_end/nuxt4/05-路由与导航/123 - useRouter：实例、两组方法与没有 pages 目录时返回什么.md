---
url: /posts/c30fd36a6469b1036cb564eb077c74eb/
title: 123 - useRouter：实例、两组方法与没有 pages 目录时返回什么
date: 2026-11-22T09:00:00+08:00
updated: 2026-11-22T09:00:00+08:00
author: cmdragon

summary:
  摘要：`useRouter` 的官方定义是"返回 router 实例"，用法是 `const router = useRouter()`；如果只需要在模板里用实例，官方给的是 **`$router`**（示例是 `<button @click="$router.back()">Back</button>`）。这一篇要划清三件事。**第一，它与 vue-router 的关系取决于有没有 pages 目录**：官方原话是"**如果你有一个 `app/pages/` 目录，`useRouter` 与 `vue-router` 提供的在行为上完全相同。**"而反过来——"**如果你没有 `app/pages/` 文件夹，那么 `useRouter` 将返回一个通用（universal）router 实例，带有类似的 helper 方法，但要注意，并非所有特性都被支持，或以与 `vue-router` 完全相同的方式表现。**"**第二，方法分两组，用途完全不同**：一组是**改路由表**——`addRoute()`（"`parentName` 可以提供，用来把新路由作为既有路由的子路由加入"）、`removeRoute()`（按名字移除一条既有路由）、`getRoutes()`（"获取所有路由记录的完整列表"）、`hasRoute()`（"检查是否具有给定名字的路由"）、`resolve()`（"返回一个路由位置的规范化版本，还包含一个带上任何已有 base 的 `href` 属性"）；另一组是**基于 History API 动位置**——`back()`（"若可能则在历史里后退，等同于 `router.go(-1)`"）、`forward()`（等同于 `router.go(1)`）、`go()`（"在历史里前进或后退，没有 `router.back()` 与 `router.forward()` 所强制的层级限制"）、`push()`、`replace()`。**第三，后两个官方都写了替代方案**：`push()` 的说明末尾是"**建议改用 `navigateTo`。**"，`replace()` 同样写着"**建议改用 `navigateTo`。**"，而 `replace()` 的定义本身是"通过替换当前历史栈条目来程序化地导航到一个新 URL"。官方在 addRoute 与 push 之间给了一条极有用的区分："**`router.addRoute()` 把路由细节添加进一个路由数组，它在构建 Nuxt 插件时很有用；而 `router.push()` 另一方面会立即触发一次新的导航，它在页面、Vue 组件与组合式函数里很有用。**"此外这个实例还提供 `afterEach`、`beforeEach`、`beforeResolve` 三个导航守卫辅助方法，但官方紧接着说"**Nuxt 有一个叫路由中间件（route middleware）的概念，它简化了导航守卫的实现并提供了更好的开发体验**"；Promise 与错误处理方面是 `isReady()`（"返回一个在 router 完成初始导航时 resolve 的 Promise"）与 `onError`（"添加一个在导航期间每次发生未捕获错误时被调用的错误处理器"）。

categories:
  - 前端开发

tags:
  - nuxt4
  - userouter
  - 路由
  - history-api
  - 导航

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一篇是"我在哪"（`useRoute`），这一篇是"方向盘"（`useRouter`）。方向盘上有两类按钮：一类改地图，一类改位置——而官方对第二类里的两个按钮明确说了：别按，用另一个函数。

## 取值与模板入口

> `useRouter` 组合式函数返回 router 实例。

```vue [app/pages/index.vue]
<script setup lang="ts">
const router = useRouter()
</script>
```

只在模板里需要实例时，官方给的是 `$router`：

```vue [app/pages/index.vue]
<template>
  <button @click="$router.back()">
    Back
  </button>
</template>
```

这与 121 的 `route` / `$route` 是同一套设计：脚本里用组合式函数，模板里用 `$` 前缀。

## 第一件事：有没有 `app/pages/`，决定它是不是"真"的 router

官方两处说法必须成对记：

| 情形 | 官方原话 |
| --- | --- |
| **有** `app/pages/` 目录 | "如果你有 `app/pages/` 目录，`useRouter` 与 `vue-router` 提供的**在行为上完全相同**。" |
| **没有** `app/pages/` 目录 | "如果你没有 `app/pages/` 文件夹，那么 `useRouter` 将**返回一个通用（universal）router 实例**，带有类似的 helper 方法，**但要注意，并非所有特性都被支持，或以与 `vue-router` 完全相同的方式表现。**" |

这一条与 090/104 串成一线：**没有 pages 目录时 Nuxt 连 vue-router 依赖都不引入**，所以那时你拿到的实例是 Nuxt 自己提供的替身——方法名像，行为不保证一致。

所以判据很简单：**要依赖某个 vue-router 特性的精确行为，先确认项目里有 `app/pages/`。**（057 还提过另一种强制方式：`pages: true` 或 `router.options.ts`。）

## 第二件事：两组方法，一组改地图、一组改位置

### 改地图：路由表操作

官方逐项说明：

| 方法 | 官方说明 |
| --- | --- |
| `addRoute()` | 向 router 实例添加一条新路由。**可以提供 `parentName`，把新路由作为某条既有路由的子路由加入。** |
| `removeRoute()` | 按其名字移除一条既有路由。 |
| `getRoutes()` | 获取所有路由记录的完整列表。 |
| `hasRoute()` | 检查是否具有给定名字的路由。 |
| `resolve()` | 返回一个路由位置的**规范化版本**，**还包含一个 `href` 属性，它会带上任何已有的 base。** |

```ts [Example]
const router = useRouter()

router.addRoute({ name: 'home', path: '/home', component: Home })
router.removeRoute('home')
router.getRoutes()
router.hasRoute('home')
router.resolve({ name: 'home' })
```

`resolve()` 这一格值得单独看：它是**不导航**就能拿到"这条位置最终会变成什么 URL"的方法，且带 base——多路径部署（`app.baseURL` 不是 `/`）时，用它拼 `href` 比自己拼字符串可靠。

### 改位置：History API

| 方法 | 官方说明 |
| --- | --- |
| `back()` | 若可能则在历史里后退，**等同于 `router.go(-1)`**。 |
| `forward()` | 若可能则在历史里前进，**等同于 `router.go(1)`**。 |
| `go()` | 在历史里前进或后退，**没有 `router.back()` 与 `router.forward()` 所强制的层级限制**。 |
| `push()` | 通过在历史栈里压入一个条目来程序化地导航到一个新 URL。**建议改用 `navigateTo`。** |
| `replace()` | 通过替换历史栈里的当前条目来程序化地导航到一个新 URL。**建议改用 `navigateTo`。** |

```ts [Example]
const router = useRouter()

router.back()
router.forward()
router.go(3)
router.push({ path: '/home' })
router.replace({ hash: '#bio' })
```

`go()` 与 `back()` 的差别官方写得很清楚：`back`/`forward` 有层级限制，`go(n)` 没有——所以"退三步"这种跨越多条记录的移动只能用 `go()`。

## 关键的一条区分：改地图 vs 立刻导航

官方在 addRoute 示例后面挂了这条 note，它比方法表更有用：

> **`router.addRoute()` 把路由细节添加进一个数组，它在构建 Nuxt 插件时很有用；而 `router.push()` 另一方面会立即触发一次新的导航，它在页面、Vue 组件与组合式函数里很有用。**

| 你想做的事 | 属于哪组 | 用哪个 |
| --- | --- | --- |
| 运行时多出一条路由（比如按后台配置生成页面） | 改地图 | `router.addRoute()` / `removeRoute()`，官方点名场景是"构建插件时" |
| 看现在有哪些路由记录 | 读地图 | `getRoutes()` / `hasRoute('home')` |
| 知道某个位置会变成什么 URL | 读地图 | `resolve()`（拿到规范化位置 + 带 base 的 `href`） |
| 回到上一页、前进、跳 n 步 | 动位置 | `back()` / `forward()` / `go(n)` |
| **去某个新地址** | 动位置 | **`navigateTo`（官方对 `push`/`replace` 两处都这么建议）** |

最后一行是本篇最重要的结论：**`router.push()` 能用，但官方的推荐是 `navigateTo`。** 为什么，下一篇（124）展开——核心在于服务端那一侧它接不住。

## 守卫：能力在，但官方推荐另一条路

官方说明："`useRouter` 组合式函数提供 `afterEach`、`beforeEach` 与 `beforeResolve` 这些 helper 方法，它们充当导航守卫。"

紧接着就是转向建议：**"然而，Nuxt 有一个叫路由中间件的概念，它简化了导航守卫的实现并提供了更好的开发体验。"**

也就是说这三者是可用的底层入口，但**项目里做"导航前逻辑"的正解是路由中间件**（128 起整段展开；132 会把"中间件的返回值就是指令"讲透）。

## 完成度与错误：`isReady()` 与 `onError`

| 方法 | 官方说明 |
| --- | --- |
| `isReady()` | 返回一个 Promise，**在 router 完成初始导航时 resolve**。 |
| `onError` | 添加一个错误处理器，**在导航期间每次发生未被捕获的错误时被调用**。 |

这两格合起来是"等路由稳下来"和"导航期间漏掉的错误兜住"，与 122 的取数 `cause`、132 的中间件返回值是三套不同的观察面。

## 自查

1. 有 `app/pages/` 与没有它，`useRouter()` 返回的东西差别是什么？官方的原话分别怎么说？
2. `addRoute()` 与 `push()` 的差别，官方用哪句话区分？各自适合写在哪？
3. 不导航就能拿到"这条位置最终变成什么 URL、且带上 base"的方法是哪个？
4. `back()` 与 `go(n)` 的差别官方怎么描述？
5. 官方对 `push()` 与 `replace()` 都写了哪一句建议？
6. `useRouter` 提供哪三个导航守卫方法？官方推荐项目里用哪个机制替代？
7. 想在导航完成后做事、以及兜住导航期间的未捕获错误，分别用哪个方法？

## 下一篇

124 讲 `navigateTo`：它为什么在客户端和服务端都能用、中间件里为什么必须 **return** 它的结果，以及 `<script setup>` 里 await 它之后代码为什么还会继续跑。
