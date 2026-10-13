---
url: /posts/7a8901bf9b1944d2d99af35e9cb34bf4/
title: 022 - Vapor Mode：让个别组件跳过虚拟 DOM 渲染
date: 2026-10-19T10:00:00+08:00
updated: 2026-10-19T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：Vapor Mode 是 Vue 3.6 引入的一种替代编译策略，它在没有 Virtual DOM 的情况下渲染组件，从而降低内存占用并改善运行时性能。官方明确它是实验特性、需要 Vue 3.6 或更新版本且 API 可能改变；如果你在一个更旧的 Vue 版本上开启 vue.vapor，Nuxt 会警告并把它关闭。Nuxt 以 interop 模式支持它：应用仍用 Virtual DOM 渲染，你通过在 `<script setup>` 上加 vapor 属性把单个组件或页面选进 Vapor，因此可以渐进采用，而 Nuxt 的其余部分（路由、useAsyncData、布局与大多数内置组件）保持不变地继续工作；开启方式是配置 vue.vapor: true，它会安装 Vue 的 vaporInteropPlugin。官方还给出六条已知限制，其中三条会直接影响你已有代码：Options API 在 Vapor 组件中不受支持（因此 defineNuxtComponent 的 asyncData 与 fetchKey 不可用，要改用 `<script setup>` 配 useAsyncData）、模板 ref 不暴露 $el、以及在 Vapor 的 `<script setup>` 里 await 之后调用的 Nuxt 组合式函数可能丢失 Nuxt 上下文。

categories:
  - 前端开发

tags:
  - nuxt4
  - vapor
  - vue
  - 性能
  - 实验特性

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


认知奠基这一段收尾，我放了一个"看着离你很远、但早晚会遇到"的特性：Vapor Mode。写它不是为了让你现在去开，而是让你**在读到它时知道它是哪一类东西**——性能优化？语法变更？还是编译策略。

## 它是什么：另一种编译策略

官方的定义一句话：

> Vapor Mode 是 Vue 3.6 引入的一种**替代编译策略**，它在**没有 Virtual DOM** 的情况下渲染组件，从而降低内存占用并改善运行时性能。

注意它属于"怎么把模板变成代码"这一层，**不是新的组件写法**。它也不是"性能开关"——收益来自少了一层虚拟 DOM 的比对。

## 三个限定条件，先看清

官方在标题下第一条就是警告：

> Vapor Mode 支持是**实验性的**，并且需要 **Vue 3.6 或更新版本**。API 可能改变。

第二条关于版本不够时会怎样：

> 如果你在一个更旧的 Vue 版本上启用 `vue.vapor`，Nuxt 会**警告并把它关闭**。

第三条是版本要求怎么落地——它给的示例很直白：

```json [package.json]
{
  "dependencies": {
    "vue": "^3.6.0-rc.2"
  }
}
```

> Vapor Mode 需要一个同时包含 Vapor 与其互操作修复的 Vue 版本。在 Vue 3.6 处于预发布阶段期间，安装一个包含这些修复的候选发布版（或更新版本）。

（这条正好接上 016 讲那句"破坏性变更只会发生在明确标记为 `experimental` 的选项之内"——Vapor 就是这样一个选项。）

