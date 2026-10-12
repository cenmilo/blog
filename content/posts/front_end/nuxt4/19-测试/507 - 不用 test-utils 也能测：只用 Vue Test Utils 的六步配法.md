---
url: /posts/8678c2dc01af7d3542bf660e41bd77d3/
title: 不用 test-utils 也能测：只用 Vue Test Utils 的六步配法
date: 2027-03-30T09:00:00+08:00
updated: 2027-03-30T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方在测试页末尾给了一条不装 `@nuxt/test-utils` 的路，成立条件写得很窄：**只测那些不依赖 Nuxt 组合式函数、自动导入或上下文的组件**。满足条件时六步就能跑起来——装 `vitest`、`@vue/test-utils`、`happy-dom`、`@vitejs/plugin-vue` 四个包，写一份只含 `plugins: [vue()]` 与 `environment: 'happy-dom'` 的 `vitest.config.ts`，在 `package.json` 加 `"test": "vitest"`，建一个 `<HelloWorld>`，在组件旁边建 `HelloWorld.spec.ts` 用 `mount` 测，然后跑。这份配置里没有 Nuxt，也没有 `.nuxt` 类型上下文。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - vue-test-utils
  - vitest
  - 组件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一个纯展示的组件——一段模板、几个 props、没有任何取数和注入——你想给它写测试。走 `@nuxt/test-utils` 那条路，你得配 project、要 `.nuxt` 类型、要 Nuxt 环境。这些对一个静态组件来说全是多余动作。

官方确实为这种情况准备了一节，就在测试页 Unit Testing 的最后。

## 先读条件，再读步骤

那节第一句把适用范围钉死了：

> If you prefer to use `@vue/test-utils` on its own for unit testing in Nuxt, and you are only testing components which do not rely on Nuxt composables, auto-imports or context, you can follow these steps to set it up.

两个前提：**只用 Vue Test Utils**，而且被测组件**不依赖 Nuxt 的组合式函数、自动导入或上下文**。三项里任何一项沾上，就不在这节的适用范围内——那种组件要用 Nuxt 环境的 `mountSuspended`／`renderSuspended`。

## 六步，一步不落

**第 1 步：装依赖。** 官方给的四个包是 `vitest`、`@vue/test-utils`、`happy-dom`、`@vitejs/plugin-vue`，npm 写法：

```bash [npm]
npm i --save-dev vitest @vue/test-utils happy-dom @vitejs/plugin-vue
```

同一段还给了 yarn（`yarn add --dev`）、pnpm（`pnpm add -D`）、bun（`bun add --dev`）三条等价命令。注意**这一节没有 deno**——前面 Playwright 那节的安装命令块里 deno 是有的，这节没有。

**第 2 步：建 `vitest.config.ts`。** 内容官方整份给出：

```ts
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'happy-dom',
  },
})
```

四行配置说明了很多事：用的是 Vitest 自己的 `defineConfig`（**不是** `defineVitestConfig`），环境写死 `happy-dom`，插件只有 `@vitejs/plugin-vue`。整个文件里没有 Nuxt 相关的任何东西——没有 `@nuxt/test-utils/config`，没有 project 划分，没有 `environmentOptions.nuxt`。

**第 3 步：在 `package.json` 加一条 test 命令。** 官方示例连上下文一起给：

```json
"scripts": {
  "build": "nuxt build",
  "dev": "nuxt dev",
  ...
  "test": "vitest"
},
```

`...` 是文档里就有的省略号，意思是保留你原有的其他脚本。

**第 4 步：建一个测试用的组件** `app/components/HelloWorld.vue`：

```vue
<template>
  <p>Hello world</p>
</template>
```

**第 5 步：写单元测试**，位置官方写的是 `~/components/HelloWorld.spec.ts`：

```ts
import { describe, expect, it } from 'vitest'
import { mount } from '@vue/test-utils'

import HelloWorld from './HelloWorld.vue'

describe('HelloWorld', () => {
  it('component renders Hello world properly', () => {
    const wrapper = mount(HelloWorld)
    expect(wrapper.text()).toContain('Hello world')
  })
})
```

这一段里有两个与前面几篇相反的细节，值得刻意注意：**组件是显式 `import` 进来的**（`./HelloWorld.vue`），不是从 `#components` 拿；`mount` **直接来自 `@vue/test-utils`**，不是 `mountSuspended`。这正是"不依赖自动导入"的代价与好处——你不需要 Nuxt 的自动导入，也就享受不到它。

还有一个位置细节：示例测试文件放在 **`~/components/` 里、紧贴被测组件**，而不是 `test/` 目录下。文档在这一节完全没提 `test/`，所以这是它给出的另一种布局。

**第 6 步：跑。** npm 是 `npm run test`，另有 `yarn test`、`pnpm run test`、`bun run test` 三条。

结尾那句是官方原话："Congratulations, you're all set to start unit testing with `@vue/test-utils` in Nuxt! Happy testing!"

## 这条路上你放弃了什么

按文档能列出来的缺失，就是它给的另一套东西的反面：

| 你放弃的 | 在 Nuxt 环境那条路上的对应物 |
| --- | --- |
| 异步 setup 与插件注入 | `mountSuspended`／`renderSuspended` 的两项能力 |
| `route` 与 `spy` 选项、`setupState` | 两个挂载 helper 都提供 |
| 自动导入 mock | `mockNuxtImport` |
| 组件 mock | `mockComponent` |
| 假接口 | `registerEndpoint` |
| Nuxt 别名与自动导入的类型上下文 | 默认只覆盖 `test/nuxt/` 与 `tests/nuxt/` |
| 服务端渲染与浏览器行为 | `@nuxt/test-utils/e2e` 与 Playwright |

这份表不是"所以别用这条路"，而是它的适用边界：官方把它写成**只给不依赖 Nuxt 的组件**用。哪天那个组件里多出一句 `useState`，测试就会从"这条路能走"变成"要走另一条"。

## 自查

- 成立条件是官方的两个限定：只用 Vue Test Utils，且组件不依赖组合式函数／自动导入／上下文。
- 六步的四个包与那份 4 行配置里没有 Nuxt：环境是 `happy-dom`，插件只有 `@vitejs/plugin-vue`。
- 测试文件在示例里紧贴组件（`~/components/HelloWorld.spec.ts`），组件用相对路径显式导入。
- 这一节没有 deno 安装命令，其他三个包管理器都有。
- 组件一旦需要 Nuxt 运行时（异步 setup、注入、自动导入），这条路不再适用，改用 `mountSuspended` 或 `renderSuspended`。

## 下一篇

下一篇 508 讲第三种执行环境：Vitest Browser Mode，配置四行、`page.render` 与 `render` 两种入口，以及那八个返回项。

## 本文依据

- https://nuxt.com/docs/4.x/getting-started/testing（Using `@vue/test-utils` 整节：适用条件原句、六步全部内容与代码（四包安装命令含 npm/yarn/pnpm/bun、`vitest.config.ts` 全文、`package.json` scripts 片段、`app/components/HelloWorld.vue`、`~/components/HelloWorld.spec.ts` 的 `mount` 测试、四条运行命令、结尾祝贺句））
- https://nuxt.com/docs/4.x/getting-started/testing（对照用：`mountSuspended`/`renderSuspended` 的 `route`/`spy`/`setupState`、`mockNuxtImport`/`mockComponent`/`registerEndpoint`、TypeScript Support in Tests 的默认目录）
