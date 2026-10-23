---
url: /posts/ba0baa4a3c82e56d8e60408c3da3c92f/
title: 506 - helper 全表：五个入口的参数、返回与不能混用
date: 2027-03-29T11:00:00+08:00
updated: 2027-03-29T11:00:00+08:00
author: cmdragon

summary:
  摘要：Helpers 一节一共五个入口，全在 `@nuxt/test-utils/runtime`：`mountSuspended`、`renderSuspended`、`mockNuxtImport`、`mockComponent`、`registerEndpoint`。这一篇给一张对照全表，并展开前面没细讲的 `mockComponent`：第一个参数可以是 PascalCase 组件名或相对路径（alias 也行），第二个参数可以是组件选项对象、工厂函数，或直接 `() => import('./MockComponent.vue')`；有一条硬提醒——**工厂函数里不能引用局部变量，因为它会被提升**，要用 Vue 的 API 得在工厂内部 `await import('vue')`。另一条边界在 Helpers 末尾：`/runtime` 与 `/e2e` 不能出现在同一个文件里。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - helper
  - mock
  - 组件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

五个 helper 都从 `@nuxt/test-utils/runtime` 引，但它们的参数形状完全不一样：有的第一个参数是组件，有的是字符串名字，有的是接口路径。写测试时最费时间的不是断言，是记这个。

## 一张全表

| helper | 官方定位 | 第一个参数 | 第二个参数 | 返回 |
| --- | --- | --- | --- | --- |
| `mountSuspended` | 在 Nuxt 环境里挂载任意 Vue 组件，允许 async setup 与插件注入 | 组件（示例从 `#components` 具名导入，或 `~/app.vue`） | 选项对象：`@vue/test-utils` 的 mount 选项 + `route`（默认 `/`，`false` 跳过初始路由切换）+ `spy`（默认 `false`） | mount 结果 + `setupState`（组件 setup 的返回） |
| `renderSuspended` | 同上，但用 `@testing-library/vue` | 组件 | `@testing-library/vue` 的 render 选项 + 同一对 `route`、`spy` | render 结果 + `setupState`；DOM 落在 `<div id="test-wrapper"></div>` |
| `mockNuxtImport` | mock Nuxt 的自动导入 | 自动导入的名字（字符串）**或**要 mock 的目标本身 | 工厂函数，可接 `original` 拿原实现 | 无（文档未描述返回值） |
| `mockComponent` | mock Nuxt 的组件 | PascalCase 组件名**或**组件相对路径 | 组件选项对象、工厂函数，或 `() => import(...)` | 无（文档未描述返回值） |
| `registerEndpoint` | 造一个返回假数据的 Nitro 接口 | 接口名（例如 `/test/`） | 返回假数据的工厂函数，**或**含 `handler`/`method`/`once` 的对象 | 无（文档未描述返回值） |

后三个是"改环境"的，没有返回可用对象——文档在三处都没写返回值，只写了怎么用。

## mockComponent：三种写法

定义两句：**"`mockComponent` allows you to mock Nuxt's component."**，然后 "The first argument can be the component name in PascalCase, or the relative path of the component. The second argument is a factory function that returns the mocked component."

官方给的示例把三种写法放在同一段里，注释标了各自的差异：

```ts
import { mockComponent } from '@nuxt/test-utils/runtime'

mockComponent('MyComponent', {
  props: {
    value: String,
  },
  setup (props) {
    // ...
  },
})

// relative path or alias also works
mockComponent('~/components/my-component.vue', () => {
  // or a factory function
  return defineComponent({
    setup (props) {
      // ...
    },
  })
})

// or you can use SFC for redirecting to a mock component
mockComponent('MyComponent', () => import('./MockComponent.vue'))

// your tests here
```

拆开是三条通路：

- **直接给选项对象**（第一段）：第二个参数不是函数，而是 `{ props, setup }` 这样的组件定义。
- **给路径 + 工厂**（第二段）：路径可以是相对路径，注释明写 **alias 也行**，所以 `~/components/my-component.vue` 这种写法是被示范过的。
- **重定向到另一个 SFC**（第三段）：工厂里直接 `import('./MockComponent.vue')`，官方注释是 "or you can use SFC for redirecting to a mock component"。

## 那条提升限制

紧跟示例的 Note 是这一节最容易撞的：

> **Note**: You can't reference local variables in the factory function since they are hoisted. If you need to access Vue APIs or other variables, you need to import them in your factory function.

原因是这些工厂会被提升，提升之后你文件顶部的局部变量在它眼里还不存在。官方给的解法是把依赖搬进工厂内部：

```ts
import { mockComponent } from '@nuxt/test-utils/runtime'

mockComponent('MyComponent', async () => {
  const { ref, h } = await import('vue')

  return defineComponent({
    setup (props) {
      const counter = ref(0)
      return () => h('div', null, counter.value)
    },
  })
})
```

注意这段的结构：工厂是 `async`，`ref` 与 `h` 在函数体里 `await import('vue')` 拿到，然后才在 `defineComponent` 里用。文档没有列出"哪些东西算局部变量"，但从这句 Note 的写法看，判断标准是**它在不在工厂函数体内被解析到**——凡是在文件顶层引的，都算。

## 与 e2e 那侧的边界

Helpers 一节末尾有一节专门划边界，第一句就是硬限制：

> `@nuxt/test-utils/runtime` and `@nuxt/test-utils/e2e` need to run in different testing environments and so can't be used in the same file.

官方给的解法是拆文件，并给出两个文件示例。一个是运行侧（`app.nuxt.spec.ts`）：

```ts
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport('useState', () => {
  return () => {
    return { value: 'mocked storage' }
  }
})
```

另一个是端到端侧（`app.e2e.spec.ts`）：

```ts
import { $fetch, setup } from '@nuxt/test-utils/e2e'

await setup({
  setupTimeout: 10000,
})

// ...
```

从这两段能读出三件事：**文件后缀被官方用作环境标记**（`.nuxt.spec.ts` 对 `.e2e.spec.ts`）；分环境另一种办法是在文件顶部写 `// @vitest-environment nuxt`；以及 `setupTimeout` 是可以按文件压小的（这里示例给 `10000`，默认是 `120000`，Windows 上 `240000`）。

## 自查

- 五个 helper 全在 `/runtime`，前两个"挂/渲染组件"、后三个"改环境"；后三个文档没写返回值。
- `mockComponent` 三条通路：选项对象、路径（相对或 alias）＋工厂、工厂里 `import()` 另一个 SFC。
- 工厂函数**不能引用局部变量**（会被提升）；要用 Vue 的 API 就在工厂里 `await import('vue')`。
- `/runtime` 与 `/e2e` 不能同文件；分法有两种——按后缀（`.nuxt.spec.ts`／`.e2e.spec.ts`）或按文件顶部 `// @vitest-environment nuxt`。
- `registerEndpoint` 默认 GET，改方法要传对象（`handler`/`method`/`once`），`once` 只服务第一个匹配请求。

## 下一篇

下一篇 507 讲官方给的另一条路：完全不装 `@nuxt/test-utils`，只用 Vitest 加 Vue Test Utils 测组件，六步配置，以及它成立的前提条件。
