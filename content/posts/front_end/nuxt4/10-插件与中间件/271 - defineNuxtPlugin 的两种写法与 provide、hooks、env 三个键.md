---
url: /posts/d7b10eabc99250da8dec79bb893c1888/
title: defineNuxtPlugin 的两种写法与 provide、hooks、env 三个键
date: 2027-01-10T10:00:00+08:00
updated: 2027-01-10T10:00:00+08:00
author: cmdragon

summary:
  摘要：defineNuxtPlugin 可以传函数、也可以传对象。函数写法最常见；对象写法多出一组配置键，其中和"往应用上挂能力、订阅运行时事件、控制是否参与岛屿渲染"直接相关的是三个键：provide、hooks、env。这一篇把两种写法各自的最小示例落地，逐个讲这三个键干什么、返回值长什么样，并把两个必须知道的坑放进正文：对象键必须静态可分析、provide 出去的 ref/computed 在模板里不会自动解包。

categories:
  - 前端开发

tags:
  - nuxt4
  - defineNuxtPlugin
  - provide
  - hooks
  - 插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

这一节只讲 `defineNuxtPlugin` 这一个函数的两种写法，以及对象写法里三个真正会改变插件行为的键：provide、hooks、env。顺序相关的键（enforce/order/parallel/dependsOn）下一篇单独讲，别混。

## 两种写法

官方 define-nuxt-plugin 页把写法一分为二：

> A plugin can be defined in two ways:
> **Function Plugin**: A function that receives the `NuxtApp` instance and can return a promise with a potential object with a `provide` property if you want to provide a helper on `NuxtApp` instance.
> **Object Plugin**: An object that can include various properties to configure the plugin's behavior, such as `name`, `enforce`, `dependsOn`, `order`, `parallel`, `setup`, `hooks`, and `env`.

函数写法——一个箭头函数，入参 `nuxtApp`：

```ts [plugins/hello.ts]
export default defineNuxtPlugin((nuxtApp) => {
  // 用 nuxtApp 做点事
})
```

对象写法——把逻辑放进 `setup`，其余键做配置：

```ts [plugins/hello.ts]
export default defineNuxtPlugin({
  name: 'my-plugin',
  enforce: 'pre',
  async setup (nuxtApp) {
    // 这里等价于一个普通函数插件
  },
  hooks: {
    'app:created' () {
      const nuxtApp = useNuxtApp()
    },
  },
  env: {
    islands: true,
  },
})
```

## 键 provide：往 NuxtApp 上挂全局 helper

要往应用实例上挂一个"哪都能拿到"的 helper，官方写法是**返回一个带 `provide` 键的对象**（不是直接调 nuxtApp 的某个方法）。两种写法都返回这个形状：

```ts [plugins/hello.ts]
export default defineNuxtPlugin(() => {
  return {
    provide: {
      hello: (msg: string) => `Hello ${msg}!`,
    },
  }
})
```

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

提供出去之后，`hello` 变成 NuxtApp 上的 `$hello`（约定：provide 的键名前面加 `$`），组件里这样用：

```vue [app/components/Hello.vue]
<script setup lang="ts">
const { $hello } = useNuxtApp()
</script>

<template>
  {{ $hello('world') }}
</template>
```

provide 出去的东西，Nuxt 会自动帮你在 `useNuxtApp()` 的返回值和模板里生成类型。想手动声明，就 declare `NuxtApp` 和 `ComponentCustomProperties` 两个接口（`declare module '#app' { interface NuxtApp { $hello (msg: string): string } }` + `declare module 'vue' { interface ComponentCustomProperties { $hello (msg: string): string } }`）。

两个必须记住的 provide 相关坑：

1. **provide 一个 ref 或 computed，在组件 `<template>` 里不会自动解包。** 官方 warning：
   > **If your plugin provides a `ref` or `computed`, it will not be unwrapped in a component `<template>`.**
   > This is due to how Vue works with refs that aren't top-level to the template.
   意思是 `$count` 是个 ref，模板里写 `$count` 拿到的是那个 ref 对象本身，要显示得用 `.value` 或在 `<script setup>` 里解一层。
