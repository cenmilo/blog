---
url: /posts/b06cb7849e987189aefd6b3020da069b/
title: 274 - nuxtApp.provide 与 useNuxtApp 取 $xxx 的规则
date: 2027-01-11T10:00:00+08:00
updated: 2027-01-11T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：往 NuxtApp 上挂一个全局 helper，官方给了两条写法：直接调 nuxtApp.provide(name, value)，或在插件里 return { provide: {...} }。两者挂出去的键名前面都会自动加一个 $，取的时候用 useNuxtApp().$名字。这一篇把这两条写法的官方示例对齐，讲清 $ 前缀这个命名约定、provide 出去的是响应式对象时"模板里不解包"那个坑（271 见过一次，这里连上原理），以及怎么给 $xxx 补类型。最后照旧把"能组合式就别 provide"这句官方建议放上。

categories:
  - 前端开发

tags:
  - nuxt4
  - provide
  - useNuxtApp
  - nuxtApp
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

上一篇（273）的 vueApp 配的是 Vue 那一层。这一篇讲的是挂到 **NuxtApp 这一层**——让某个 helper 变成"哪里 `useNuxtApp()` 都能拿到"的全局方法。

## 写法一：直接调 nuxtApp.provide

use-nuxt-app 页对 `provide` 的说明：

> `nuxtApp` is a runtime context that you can extend using Nuxt plugins. Use the `provide` function to create Nuxt plugins to make values and helper methods available in your Nuxt application across all composables and components.
> `provide` function accepts `name` and `value` parameters.

示例：

```ts [app/plugins/hello.ts]
const nuxtApp = useNuxtApp()
nuxtApp.provide('hello', name => `Hello ${name}!`)

// 打印 "Hello name!"
console.log(nuxtApp.$hello('name'))
```

这里就把两条规则一次展示了：

- 你 `provide('hello', ...)`，挂出去的键叫 **`$hello`**——**自动加 `$` 前缀**。这是约定，官方原话紧跟在示例后面：

> As you can see in the example above, `$hello` has become the new and custom part of `nuxtApp` context and it is available in all places where `nuxtApp` is accessible.

- "在所有能访问到 `nuxtApp` 的地方"都能用 `$hello`。也就是任何 `useNuxtApp()` 拿得到实例的位置。

## 写法二：插件 return 一个 provide 对象

plugins 页与 define-nuxt-plugin 页给的更常见写法：不是在函数里调 `provide`，而是**从插件返回一个带 `provide` 键的对象**：

```ts [plugins/hello.ts]
export default defineNuxtPlugin(() => {
  return {
    provide: {
      hello: (msg: string) => `Hello ${msg}!`,
    },
  }
})
```

对象写法则把 `return` 放在 `setup` 里：

```ts [plugins/hello-object-syntax.ts]
export default defineNuxtPlugin({
  name: 'hello',
  setup () {
    return {
      provide: {
        hello: (msg: string) => `Hello ${msg}!`,
      },
    }
  },
})
```

两种写法效果一致：`provide` 对象里的键 `hello`，取出来是 `$hello`。取用侧就是这一句：

```vue [app/components/Hello.vue]
<script setup lang="ts">
const { $hello } = useNuxtApp()
</script>

<template>
  {{ $hello('world') }}
</template>
```

## 坑：provide 一个 ref/computed，模板里不解包

如果 provide 的值是响应式的（`ref` 或 `computed`），有个反直觉行为。plugins 页的 warning：

> **If your plugin provides a `ref` or `computed`, it will not be unwrapped in a component `<template>`.**
> This is due to how Vue works with refs that aren't top-level to the template.

含义：模板里的自动解包（写 `count` 不用写 `count.value`）**只对"顶层直接来自 setup 返回的 ref"生效**。`$count` 是从 `useNuxtApp()` 对象上取下来的、不是模板作用域的顶层 ref，所以不解包。你 provide 出去一个 `ref(0)`，模板里写 `$count` 拿到的是那个 ref 对象本身，要显示数值得写 `$count.value`，或者在 `<script setup>` 里先解一层。

