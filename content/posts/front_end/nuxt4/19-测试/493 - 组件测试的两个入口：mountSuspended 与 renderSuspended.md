---
url: /posts/bde4a468fc7eac26f13ec58519ff9437/
title: 493 - 组件测试的两个入口：mountSuspended 与 renderSuspended
date: 2027-03-25T10:00:00+08:00
updated: 2027-03-25T10:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 里的组件不能直接用 Vue 的 `mount` 挂——组件里的异步 `setup` 和插件提供的注入都会落空。官方给的两个入口是 `mountSuspended` 和 `renderSuspended`，都从 `@nuxt/test-utils/runtime` 引。区别只有一个：前者包的是 `@vue/test-utils` 的 `mount`，后者走 `@testing-library/vue` 的 `render`，要额外自装依赖、开 Vitest globals，而且渲染结果固定在 `<div id="test-wrapper"></div>` 里。两个入口的选项都是「库里那套 + `route`（默认 `/`，传 `false` 跳过初始路由切换）+ `spy`（默认 `false`）」，返回值都额外带一个 `setupState`，也就是组件 `setup` 的返回。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组件测试
  - vitest
  - test-utils
  - 测试
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

你在 `test/nuxt/` 里建了第一个组件测试，从 Vue 那边把 `mount` 拿过来，一跑就红：组件里那句 `await useFetch('/api/x')` 没人应答，或者 `useState` 相关的调用直接说找不到上下文。

原因不在你的写法，在于你的组件不是普通 Vue 组件——它假定外面有一个已经初始化好的 Nuxt 应用。官方给的两个入口就是替你把这层环境垫上。

## 两个入口分别是什么

`mountSuspended` 的定义原文：

> `mountSuspended` allows you to mount any Vue component within the Nuxt environment, allowing async setup and access to injections from your Nuxt plugins.

拆开看是三个信息：在 **Nuxt 环境里**挂载、允许 **async setup**（也就是 `setup` 里能 `await`）、允许**取到你在 Nuxt 插件里 provide 出去的注入**。它从 `@nuxt/test-utils/runtime` 导入。

`renderSuspended` 的定义只多了一个状语：

> `renderSuspended` allows you to render any Vue component within the Nuxt environment using `@testing-library/vue`, allowing async setup and access to injections from your Nuxt plugins.

三个信息一样，多了 **using `@testing-library/vue`**。这一句就决定了它俩的取舍：你打算用哪套查询 API 写断言。

## mountSuspended 的三条官方示例

第一条最小，断言的是文本：

```ts [tests/components/SomeComponents.nuxt.spec.ts]
import { expect, it } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { SomeComponent } from '#components'

it('can mount some component', async () => {
  const component = await mountSuspended(SomeComponent)
  expect(component.text()).toMatchInlineSnapshot(
    '"This is an auto-imported component"',
  )
})
```

注意组件是**从 `#components` 具名导入**的，也就是你在模板里不写 import 就能用的那个自动导入列表，测试里同样能拿到。官方示例文件顶部还有一段 `declare module '#components'` 的补类型代码，那是文档为了让示例通过类型检查加的，不是你要照抄的部分。

第二条挂整个 `app.vue`，并且传了路由：

```ts [tests/App.nuxt.spec.ts]
import { expect, it } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import App from '~/app.vue'

it('can also mount an app', async () => {
  const component = await mountSuspended(App, { route: '/test' })
  expect(component.html()).toMatchInlineSnapshot(`
      "<div>This is an auto-imported component</div>
      <div> I am a global component </div>
      <div>/</div>
      <a href="/test"> Test link </a>"
    `)
})
```

这两段放一起能看出一件事：**快照里那行 `<div>/</div>` 是 `/`，不是 `/test`**。`route: '/test'` 设的是初始路由，页面里显示的当前路径仍然是 `/`。文档没解释这个差异，你要断言路径显示的话，先按实际渲染结果写。

第三条是 `spy`：

```ts [tests/components/SomeComponents.nuxt.spec.ts]
import { expect, it, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { SomeComponent } from '#components'

it('can spy on setup state', async () => {
  const component = await mountSuspended(SomeComponent, { spy: true })
  vi.mocked(component.setupState.someMethod).mockImplementation(() => 'mocked')
})
```

`setupState` 就是组件 `setup` 的返回值，不开 `spy` 你拿不到可替换的实现。

