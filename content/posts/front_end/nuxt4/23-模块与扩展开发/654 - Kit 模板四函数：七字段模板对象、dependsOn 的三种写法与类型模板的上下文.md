---
url: /posts/f5ad431283b9cff323b2167cb5f74c63/
title: 654 - Kit 模板四函数：七字段模板对象、dependsOn 的三种写法与类型模板的上下文
date: 2027-05-18T09:00:00+08:00
updated: 2027-05-18T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：模板就是"由模块生成的额外文件"，这一节有四个函数。**`addTemplate` 的模板对象七个字段**，其中最新、也最容易漏的是 `dependsOn`：**什么都不声明的模板会在每一次变化时重新生成**，所以要么给 `[]`（表示它不读被监听文件的内容，dev 下文件被编辑时可以跳过重编译）、要么列已知来源键（`'pages'`、`'plugins'`）、要么给一个按变化判断的函数。`addTypeTemplate` **只写进 buildDir 并登记为类型**，它的第二个参数 `context` 控制类型进哪个上下文——**省略时只进 Nuxt 上下文，要进服务端得显式 `{ nitro: true }`**。`addServerTemplate` 只有两个必填字段。**这一节有两处原文瑕疵值得防身**：`addTypeTemplate` 示例前的句子函数名是空的（写成"By default, －－ only adds..."），而 `addServerTemplate` 注册的 `filename` 是 `.mjs`、运行时示例却 import `.js`。

categories:
  - 前端开发

tags:
  - nuxt4
  - 虚拟文件
  - 模板生成
  - NuxtKit
  - 类型声明
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`#build/` 下那些看起来凭空出现的文件，就是模块用这一节的函数生成的。四个函数分工不同，而**默认行为里藏着一个会让 dev 变慢的坑**。

## 这一节怎么介绍四个函数

开场句：**模板让你在开发期与构建期生成额外的文件；这些文件会出现在虚拟文件系统里，可以在插件、布局、组件等处使用**。

然后点名：**`addTemplate` 与 `addTypeTemplate` 用于给 Nuxt 应用添加模板；`updateTemplates` 用于重新生成匹配过滤条件的模板**。

**注意这句只点了三个名字，而页上有四个 `##` 条目**——`addServerTemplate` 没在开场句里出现，只在它自己那一节被介绍（**添加一个可在 Nitro 服务端构建里使用的虚拟文件**）。查找时别以为它不属于这一节。

## `addTemplate`：七个字段

```ts
function addTemplate (template: NuxtTemplate | string): ResolvedNuxtTemplate
```

定性：**在构建期渲染给定模板到虚拟文件系统，并可选地写入项目 `buildDir` 磁盘**。它有返回值（`ResolvedNuxtTemplate`），页面没有说明返回值能做什么。

`template` 可以是模板对象或路径字符串（**给字符串时转成 `src` 为该值的模板对象**）：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `src` | `string` | `false` | 模板路径；**不给 `src` 就必须给 `getContents`** |
| `filename` | `string` | `false` | 模板文件名；不给时从 `src` 生成，**此时 `src` 变成必需** |
| `dst` | `string` | `false` | 目标路径；不给时由 `filename` 与 `buildDir` 生成 |
| `options` | `Options` | `false` | 传给模板的选项 |
| `getContents` | `(data: Options) => string \| Promise<string>` | `false` | 以 `options` 对象被调用，返回字符串或 promise；**给了 `src` 时这个函数被忽略** |
| `write` | `boolean` | `false` | **为 `true` 时写入目标文件；否则只存在于虚拟文件系统** |
| `dependsOn` | `Array<'pages' \| 'plugins'> \| ((change, ctx) => boolean)` | `false` | 见下一节 |

## `dependsOn`：不写就每次重算

这一项的说明句是整页最有信息量的一句：**除了 `nuxt.options` 与应用解析出的结构之外，这个模板的输出还可能依赖哪些被监听输入**；**如果模板从不读取被监听文件的内容，就设成 `[]`，这样在 dev 下文件变化而没有文件增删时，Nuxt 可以跳过重新编译它**；如果模板读的是那些已知来源，**就按键列出来**；否则**传一个函数按每次变化来决定**。