这是纯 Vue 的规则（官方那句链接指到 Vue 文档的 "Caveat when unwrapping in templates"），不是 Nuxt 特有，但因为它俩都叫"响应式"，很容易在这里翻车。能不 provide 响应式对象就尽量别；真要共享状态，用 `useState`（251）。

## 给 $xxx 补类型

从插件 return 的 helper，官方说会自动带类型："If you return your helpers from the plugin, they will be typed automatically; you'll find them typed for the return of `useNuxtApp()` and within your templates."

要手动声明，就得同时 declare 两个接口（plugins 页的写法）：

```ts [index.d.ts]
declare module '#app' {
  interface NuxtApp {
    $hello (msg: string): string
  }
}

declare module 'vue' {
  interface ComponentCustomProperties {
    $hello (msg: string): string
  }
}

export {}
```

两个接口对应两个取用处：`#app` 的 `NuxtApp` 让 `useNuxtApp().$hello` 有类型；`vue` 的 `ComponentCustomProperties` 让模板里 `$hello` 有类型。末尾那句 `export {}` 是官方反复强调的——augment 一个模块必须带 import/export，否则变成覆盖而不是扩展。

还有个跨插件取用的细节，plugins 页 note：想在**另一个插件里**用某个已 provide 的 helper，可以调 `useNuxtApp()` 拿有类型的版本，但官方紧跟一句提醒——"in general, this should be avoided unless you are certain of the plugins' order."（插件间互相取用要先确定顺序，见 272）。

## 官方更推荐：能用组合式就别 provide

use-nuxt-app 的 `runWithContext` 那节不涉，但 provide 相关这句 important 必须放这儿，因为它决定你该不该走这条路：

> Note that we highly recommend using `composables` instead of providing helpers to avoid polluting the global namespace and keep your main bundle entry small.

一句 `hello(msg)` 级别的工具，直接 `export function useHello() { ... }` 放进 `app/composables/` 自动导入，比挂 `$hello` 更干净——provide 是在往全局命名空间加 `$` 键，会让主包变大。provide 的真正用武之地，是那些"确实需要挂在应用实例上、跨插件访问"的东西。

## 自查

- 取用记得 `$` 前缀：`provide('hello', ...)` → 读的是 `$hello`。
- provide 的是 `ref`/`computed`？模板里不会自动解包，要么 `.value`、要么先在 script 解一层——或者直接改用 `useState`。
- 跨插件用别人 provide 的 helper？先确认插件顺序（272），否则会拿到 undefined。
- 只是个小 helper？按官方建议封组合式，别挂 `$xxx` 污染全局、撑大主包。

## 下一篇

下一篇 275 讲 `nuxtApp.hook`：在插件里订阅运行时钩子（`app:created`、`vue:error`、`page:start` 这些渲染生命周期上的点）。

## 本文依据

- useNuxtApp（`api/composables/use-nuxt-app`）`provide (name, value)` 一节："Use the `provide` function to create Nuxt plugins to make values and helper methods available in your Nuxt application across all composables and components."、`provide` 接受 `name` 和 `value`、`nuxtApp.provide('hello', ...)` + `nuxtApp.$hello('name')` 示例、"`$hello` has become the new and custom part of `nuxtApp` context and it is available in all places where `nuxtApp` is accessible."
- plugins 目录（`directory-structure/app/plugins`）：Providing Helpers 两种写法（函数返回 `{ provide: {...} }` 与对象 setup 返回 `{ provide: {...} }`）、组件里 `const { $hello } = useNuxtApp()`、Typing Plugins（自动类型说明、`declare module '#app' { interface NuxtApp }` + `declare module 'vue' { interface ComponentCustomProperties }`、跨插件 `useNuxtApp()` 取用及其"避免除非确定顺序"note）、important "we highly recommend using composables instead of providing helpers ..."、warning "If your plugin provides a `ref` or `computed`, it will not be unwrapped in a component `<template>`."
- defineNuxtPlugin（`api/utils/define-nuxt-plugin`）：Basic Usage 里 `return { provide: { hello: ... } }` 形状。