## 选项与返回值：一次列全

官方对 `mountSuspended` 的选项说明是两行加一句前提——「The options object accepts `@vue/test-utils` mount options and the following properties」：

| 属性 | 官方说明 | 类型 | 默认 |
| --- | --- | --- | --- |
| `route` | the initial route, or `false` to skip the initial route change | `string \| boolean`（示例里出现 `'/test'`） | `/` |
| `spy` | enable spy component setup state | `boolean` | `false` |

返回值那一句：「The return object includes the `@vue/test-utils` mount result and the following properties」，多出来的是 `setupState`: the return value of the component setup。

`renderSuspended` 的这两张表**逐字相同**，只是底座换成 `@testing-library/vue`：选项接受该库 `render` 的选项加同一对 `route`、`spy`（`spy` 那行直接写"See the `mountSuspended` example above"），返回是 `render` 结果加 `setupState`。

## renderSuspended 多出来的三条要求

定义之外，文档给了三条不做就用不起来的说明：

- **要自己装 `@testing-library/vue`**。原文是 "Install `@testing-library/vue` in your project to use these"，指的是 `screen`、`fireEvent` 那批工具，`@nuxt/test-utils` 不带它。
- **要打开 Vitest 的 globals**。原文：Testing Library 依赖 testing globals 做清理，"You should turn these on in your Vitest config"。
- **渲染位置是写死的**：「The passed in component will be rendered inside a `<div id="test-wrapper"></div>`」。

对应的示例只有断言方式变了：

```ts [tests/components/SomeComponents.nuxt.spec.ts]
import { expect, it } from 'vitest'
import { renderSuspended } from '@nuxt/test-utils/runtime'
import { SomeComponent } from '#components'
import { screen } from '@testing-library/vue'

it('can render some component', async () => {
  await renderSuspended(SomeComponent)
  expect(screen.getByText('This is an auto-imported component')).toBeDefined()
})
```

挂 `app.vue` 那条示例的快照外层就是 `<div id="test-wrapper">`，正好印证上一条。

## 该选哪个

文档没给推荐。能从记载里推出来的只有依据：`mountSuspended` 的 `::note` 写的是「内部包的就是 `@vue/test-utils` 的 `mount`，能传哪些选项去看 Vue Test Utils 文档」；`renderSuspended` 的对应依据是 Testing Library 的查询语义（`getByText` 按可见文本找，不写选择器）。

所以分界线是断言风格，不是能力：要 `wrapper.text()`、`wrapper.html()`、`find()` 选择器就前者；要 `screen.getByRole`、`fireEvent` 就后者，代价是多装一个依赖并开 globals。两个都别拿来测真实浏览器行为——那属于 `@nuxt/test-utils/e2e`，下一篇讲。

## 自查

- 组件里 `await` 过数据、用过插件注入，就不能裸 `mount`；`mountSuspended` 与 `renderSuspended` 补的正是「Nuxt 环境 + async setup + 注入」。
- 二者选项都是「库自身选项 + `route` + `spy`」，`route` 默认 `/`、传 `false` 跳过初始路由切换，`spy` 默认 `false`；返回都多一个 `setupState`。
- `renderSuspended` 的三处额外成本：自装 `@testing-library/vue`、开 Vitest globals、DOM 根节点固定在 `#test-wrapper`。
- 官方示例从 `#components` 具名导入组件；`declare module '#components'` 是示例为类型检查加的，不是必需写法。
- 传 `route: '/test'` 之后快照里的路径行仍显示 `/`，文档未解释，断言按实际渲染写。

## 下一篇

下一篇 494 讲端到端这一侧：`setup()` 的三组选项（Nuxt 配置、时间、功能）逐条默认值，以及 `createPage` 和 Playwright 测试运行器两种浏览器驱动方式的差别。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/testing（Helpers 一节：`mountSuspended` 与 `renderSuspended` 的定义句、`::note` 的 wraps `mount`、三条示例与快照内容、`route`/`spy` 两行选项说明、返回值含 `setupState`；`renderSuspended` 的自装依赖／testing globals／`<div id="test-wrapper"></div>` 三条）
- https://nuxt.com/docs/4.x/getting-started/testing（Unit Testing 的目录与文件名约定，示例文件名 `*.nuxt.spec.ts`）