2. **能用组合式就别用 provide。** plugins 页那句 important 直接写明了：
   > we highly recommend using `composables` instead of providing helpers to avoid polluting the global namespace and keep your main bundle entry small.
   这一条在下一篇 270 里已经出现过，放在这里是因为它约束的就是 provide 这个用法。

## 键 hooks：订阅运行时钩子

对象写法的 `hooks` 键，类型是 `Partial<RuntimeNuxtHooks>`——也就是"应用运行时钩子"的集合，可以在这里一次性注册、不用手动 `nuxtApp.hook(...)`。官方示例只演示了订阅 `app:created`（应用实例创建完成的那一刻）：

```ts
hooks: {
  'app:created' () {
    console.log('App created!')
  },
},
```

这里有个非常具体的好处，plugins 页在对象写法那句 note 里说清楚了：

> Nuxt does statically pre-load any hook listeners when using object-syntax, allowing you to define hooks without needing to worry about order of plugin registration.

翻成人话：如果你用函数写法 `nuxtApp.hook('app:created', ...)` 注册钩子，这个钩子的名字必须**先**由 Nuxt 广播、**再**由你的插件去订阅——广播的时机和插件注册的顺序有关，你就要操心顺序。对象写法的 `hooks` 会被 Nuxt **静态预加载**：Nuxt 在还没跑你的插件时，就已经知道"这个插件订阅了 `app:created`"，所以顺序不用你管。这一条在跨插件的钩子场景非常实用，也正是对象写法的核心动机之一。

具体哪些钩子名有效（`app:created` / `app:mounted` / `page:loading:start` / `vue:error` 等）在 `api/advanced/hooks` 那一页有全表——这一章后面会有一篇专门走这些钩子，这里先记住"hooks 是对象写法给你的一次性注册入口"。

## 键 env：控制岛屿 / 服务端渲染时是否跑

`env` 只有一个键 `islands?: boolean`。官方在参数表里给的描述：

> Set this value to `false` if you don't want the plugin to run when rendering server-only or island components.

先解释一下"岛屿"（第 106-109 篇 P04/P05 里已经铺过基础）：`.server.vue` 组件、或者 `components/islands/` 里的组件，会被 Nuxt 用**一个隔离的 Vue 应用**单独渲染——那个隔离应用**默认也会把插件再跑一遍**。有些插件（比如做埋点、做浏览器副作用、做全局 fetch 缓存的）在渲染一个"只展示、只有小段 HTML"的岛屿时不该跑，那就在 `env.islands` 设 `false`。设完之后，服务端渲染岛屿时这个插件被跳过，其他时机照常跑。

官方那一句 important 直接把它放在 plugins 页的对象写法示例里、并配了一句解释性说明："Set this value to `false` if you don't want the plugin to run when rendering server-only or island components."，`env` 类型就一个键 `islands?: boolean`。

## 静态分析：对象写法的一个隐藏约束

对象写法多出来的便利（`hooks` 预加载、类型更好推），代价是 plugins 页那句 note：

> If you are using the object-syntax, the properties are statically analyzed to produce a more optimized build. So you should not define them at runtime.
> For example, setting `enforce: import.meta.server ? 'pre' : 'post'` would defeat any future optimization Nuxt is able to do for your plugins.

含义很直白：`name`/`enforce`/`parallel`/`dependsOn`/`env` 这些键，Nuxt 是在**构建期**读它们来做优化的（把 `hooks` 提前预加载、把 `dependsOn` 依赖图算好等）。你如果按运行时条件动态给值——比如上面那句按 `import.meta.server` 决定 enforce——Nuxt 构建时看到的就是一个表达式而不是字面量，优化就失效了。