最后一句请务必记住：**A template that declares nothing is regenerated on every change.**（什么都不声明的模板会在每一次变化时重新生成。）

页面给的默认行为解释是：**Nuxt 默认会在每个被监听文件事件上重新编译模板，因为它无法知道模板读了什么**。

三种写法各一段：

```ts
addTemplate({
  filename: 'my-module/config.mjs',
  dependsOn: [],
  getContents: () => 'export default ' + JSON.stringify(options),
})
```

```ts
addTemplate({
  filename: 'my-module/routes.mjs',
  dependsOn: ['pages'],
  getContents: ({ app }) => generateRoutes(app.pages),
})
```

```ts
addTemplate({
  filename: 'my-module/content.mjs',
  dependsOn: ({ path }) => path.endsWith('.yaml'),
  getContents: () => generateContents(),
})
```

对应三种情形：**只用配置的给 `[]`；读页面清单的给 `'pages'`；自己扫 YAML 这类未知来源的用函数**。

注意第二段的写法——`getContents` 的解构参数是 `{ app }`，**这就是文档里说的"应用的解析结构"会作为数据传进来**（与 `dependsOn: ['pages']` 配套）。

## 从 `#build` 导入

完整的一对示例（模块生成 `meta.config.mjs`，运行时插件消费它）：

```ts [module.ts]
import { addTemplate, defineNuxtModule } from '@nuxt/kit'
import { defu } from 'defu'

export default defineNuxtModule({
  setup (options, nuxt) {
    const globalMeta = defu(nuxt.options.app.head, {
      charset: options.charset,
      viewport: options.viewport,
    })

    addTemplate({
      filename: 'meta.config.mjs',
      getContents: () => 'export default ' + JSON.stringify({ globalMeta, mixinKey: 'setup' }),
    })
  },
})
```

```ts [runtime/plugin.ts]
import { createHead as createServerHead } from '@unhead/vue/server'
import { createHead as createClientHead } from '@unhead/vue/client'
import { defineNuxtPlugin } from '#imports'
// @ts-expect-error - virtual file
import metaConfig from '#build/meta.config.mjs'

export default defineNuxtPlugin((nuxtApp) => {
  const createHead = import.meta.server ? createServerHead : createClientHead
  const head = createHead()
  head.push(metaConfig.globalMeta)

  nuxtApp.vueApp.use(head)
})
```

**运行时侧要写成 `#build/meta.config.mjs`**，并且官方在 import 那行上面加了 `// @ts-expect-error - virtual file`：**因为这份文件不在类型系统的视野里**。第二处细节是 `defineNuxtPlugin` **从 `#imports` 显式导入**——这正是"发布模块的 runtime 不启用自动导入"那条警告的正确应对。

## `addTypeTemplate`：第二个参数决定进哪个上下文

```ts
function addTypeTemplate (template: NuxtTypeTemplate | string, context?: { nitro?: boolean, nuxt?: boolean }): ResolvedNuxtTemplate
```

定性：**在构建期渲染给定模板到项目 buildDir，然后把它登记为类型**。

模板对象的字段与 `addTemplate` 相同但**少两项**：`src`、`filename`、`dst`、`options`、`getContents` 五项，**没有 `write`，也没有 `dependsOn`**。

`context` 那一栏的说明值得逐句读：**可选的上下文对象用来控制类型加到哪里；省略时类型只加到 Nuxt 上下文**。两个键：

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `nuxt` | `boolean` | `false` | 为 `true` 时把类型加进 Nuxt 上下文 |
| `nitro` | `boolean` | `false` | 为 `true` 时把类型加进 Nitro 上下文 |

用法：

```ts
addTypeTemplate({
  filename: 'types/auth.d.ts',
  getContents: () => `declare module '#auth-utils' {
  interface User {
    id: string;
    name: string;
  }

}`,
}, {
  nitro: true,
})
```

这样服务端代码才能用它：

```ts [server/api/auth.ts]
import type { User } from '#auth-utils'

export default eventHandler(() => {
  const user: User = {
    id: '123',
    name: 'John Doe',
  }
  return user
})
```

