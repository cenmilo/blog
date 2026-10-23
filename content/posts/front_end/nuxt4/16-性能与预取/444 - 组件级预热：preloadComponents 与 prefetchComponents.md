---
url: /posts/27873c4c744d3974e13e77adb0946719/
title: 444 - 组件级预热：preloadComponents 与 prefetchComponents
date: 2027-03-09T09:00:00+08:00
updated: 2027-03-09T09:00:00+08:00
author: cmdragon

summary:
  摘要：`preloadComponents` 与 `prefetchComponents` 是手动的组件预热入口，但官方给它们划了三条硬边界：只对**已在应用里全局注册**的组件生效（默认这些组件被注册成异步组件）、组件名**必须写 PascalCase 形式**、**在服务端完全没有效果**。更值得先知道的是官方那句自述——`prefetchComponents` 的当前实现与 `preloadComponents` 行为完全相同，做的是 preload 而不是 prefetch，官方标了"我们正在改进这个行为"。两个函数都来自同一个源文件，参数都支持单个字符串或字符串数组，返回值都是可 await 的 Promise。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预热
  - preloadComponents
  - 组件
  - 性能
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

页面上有个"展开详情"的按钮，一点才去下载那个组件的 chunk，用户等了一截。你搜到 `prefetchComponents`，在按钮的点击处理里调它，或者干脆在页面加载后调一遍——**结果什么都没发生**。

这不是参数写错了。这两个函数各自的参考页加起来不到三十行，边界全在那几行里。

## preload 与 prefetch 的官方定义

`preloadComponents` 那一节的第一段是这么定义预热的：

> Preloading components loads components that your page will need very soon, which you want to start loading early in rendering lifecycle. This ensures they are available earlier and are less likely to block the page's render, improving performance.

重点是 **early in rendering lifecycle**（在渲染生命周期里提早开始），目标是"别挡着这个页面的渲染"。

`prefetchComponents` 那一节的定义口吻不同：

> Prefetching component downloads the code in the background, this is based on the assumption that the component will likely be used for rendering, enabling the component to load instantly if and when the user requests it. The component is downloaded and cached for anticipated future use without the user making an explicit request for it.

也就是**后台下载、假定用户接下来会要**。听起来一个服务于当前渲染、一个服务于下一次交互。

## 然后官方自己说：现在这俩是一样的

`prefetchComponents` 页面带了一条 `::note`，直接把上面的区分推翻了：

> Current implementation behaves exactly the same as `preloadComponents` by preloading components instead of just prefetching we are working to improve this behavior.

三件事在这句里：**当前实现完全等同于 preload**、它做的是 preload 而不是 prefetch、官方写着"正在改进"。

所以你现在按"prefetch 更不紧急、优先级更低"来编排策略，是在按一个还不存在的行为做设计。两个函数的源文件链接指向同一份 `composables/preload.ts`，也和这句话对得上。

## 三条硬边界

两节对用法的描述是一字不差的两段：

> Use `preloadComponents` to manually preload individual components that have been registered globally in your Nuxt app. By default Nuxt registers these as async components. You must use the Pascal-cased version of the component name.

> Use `prefetchComponents` to manually prefetch individual components that have been registered globally in your Nuxt app. By default Nuxt registers these as async components. You must use the Pascal-cased version of the component name.

拆出来三条：

| 边界 | 官方原话依据 | 写错的表现 |
| --- | --- | --- |
| 只对**全局注册**的组件生效 | "components that have been registered globally in your Nuxt app" | 普通按名字解析的组件不在它的名单里（文档未写会报什么） |
| 名字要 **PascalCase** | "You must use the Pascal-cased version of the component name" | 写 kebab-case 或目录前缀形式对不上 |
| **服务端无效** | "On server, `preloadComponents` will have no effect." / `prefetchComponents` 同句 | 放在服务端跑的插件里等于空转 |

第一条是最容易忽略的。437 那篇列过全局注册的代价——`components.global: true` 会给**每个**组件造一个异步 chunk、`global` 目录或 `.global.vue` 后缀的每个全局组件单独出一个 chunk，官方还补了句"be careful not to overuse"。也就是说：这两个函数能预热的对象，恰好就是那批本身有体积代价的全局组件。

## 用法

两节给的示例形状相同，单个和数组都支持：

```ts
await preloadComponents('MyGlobalComponent')

await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])
```

```ts
await prefetchComponents('MyGlobalComponent')

await prefetchComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])
```

官方在两节里都写了 `await`。结合上面"服务端无效"那条，这两个调用要放在**浏览器里跑**的位置——组件的 setup、事件处理函数、或者客户端插件。

## 怎么确认它真的取了什么

一个可执行的验证动作，不依赖猜：

1. 打开浏览器网络面板，清空记录。
2. 调用 `preloadComponents('MyGlobalComponent')` 并 await。
3. 看有没有一条新的 JS 请求进来。没有，就先查两件事：这个组件到底是不是全局注册的（`global` 目录、`.global.vue` 后缀或 `components.global: true`），以及你写的名字是不是 PascalCase 的那个组件名。
4. 再查调用发生在哪一侧——同一段代码若在服务端也执行，那一遍是空转。

## 自查

- 两个函数只对全局注册的组件有效，名字必须 PascalCase，服务端一律无效果。
- `prefetchComponents` 当前实现等同 `preloadComponents`，官方明写在改进中——别按"两者优先级不同"设计策略。
- 能被它预热的组件，正是那批每个都占一个独立 chunk 的全局组件，全局注册本身要克制。
- 示例里都带 `await`，调用点要放在浏览器执行的位置。

## 下一篇

下一篇 445 讲 `preloadRouteComponents`：它预热的是**整条路由的组件**而不是单个组件，官方给的示例特意不 await——理由是别挡住当前组件的 setup，同时"如果你用 `<NuxtLink>`，Nuxt 已经自动预取需要的路由了"。