## 怎么开：配置一个键，逐组件加属性

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  vue: {
    vapor: true,
  },
})
```

官方对这一行的解释：

> 这会安装 Vue 的 `vaporInteropPlugin`，它让 Vapor 与 Virtual DOM 组件能彼此相邻渲染。然后你可以通过**给 `<script setup>` 添加 `vapor` 属性**，把任何组件或页面标记为 Vapor。

```vue [app/pages/index.vue]
<script setup vapor lang="ts">
const count = ref(0)
</script>
```

而它的整体策略是渐进：

> Nuxt 以 **interop 模式**支持 Vapor Mode：你的应用仍然用 Virtual DOM 渲染，而你把**单个组件或页面**通过写成 Vapor 单文件组件的方式选入 Vapor。这让你可以**渐进采用** Vapor，而 Nuxt 的其余部分（路由、`useAsyncData`、布局以及大多数内置组件）保持不变地继续工作。

**这条很重要**：`vue.vapor: true` 不会把整个应用变成 Vapor。它只是"允许你逐个标记"。真正决定某个组件走哪条路径的，是那个 `vapor` 属性。

官方对使用场景也给了一句建议：

> 我们建议在**一个新项目**里尝试 Vapor Mode，而不是把它加到已有项目里。

## 六条已知限制：逐条对照你已经学的东西

官方说得很清楚为什么会有这些：

> 因为 Nuxt 以互操作模式运行 Vapor，一些依赖 Virtual DOM 的模式在 Vapor 组件内表现得不同。

**一、完整 Vapor 应用尚不支持。**

> 应用根仍然是 Virtual DOM，你按组件选入；我们以后可能会支持完整 Vapor 应用。

**二、模板 ref 拿不到 `$el`。**

> Vapor 组件上的模板 ref 不暴露 `$el`。

**三、一些内置组件读不到 Vapor 的插槽内容。** 这条最要小心：

> 一些内置组件会检查它们的插槽内容，而无法读取 Vapor 插槽子节点。`<ClientOnly>` 会跳过它的属性透传下沉，而 `<NuxtIsland>`、服务端组件，以及读取文本子节点的 head 组件（例如 `<Title>`、`<Style>` 与 `<Noscript>`）在你传入 Vapor 插槽时会告警。请**直接把值传进去**（例如给 `<Title>` 传一个字符串）。

也就是说这些组件不是"不能用"，而是**用法要换**：从"往插槽里塞节点"改成"直接传值"。

**四、Options API 不受支持。** 这条直接废掉 162 讲那条路（编号在后，但机制你已经知道了）：

> Options API 在 Vapor 组件中**不受支持**，因此通过 `defineNuxtComponent` 使用 `asyncData` 与 `fetchKey` 都不可用。请改用 `<script setup>` 配合 `useAsyncData`。

**五、带 key 的 `onPrehydrate` 会退化。**

> 带 key 的 `onPrehydrate` 会退化成不带 key 的形式（并在开发环境告警），因为没有组件实例可以挂载这个 key。

**六、await 之后调组合式函数可能丢上下文。** 这条和 130/151 那套"上下文"知识连在一起：

> 在 Vapor 的 `<script setup>` 里 `await` **之后**调用的 Nuxt 组合式函数，**可能丢失 Nuxt 上下文**。请尽可能在第一个 `await` 之前调用它们。

注意这条与 130 讲的关系：官方在讲自动导入时说的是"`<script setup>`、`defineNuxtComponent`、`defineNuxtPlugin`、`defineNuxtRouteMiddleware` 这四个位置里，框架做了转换，`await` 之后仍保留上下文"。**而在 Vapor 组件里，这个保障被官方明确打了问号。** 这就是"实验特性"三个字的真实成本。

官方的收尾判断也值得抄：

> 大多数其它 Nuxt 组合式函数与内置组件都能照常工作，因为它们依赖的是**注入**，而不是组件实例。

## 现在要不要用它：我的判断

三个事实放一起就够决策了：

1. 官方说它是实验特性、API 可能改变；
2. 官方建议在新项目里试，而不是加到已有项目；
3. 六条限制里有三条会推翻你已经写过的写法（Options API、`$el`、`await` 之后的上下文）。

所以：**把它当"以后要评估的性能选项"，不要当默认工程实践。** 真有性能诉求时，先按 010/012 讲那条链去看产物和取数（分包、按需水合、`routeRules` 缓存），这些是文档里成熟的路径；Vapor 留到最后。

（这是我给的建议，不是官方结论；官方给的是"建议在新项目尝试"。）

## 自查

- [ ] 你能说出 Vapor 省的是哪一层（Virtual DOM）而不是"打包更小"吗？
- [ ] 你的 Vue 版本满足 3.6+（含互操作修复）吗？不满足时会发生什么？
- [ ] `vue.vapor: true` 之后，具体某个组件走哪条路径由什么决定？
- [ ] 六条限制里，有哪几条会命中你已有的写法（Options API、`$el`、内置组件插槽、keyed `onPrehydrate`、await 之后调组合式函数）？
- [ ] 有没有把它当成"性能开关"直接开在存量项目上？

## 下一篇

023 起是环境与第一个项目那一组（023-049），题目按官方文档逐页拆出来定。

## 本文依据

- `guide/concepts/vuejs-development`（Vapor Mode 支持为实验性、需 Vue 3.6+、API 可能改变；作为 Vue 3.6 引入的替代编译策略、无 Virtual DOM 渲染从而降低内存与改善运行时性能；在更旧 Vue 上启用 vue.vapor 会被警告并关闭；package.json 里 ^3.6.0-rc.2 的例子与"需要同时包含 Vapor 与互操作修复的版本"；vue.vapor: true 与安装 vaporInteropPlugin、给 `<script setup>` 加 vapor 属性以把任意组件或页面标记为 Vapor；interop 模式下应用仍用 Virtual DOM 渲染、可渐进采用、路由/useAsyncData/布局/多数内置组件照常工作；建议在新项目尝试；六条已知限制（完整 Vapor 应用不支持、模板 ref 无 $el、内置组件读不到 Vapor 插槽内容且 <ClientOnly> 跳过属性透传而 <NuxtIsland>/服务端组件/<Title>/<Style>/<Noscript> 会告警并应直接传值、Options API 不支持使 defineNuxtComponent 的 asyncData 与 fetchKey 不可用并改用 `<script setup>` + useAsyncData、keyed onPrehydrate 退化并告警、Vapor `<script setup>` 中 await 之后可能丢上下文而应尽量在第一个 await 之前调用；"依赖注入而非组件实例所以多数照常工作"）
- `community/roadmap`（破坏性变更只在明确标记 experimental 的选项内出现——用于解释 Vapor 的"API 可能改变"）
- `guide/concepts/auto-imports`（await 之后仍保留上下文的四个受支持位置，与 Vapor 下的例外形成对照）
