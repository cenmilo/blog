---
url: /posts/ff3c2bcf774dfd75811d2878fe089e31/
title: 283 - useRuntimeHook：在组件里订阅运行时钩子并自动清理
date: 2027-01-14T10:00:00+08:00
updated: 2027-01-14T10:00:00+08:00
author: cmdragon

summary:
  摘要：275 里订阅钩子用的是插件的 nuxtApp.hook，那份订阅不随组件卸载而消失。useRuntimeHook 是给组件用的版本——在 setup 里 useRuntimeHook(name, fn) 注册一个运行时钩子，组件作用域销毁时它会自动注销，不用你手动清理。这一篇讲它的签名（无返回值）、v3.14+ 的版本门槛、和 nuxtApp.hook 的区别（自动 dispose），并把官方那个 link:prefetch 示例落地。钩子名仍是 275 那张 App Hooks (Runtime) 表里的名字。

categories:
  - 前端开发

tags:
  - nuxt4
  - useRuntimeHook
  - 钩子
  - 组件
  - 清理
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

275 讲的是 `nuxtApp.hook(name, cb)`：在插件里订阅钩子，那份订阅会活在整个应用生命周期里。但有些钩子你只想在**某个组件还在屏上时**响应——组件被卸载了，这个监听就该跟着没了。`useRuntimeHook` 就是干这个的，重点是它**自动帮你清理**。

## 它是什么

官方定位句把两个卖点都写进去了：

> Registers a runtime hook in a Nuxt application and ensures it is properly disposed of when the scope is destroyed.

也就是：注册一个运行时钩子监听 + 作用域销毁时确保它被正确注销。版本门槛另有一条 important：

> This composable is available in Nuxt v3.14+.

## 签名与参数

```ts
function useRuntimeHook<THookName extends keyof RuntimeNuxtHooks> (
  name: THookName,
  fn: RuntimeNuxtHooks[THookName] extends HookCallback ? RuntimeNuxtHooks[THookName] : never,
): void
```

参数说明官方给了两条：

- `name`：要注册的运行时钩子的名字。官方直接让你去看那份清单——"You can see the full list of runtime Nuxt hooks here"，链接指向 `api/advanced/hooks#app-hooks-runtime`（就是 275 那张 App Hooks (Runtime) 表）。
- `fn`：钩子触发时要执行的回调。官方特别注明 **"The function signature varies based on the hook name."**——回调的参数随钩子不同而不同（`vue:error` 是 `err, target, info`，`link:prefetch` 是那个 link 对象，等等）。

返回值最关键：

> The composable doesn't return a value, but it automatically unregisters the hook when the component's scope is destroyed.

无返回值（`: void`），并且它在**组件作用域销毁时自动注销**这个钩子。这就是它和 `nuxtApp.hook` 的核心差别——`nuxtApp.hook` 的订阅是应用级的、不随组件消失；`useRuntimeHook` 是组件级的、卸载即清。

## 官方示例：监听链接预取，组件一卸载就不再打日志

```vue [pages/index.vue]
<script setup lang="ts">
// 注册一个每次有链接被预取就执行的钩子，但它会在组件卸载时
// 被自动清理（不会再被调用）
useRuntimeHook('link:prefetch', (link) => {
  console.log('Prefetching', link)
})
</script>
```

这个例子把三件事演示齐了：在页面/组件的 `<script setup>` 里调用；钩子名 `'link:prefetch'`（是那张表里、客户端侧的一个钩子，"Called when a `<NuxtLink>` is observed to be prefetched"）；回调拿到的 `link` 参数由钩子本身决定。

## 什么时候用 useRuntimeHook，什么时候用 nuxtApp.hook

把两个选择摆一起：

| | `nuxtApp.hook`（275） | `useRuntimeHook`（本篇） |
| --- | --- | --- |
| 用在哪 | 主要插件（也可组合式） | 组件/页面的 setup |
| 谁触发注销 | 不随组件消失，应用级常驻 | 组件作用域销毁时自动注销 |
| 返回值 | — | 无（`: void`） |
| 钩子名范围 | App Hooks (Runtime) | 同一张表（`name` 要 `keyof RuntimeNuxtHooks`） |

判据很简单：这个监听的生命周期**该不该等于某个组件**？该 → `useRuntimeHook`（组件没了它自动走，省得你 `onUnmounted` 里手动 off）；不该、它属于全局/插件级 → `nuxtApp.hook`。

这也解释了为什么官方给的是 `: void`：你不需要拿到一个"注销函数"去手动调——作用域销毁就是注销时机。

## 自查

- 在组件里想订阅运行时钩子、并希望卸载时自动清理 → `useRuntimeHook(name, fn)`，不用自己写 dispose。
- 需要的是应用级常驻监听（比如全局错误上报） → 还是插件里的 `nuxtApp.hook`。
- `name` 必须是 App Hooks (Runtime) 表里的钩子名；`fn` 的参数形状跟着钩子走。
- v3.14+ 才有；更早版本用不了。

## 下一篇

下一篇 284 讲怎么定义自己的钩子与事件：用 `hook` 订阅、`callHook` 触发，以及通过扩展 `RuntimeNuxtHooks` 等接口给自己起的钩子名补类型。

## 本文依据

- useRuntimeHook（`api/composables/use-runtime-hook`）：定位句 "Registers a runtime hook in a Nuxt application and ensures it is properly disposed of when the scope is destroyed."；important "available in Nuxt v3.14+."；签名 `useRuntimeHook(name, fn): void`；参数 `name`（链接 runtime hooks 清单）与 `fn`（"The function signature varies based on the hook name."）；Return Values "doesn't return a value, but it automatically unregisters the hook when the component's scope is destroyed."；`pages/index.vue` 的 `useRuntimeHook('link:prefetch', (link) => ...)` 示例。
- Lifecycle Hooks（`api/advanced/hooks`）：App Hooks (Runtime) 表；`link:prefetch` 描述 "Called when a `<NuxtLink>` is observed to be prefetched."。