要写"服务端和客户端行为不同"的插件，别在对象键里做运行时判断，改用 `.server.ts`/`.client.ts` 后缀分成两个文件——那才是官方推荐的做法（270 也提过）。

## 参数表（官方原表逐行）

| Property | Type | Required | Description |
| --- | --- | --- | --- |
| `name` | `string` | `false` | Optional name for the plugin, useful for debugging and dependency management. |
| `enforce` | `'pre' \| 'default' \| 'post'` | `false` | Controls when the plugin runs relative to other plugins. |
| `dependsOn` | `string[]` | `false` | Array of plugin names this plugin depends on. Ensures proper execution order. |
| `order` | `number` | `false` | This allows more granular control over plugin order and should only be used by advanced users. **It overrides the value of `enforce` and is used to sort plugins.** |
| `parallel` | `boolean` | `false` | Whether to execute the plugin in parallel with other parallel plugins. |
| `setup` | `Plugin<T>` | `false` | The main plugin function, equivalent to a function plugin. |
| `hooks` | `Partial<RuntimeNuxtHooks>` | `false` | Nuxt app runtime hooks to register directly. |
| `env` | `{ islands?: boolean }` | `false` | Set this value to `false` if you don't want the plugin to run when rendering server-only or island components. |

`enforce`、`dependsOn`、`order`、`parallel` 四个和顺序相关的键，下一篇（272）单独展开。

## 自查

- 要挂全局 helper：函数/对象写法都 `return { provide: { ... } }`，用 `$name` 读；提供的是 `ref`/`computed` 时记得模板里不解包。
- 想订阅运行时钩子：优先对象写法的 `hooks` 键，Nuxt 静态预加载、不用管插件注册顺序。
- 只在岛屿/服务端渲染时不该跑：`env.islands: false`，别在 setup 里自己判 `islandContext`。
- 对象写法的所有键都必须**静态可分析**，`import.meta.server ? 'pre' : 'post'` 这种运行时判断会打掉构建优化——要分端就拆文件（`.server.ts`/`.client.ts`）。

## 下一篇

下一篇 272 讲插件的顺序：文件名补零规则、`enforce` 三档、`order` 数字（会覆盖 enforce）、以及 `parallel` 与 `dependsOn` 怎么组合。

## 本文依据

- defineNuxtPlugin（`api/utils/define-nuxt-plugin`）：
  - 两种写法定义句（Function Plugin / Object Plugin 两段原文）。
  - Signature：`Plugin<T>` 与 `ObjectPlugin<T>` 完整类型（八个键）。
  - Parameters 表 8 行原样落地。
  - `order` 那句 "It overrides the value of `enforce` and is used to sort plugins."
  - `env` 那句 "Set this value to `false` if you don't want the plugin to run when rendering server-only or island components."
  - Basic Usage 示例（`return { provide: { hello: (name: string) => \`Hello ${name}!\` } }`）。
  - Object Syntax Plugin 示例（`hooks: { 'app:created' () { console.log('App created!') } }`）。
- plugins 目录（`directory-structure/app/plugins`）：
  - 函数写法 provide 两种形态示例（`plugins/hello.ts` 与 `plugins/hello-object-syntax.ts`）。
  - 组件使用示例 `const { $hello } = useNuxtApp()`。
  - Typing Plugins 一节的 `declare module '#app'` / `declare module 'vue'` 两处接口扩展。
  - important "we highly recommend using composables instead of providing helpers ..."。
  - warning "If your plugin provides a `ref` or `computed`, it will not be unwrapped in a component `<template>`."
  - Object Syntax Plugins 里那句 note："If you are using the object-syntax, the properties are statically analyzed to produce a more optimized build. So you should not define them at runtime."，反例 `enforce: import.meta.server ? 'pre' : 'post'`；"Nuxt does statically pre-load any hook listeners when using object-syntax, allowing you to define hooks without needing to worry about order of plugin registration."
