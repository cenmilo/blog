---
url: /posts/1c04f0a5619075535ffd5a4a8a7d8a83/
title: 本地 modules 目录：两条自动注册模式、runtime 的硬要求与执行顺序
date: 2027-05-26T10:00:00+08:00
updated: 2027-05-26T10:00:00+08:00
author: cmdragon

summary:
  摘要：项目根下的 `modules/` 目录是"给自己项目写私有模块"的位置，**它只认两种文件模式：`modules/*/index.ts` 与 `modules/*.ts`**，命中就被自动注册，**不需要你再写进 `nuxt.config.ts` 的 `modules` 数组**。这一页还给了一个常被忽略的便利：**本地模块里可以 `import ... from 'nuxt/kit'`，这是助手子路径导入，意味着不必把 `@nuxt/kit` 加进项目依赖**。硬性要求是一条 `::note`：**所有本来该放在 `app/` 下的文件（组件、页面、组合式函数等）必须放进 `modules/your-module/runtime/app/`，理由是能正确做类型检查**。执行顺序与配置那一节一致：**先 `nuxt.config.ts` 里声明的，再 `modules/` 目录里发现的，后者按字母序**，而**给目录名加数字前缀是官方给的唯一改序办法**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 本地模块
  - 自动注册
  - runtime目录
  - 执行顺序
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

不是每个模块都要发成 npm 包。给一个项目自己写的东西，放进根目录 `modules/` 就够了——这一页讲它认哪些文件、以及放错位置会怎样。

## 两种被自动注册的文件模式

页面的定性是：**这里是放置你开发应用过程中编写的本地模块的好地方**，而自动注册匹配的模式只有两条：

- `modules/*/index.ts`
- `modules/*.ts`

**除这两种之外的文件不会被注册**——`modules/hello.vue`、`modules/hello/index.js`、`modules/nested/deep/index.ts` 都不在模式里（**页面没有说明深层嵌套会怎样，但按字面匹配，只有这一层**）。

紧接一句：**这些本地模块你不需要另外加进 `nuxt.config.ts`**。

## 一个完整例子

官方示例是两个文件一起给的：

```ts [modules/hello/index.ts]
// `nuxt/kit` is a helper subpath import you can use when defining local modules
// that means you do not need to add `@nuxt/kit` to your project's dependencies
import { addComponentsDir, addServerHandler, createResolver, defineNuxtModule } from 'nuxt/kit'

export default defineNuxtModule({
  meta: {
    name: 'hello',
  },
  setup () {
    const resolver = createResolver(import.meta.url)

    // Add an API route
    addServerHandler({
      route: '/api/hello',
      handler: resolver.resolve('./runtime/api-route'),
    })

    // Add components
    addComponentsDir({
      path: resolver.resolve('./runtime/app/components'),
      pathPrefix: true, // Prefix your exports to avoid conflicts with user code or other modules
    })
  },
})
```

```ts [modules/hello/runtime/api-route.ts]
export default defineEventHandler(() => {
  return { hello: 'world' }
})
```

**两行注释里各藏一个结论**：

**第一行讲的是导入面**：`nuxt/kit` 是本地模块定义时可以直接用的助手子路径导入，**所以不必把 `@nuxt/kit` 加进项目依赖**。这与 Kit 总页"请始终显式安装 `@nuxt/kit`"的要求并不冲突——**那条针对的是发布成 npm 包的模块**，而这里说的是项目内的本地模块。

**`pathPrefix: true` 后面那句注释是最佳实践的落地**："给你的导出加前缀，以避免与用户代码或其他模块冲突"。**这个键在组件目录那一页的描述是"用组件所在路径作为名字前缀"**，两处指的是同一个开关。

结果写得很直接：**启动 Nuxt 时 `hello` 模块会被注册，`/api/hello` 路由就可用了**。

## runtime 目录那条硬要求

页面用一条 `::note` 给出位置规则：

> Note that all components, pages, composables and other files that would be normally placed in your `app/` directory need to be in `modules/your-module/runtime/app/`. This ensures they can be type-checked properly.

**关键在最后半句：这样做是为了能被正确类型检查**，不是"可选的重构建议"。示例里 `addComponentsDir` 指的路径正是 `./runtime/app/components`，与这条规则对得上。

