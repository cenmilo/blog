---
url: /posts/b134beac720cdd8c0efbe0bb06deba71/
title: 682 - 自定义模板生成实战：把模块选项变成一个可 import 的文件
date: 2027-05-27T10:00:00+08:00
updated: 2027-05-27T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：把两节文档接到一起，就得到一条完整链路：**用 `addTemplate` 生成一个文件（`filename` ＋ `getContents`）→ 运行时从 `#build/那个文件名` import 它 → 用 `dependsOn` 告诉 Nuxt 它读什么，否则每次文件变化都会重算 → 需要类型时用 `addTypeTemplate`（不给 `context` 就只进 Nuxt 上下文，服务端要用得写 `{ nitro: true }`）→ 服务端版本换 `addServerTemplate`，它只有 `filename` 与 `getContents` 两个必填**。官方在模板那一节给的插件示例里还留了一句很实用的注释：`// @ts-expect-error - virtual file`，**因为生成的文件不在类型系统视野内**；而布局那一节的警告则说明另一类边界——**`@vitejs/plugin-vue` 不支持虚拟 `.vue` 文件，得靠 `write: true` 落盘**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模板生成
  - 虚拟文件
  - 模块作者
  - 类型声明
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

模块最实用的一招，是把"用户在你这里配的东西"变成一个能被 import 的文件。这一篇不引入新函数，而是把两处文档里散着的片段接成一条能跑通的路。

## 整条链路四步

| 步 | 动作 | 用什么 |
| --- | --- | --- |
| 1 | 生成文件 | `addTemplate({ filename, getContents })` |
| 2 | 使用它 | 运行时 `import ... from '#build/那个文件名'` |
| 3 | 控制重算 | `dependsOn`，或 `builder:watch` ＋ `updateTemplates` |
| 4 | 补类型 | `addTypeTemplate`（服务端要 `{ nitro: true }`） |

第 1、2 步合起来就是模板页那条定性：**模板让你在开发期与构建期生成额外文件；这些文件会出现在虚拟文件系统里，可以在插件、布局、组件等处使用**。

## 第一步：生成

模板页给的官方例子是"把合并后的 head 配置写成一个模块"：

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

三处要点都是可以直接套的：**`defu` 的用途是把用户写的 `app.head` 放在前面、模块默认值垫在后面**（用户能覆盖）；**`getContents` 返回的是"一段源代码文本"**，所以要用 `'export default ' + JSON.stringify(...)` 手工拼；**`filename` 是最终在 `#build` 下的名字**。

这里没有写 `dependsOn`，**而模板页明确说过：什么都不声明的模板会在每一次变化时重新生成**——下一节处理它。

## 第二步：使用

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

四件事都在这一段里：**导入面是 `#build/meta.config.mjs`**（模板页的注释也是这么写的：**文件被加入 Nuxt 内部虚拟文件系统，可以从 `#build/my-module-feature.mjs` 导入**）；**`// @ts-expect-error - virtual file` 是官方自己加的**——**生成的文件没有类型，得先压掉这一行报错**；**`defineNuxtPlugin` 从 `#imports` 显式导入**（这与"发布模块的 runtime 不启用自动导入"那条警告一致）；**用 `import.meta.server` 分支选 createHead 实现**，而不是靠插件 mode 区分。

**要拿掉那行 `@ts-expect-error`，就走第三步。**

## 第三步：让它别每次都重算

模板页那句解释是理解成本所在：**Nuxt 默认会在每个被监听文件事件上重新编译模板，因为它无法知道模板读了什么**。三种声明方式各对应一种情况：

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

对应关系是：**只用配置 → `[]`；读页面清单 → `'pages'`；自己扫未知来源（示例举的是 YAML）→ 函数**。**注意第二例里 `getContents` 解构出 `{ app }`**——这就是"应用的解析结构会作为数据传进来"的形态，它和 `dependsOn: ['pages']` 配套。

如果粒度还不够，就自己接变化事件：

```ts
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
```

这三行判断的顺序值得照抄：**先跳过 `change` 事件（内容变了不代表清单变了）**、**再把相对路径还原成绝对路径后比对目录**、**最后用 `filter` 按 `filename` 精确命中要重算的那个模板**。**`updateTemplates` 不给 `filter` 会重算全部模板**，这个默认值在这条链上代价很大。