**这一对示例正好是 642 那条限制的解法之一**：虚拟文件的类型默认只在 app 上下文可见，服务端要用就得显式打开 `nitro`。

另一处示例是给 `*.md` 这类导入补类型：

```ts
addTypeTemplate({
  filename: 'types/markdown.d.ts',
  getContents: () => `declare module '*.md' {
  import type { ComponentOptions } from 'vue'
  const Component: ComponentOptions
  export default Component
}`,
})
```

## `addServerTemplate`：只有两个必填

```ts
function addServerTemplate (template: NuxtServerTemplate): NuxtServerTemplate
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `filename` | `string` | `true` | 模板文件名 |
| `getContents` | `() => string \| Promise<string>` | `true` | 返回字符串或解析到字符串的 promise |

**它是四个里唯一把两个字段都标必填的**，也**没有 `src`**（不能从磁盘模板来），描述里那句"以 `options` 对象被调用"与它自己的签名（`()` 无参）**并不一致**。

```ts
addServerTemplate({
  filename: '#my-module/test.mjs',
  getContents () {
    return 'export const test = 123'
  },
})
```

然后服务端里：

```ts [server/api/test.ts]
import { test } from '#my-module/test.js'

export default eventHandler(() => {
  return test
})
```

**注意这两段对不上**：注册时的 `filename` 是 `#my-module/test.mjs`，import 的却是 `#my-module/test.js`。页面没有其他说明，**照抄会踩空，实际写法应以注册的那个文件名为准**。

另外这一节的 Usage 与 Example 两段代码**逐字相同**（同一份示例出现两次）。

## `updateTemplates`：一个 filter

```ts
async function updateTemplates (options: UpdateTemplatesOptions): void
```

**重新生成匹配过滤条件的模板；不提供过滤条件时所有模板都会被重新生成**。

官方示例是"页面目录变化时重算路由模板"：

```ts
import { defineNuxtModule, updateTemplates } from '@nuxt/kit'
import { resolve } from 'pathe'

export default defineNuxtModule({
  setup (options, nuxt) {
    const updateTemplatePaths = [
      resolve(nuxt.options.srcDir, 'pages'),
    ]
    // watch and rebuild routes template list when one of the pages changes
    nuxt.hook('builder:watch', async (event, relativePath) => {
      if (event === 'change') {
        return
      }

      const path = resolve(nuxt.options.srcDir, relativePath)
      if (updateTemplatePaths.some(dir => path.startsWith(dir))) {
        await updateTemplates({
          filter: template => template.filename === 'routes.mjs',
        })
      }
    })
  },
})
```

这个示例有两个可以借用的判断：**`event === 'change'` 被直接跳过**（只有增删这类事件才重算，因为文件内容变化不影响"有哪些页面"）、**用 `path.startsWith(dir)` 限定目录**、**`filter` 按 `template.filename` 精确命中**。

## 四个函数的选择

| 你要产出的东西 | 用哪个 | 关键点 |
| --- | --- | --- |
| 应用侧能 import 的虚拟文件 | `addTemplate` | **不写 `dependsOn` 就每次重算**；要落盘用 `write: true` |
| 只给用户项目补类型 | `addTypeTemplate` | **默认只进 Nuxt 上下文**，服务端要 `{ nitro: true }` |
| 服务端能 import 的虚拟文件 | `addServerTemplate` | 只有 `filename`＋`getContents` 两个必填 |
| 让已注册的模板重算 | `updateTemplates` | 不给 filter 就全部重算 |

这一节没写的：`addTemplate` 返回的 `ResolvedNuxtTemplate` 有何用途、`dependsOn` 里 `'plugins'` 具体指哪一类变化、`addTypeTemplate` 为什么不接受 `write`（它本身就写进 buildDir）、以及模板注册了同名 `filename` 时会不会互相覆盖。

下一篇讲 app-config 那一个函数，顺带把四种合并方向摆成一张表。

## 下一篇

下一篇 655：updateAppConfig：一个函数与四种配置合并方向的对照

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/templates