同类规则在三个地方各写了一遍，措辞与范围略有不同，值得并排看：

| 出处 | 原文列举的内容 | 要求的位置 |
| --- | --- | --- |
| 本页（`modules/` 目录） | components、pages、composables and other files | `modules/your-module/runtime/app/` |
| 基础配方页（组件与组合式函数两节，各重复一次） | components, pages, composables and other files | `runtime/app/` |
| 模块结构页 | Vue 组件、Composables、Nuxt 插件；Nitro 侧 API 路由、中间件、Nitro 插件；以及其他资产 | `src/runtime/`（starter 场景） |

三处一致的部分是"应用侧文件要进 `runtime/app/`"；差别在于**本页与配方页还点名了 pages**，而模块结构页那份清单里没有 pages——**它讲的是"能放哪些资产"而不是"必须放哪"**，所以这不算矛盾，但**要判断层里/模块里能不能放 pages，得看前两处**。

## 执行顺序与唯一改序办法

这一页把顺序规则又写了一遍，与配置参考里 `modules` 那节的 callout **逐字同构**：

- 先加载在 `nuxt.config.ts` 里定义的模块。
- 然后执行在 `modules/` 目录里找到的模块，**它们按字母序加载**。

改序的官方办法只有一条：**给目录名前面加数字**：

```bash [Directory structure]
modules/
  1.first-module/
    index.ts
  2.second-module.ts
```

**注意这个例子的形态**：`1.first-module/` 走 `index.ts` 模式，`2.second-module.ts` 走单文件模式，**两种模式可以混在同一层排序里**（示例本身就是一目录一文件）。

这里有个必须记住的边界：**数组里声明的模块永远先于目录发现的模块，加数字前缀只能改变目录内部的相对顺序，改变不了"目录整批在后面"这件事**。想让某个本地模块早于其他本地模块之外的东西跑，只能把它写进 `nuxt.config.ts` 的数组。

## 与 starter 的分工

同一个动作（写一个模块）在这两种场景下差别不小：

| 维度 | `modules/` 目录（本地模块） | 模块 starter（独立项目） |
| --- | --- | --- |
| 注册方式 | **放进目录即自动注册** | 装成包后写进 `modules` 数组，或用 `npm create nuxt -- -t module` 生成 |
| `@nuxt/kit` 依赖 | **不用加**（用 `nuxt/kit` 子路径） | 要在包 `dependencies` 里显式装 |
| 有没有 playground | 你自己的项目就是运行环境 | **自带一个已配好的 Nuxt 应用** |
| 构建产物 | 不需要构建 | `npm run prepack`（由 `@nuxt/module-builder` 提供，多数时候不必手动跑） |
| 发布 | 不适用 | `npm run release`（会跑 lint、test、构建，再发 npm 并推 git tag） |
| 改代码后 | 随项目正常热重载（**本页没有对本地模块的重载行为作任何说明**） | playground 的 dev **应在你改 `src` 时自动重载** |

**判据其实是一句话：这个模块会不会被第二个项目用到**。只给一个项目用，就放在 `modules/` 下省掉工具链；要给多个项目或社区用，starter 那套（含类型生成、构建、发布脚本）才划算。

页面末尾给的是一个跳转（指向模块作者指南）和一段视频链接，**没有其他内容**。

## 三条自查

1. 本地模块没生效，**先看文件名是否命中那两条模式**（最常见是写成 `modules/hello/index.js` 或多套了一层目录）。
2. 组件在模块里能渲染但类型检查报错，**检查是否放进了 `runtime/app/components` 而不是 `runtime/components`**。
3. 想调两个本地模块的先后，**只有数字前缀这一条路**；要跨"数组 vs 目录"调序，**只能把其中一个改成数组声明**。

文档未说明的还有：`modules/` 目录本身是否可被 `dir.modules` 改名（配置项那一节有 `dir.modules`，**本页没有引用它**）、本地模块能否带 `moduleDependencies`、以及自动注册是否读取模块的 `meta.compatibility`。

下一篇讲构建器：Nuxt 怎么把"打包"这件事交给可替换的一方。

## 本文依据

- https://nuxt.com/docs/4.x/directory-structure/modules
- https://nuxt.com/docs/4.x/guide/modules/getting-started