`builder:watch` 本身在钩子表里也是**唯一把"只在开发期"写进描述**的那一行，所以这段代码不参与生产构建。

## 第四步：补类型

模板页给的第一个 `addTypeTemplate` 例子就是解决"`import '*.md'` 没有类型"这件事：

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

它的定性是：**把给定模板渲染到项目 buildDir，然后把它登记为类型**——**登记方式是在生成的 `nuxt.d.ts` 里加引用**（高级配方页那句说得更完整：既写模板到磁盘，又往 `nuxt.d.ts` 加一条引用）。

要覆盖服务端上下文，得显式给 `context`：

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

**省略第二个参数时类型只进 Nuxt 上下文**，服务端 `import type { User } from '#auth-utils'` 就认不出来。

需要更细的控制，就绕开 `addTypeTemplate`，自己接类型准备的时机：

```ts
const template = addTemplate({ /* template options */ })
nuxt.hook('prepare:types', ({ references }) => {
  references.push({ path: template.dst })
})
```

**这里推的是 `template.dst`（目标路径），不是 `filename`**——`addTemplate` 有返回值，就是为了在这种地方拿到解析后的路径。

## 服务端的模板是另一个函数

高级配方页对这一处写得很干脆：**服务端要用 `addServerTemplate`**，它的注释也标了导入面与客户端不同：

```ts
addServerTemplate({
  filename: '#my-module/test.mjs',
  getContents () {
    return 'export const test = 123'
  },
})
```

**两个字段都是必填，没有 `src`**（不能从磁盘模板来）。

**服务端侧的导入面是裸的模块标识**（页面上的注释是"可以被服务端代码从 `my-server-module.mjs` 导入"），而客户端侧是 `#build/` 前缀。**但模板页自己那段示例前后不一致**：注册写作 `#my-module/test.mjs`，紧接的服务端文件却 `import { test } from '#my-module/test.js'`——**扩展名不同**。这条只能以自己的 `filename` 为准。

## 两类东西不能留在虚拟文件系统

**第一类是 `.vue`**：布局那一节的警告写得很明确——**`@vitejs/plugin-vue` 不支持虚拟 `.vue` 文件，可以把 `write: true` 传给 `addLayout` 的第一个参数来绕过这个限制**。

**第二类是你需要在磁盘上看到的东西**：`write` 的语义就是"为 `true` 时写入目标文件，否则只存在于虚拟文件系统"。排查"我生成的内容不对"这类问题，**先临时开 `write: true` 去 `buildDir` 里看生成结果**是最快的自检办法。

`addTypeTemplate` 没有 `write` 字段——因为它本身就写到 buildDir；而它也没有 `dependsOn`，这两点在两节的字段表里是明确的差别。

## 五步自检

1. `getContents` 返回的字符串能否被当作模块解析（`export default` 前缀有没有拼）。
2. `filename` 是否与导入面匹配：客户端 `#build/文件名`、服务端按注册的标识。
3. 有没有声明 `dependsOn`——**没声明就是每次变化都重算**。
4. 类型的覆盖面：需要进服务端的，`addTypeTemplate` 第二个参数给 `{ nitro: true }`。
5. 若生成的是 `.vue` 或需要肉眼检查，**开 `write: true`**。

这三处仍是文档缺口，做决定时不要当有依据：**同名 `filename` 被两个模块注册时的行为**（覆盖还是报错都没写）、**`dependsOn: ['plugins']` 具体指哪一类变化**、以及 `#build` 与 `filename` 的映射在带子目录时（示例里有 `my-module/config.mjs` 这种带路径的名字）如何解析。

下一篇把"发布模块的 runtime 里自动导入会失效"这件事讲透：为什么必须写成 `#imports` 的显式导入。

## 下一篇

下一篇 683：模块里的自动导入陷阱：#imports 显式写法

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/templates
- https://nuxt.com/docs/4.x/guide/modules/recipes-advanced
- https://nuxt.com/docs/4.x/api/kit/layout
