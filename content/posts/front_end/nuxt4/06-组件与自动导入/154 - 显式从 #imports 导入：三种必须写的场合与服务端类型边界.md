---
url: /posts/41da0dc442457f300392bece62b5c1c7/
title: 154 - 显式从 #imports 导入：三种必须写的场合与服务端类型边界
date: 2026-12-02T10:00:00+08:00
updated: 2026-12-02T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方给显式导入开的口子只有一句定义："**Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed**"，注意范围词是 **every**。但有三类场合这不是风格选择，而是必须写。**其一，发布出去的模块**：模块结构那一节用 `::warning` 写着"**Published modules cannot leverage auto-imports for assets within their runtime directory. Instead, they have to import them explicitly from `#imports` or alike.**"，给出的原因是"**Auto-imports are not enabled for files within `node_modules` (the location where a published module will eventually live) for performance reasons.**"——所以官方自己的模块示例里出现的是 `import { defineNuxtPlugin } from '#imports'`。**其二，把自动导入关掉的应用**：`imports.autoImport: false` 之后"**This will disable auto-imports completely but it's still possible to use explicit imports from `#imports`.**"。**其三，服务端代码要用 app 侧的实现**：模块服务端兼容指南里那对 legacy 文件写着 `// @ts-expect-error \`#imports\` is typed for the app, not the server build` + `import { useRuntimeConfig } from '#imports'`，并在同一页的表里把 `nitro2` 变体的导入面定义为"`h3`, `nitropack/runtime`, **`#imports`**"——这一条同时给出**类型边界**：`#imports` 的类型是为 app 上下文生成的。相关的另一句硬事实是"**In playgrounds and test fixtures on Nuxt 5, h3's auto-imported `defineEventHandler` is not available: import it from `nuxt/server`.**"。而类型这套东西能成立，靠的是生成的 tsconfig 引用了"references to auto-imports, API route types, path aliases like `#imports`, `~/file`, or `#build/file`"。

categories:
  - 前端开发

tags:
  - nuxt4
  - imports
  - 显式导入
  - 模块
  - TypeScript

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我一直把 `#imports` 当成"洁癖选项"——想让文件头看得见依赖就写。真正动手做模块之后才发现，官方文档里它有三种场合是**不写就跑不起来**的，其中一种还带着一行 `@ts-expect-error`。

## 这个别名覆盖什么

官方定义句只有一句，但范围词很满：

> Nuxt exposes **every** auto-import with the `#imports` alias that can be used to make the import explicit if needed:

```vue [app/pages/index.vue]
<script setup lang="ts">
import { computed, ref } from '#imports'

const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

也就是说：**任何一个能免写 import 的名字，都能从 `#imports` 显式导**。它与另外几个井号别名的分工，105 那张表已经立过——`#imports` 是自动导入名单的出口，`#build` 指向构建目录里的生成文件，`#shared` 指向 `shared/` 目录。

## 场合一：发布的模块必须显式导入

模块结构那一节给的警告，是这一篇最硬的依据：

> **Published modules cannot leverage auto-imports for assets within their runtime directory. Instead, they have to import them explicitly from `#imports` or alike.**
>
> Auto-imports are not enabled for files within `node_modules` (the location where a published module will eventually live) **for performance reasons.**

两句合起来是一个因果链：

| 事实 | 后果 |
| --- | --- |
| 自动导入**不为 `node_modules` 里的文件启用** | 发布到 npm 的模块，它的 `runtime/` 代码在被安装后正位于 `node_modules` |
| 所以模块的 runtime **不能指望自动导入** | 必须显式导入自己用到的东西，以及框架给的东西 |
| 原因官方归到"性能" | 这不是可以绕的临时限制，是设计选择 |

官方模板生成器的示例就是这种写法的实拍：

```ts [runtime/plugin.ts]
import { createHead as createServerHead } from '@unhead/vue/server'
import { createHead as createClientHead } from '@unhead/vue/client'
import { defineNuxtPlugin } from '#imports'
// @ts-expect-error - virtual file
import metaConfig from '#build/meta.config.mjs'
```

注意这五行里三种来源并存：包（`@unhead/vue/*`）、**框架的自动导入出口 `#imports`**、构建目录里的虚拟文件 `#build/...`（它还得配 `@ts-expect-error - virtual file`，因为文件是运行时才生成的）。

**那句"or alike"也别略过**：组件的对应出口是 `#components`（官方组件目录页原文："You can also explicitly import components from `#components` if you want or need to **bypass Nuxt's auto-importing functionality**."，161 展开）。

顺带一条本地模块的对照：本地模块（`modules/` 目录里或配置里内联的）不在 `node_modules`，官方把模块分成 published 与 local 两类、并说"In either case, they work in the same way"——**但自动导入这条限制是按文件位置生效的**，所以"我的模块放进仓库就能享受自动导入"这个直觉只对 `modules/` 下的模块定义文件成立，它的 runtime 目录一旦被打包安装就不成立了。

## 场合二：把自动导入关掉之后

152 引过的那句在这里第二次出现：

> This will disable auto-imports completely but it's still possible to use **explicit imports** from `#imports`.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    autoImport: false,
  },
})
```

这条是**用户主动选的关闭**，与场合一那条"框架不给你自动导入"是两回事。但落点相同：`#imports` 是关不掉的那条路。

## 场合三：服务端要用 app 侧实现，且类型是错位的

模块的服务端兼容指南给了一对文件（可移植版与 legacy 版），差异全在导入行上。官方原话：

> The two files differ in their imports, and wherever the handler uses an h3 v1 helper or behavior that `nuxt/server` handles differently:

```ts [runtime/server/status.ts]
import { defineEventHandler, useRuntimeConfig } from 'nuxt/server'

export default defineEventHandler(() => ({
  version: useRuntimeConfig().public.myModule.version,
}))
```

```ts [runtime/server/status.legacy.ts]
// @ts-expect-error `#imports` is typed for the app, not the server build
import { useRuntimeConfig } from '#imports'
import { defineEventHandler } from 'h3'

export default defineEventHandler(event => ({
  version: useRuntimeConfig(event).public.myModule.version,
}))
```

**这一例把"`#imports` 的边界"讲得比任何说明都清楚**，三处细节都对得上前面几篇：

| 细节 | 说明 |
| --- | --- |
| `import { useRuntimeConfig } from '#imports'` | 服务端要拿 app 侧那个实现时，走 `#imports` |
| 那行 `@ts-expect-error` 的原文 | **"`#imports` is typed for the app, not the server build"** ——类型上下文是分开的（155 那四个 tsconfig 就是这件事的具体形状） |
| `useRuntimeConfig(event)` 带了参数 | 与可移植版 `useRuntimeConfig()` 不同；这正是 151 那句"服务端同名函数行为在若干处不同"的一个实例 |

官方那一节把三种变体的导入面列成表，`#imports` 明确只属于 `nitro2`：

| Key | The file imports | Where it runs |
| --- | --- | --- |
| `nuxt` | `nuxt/server` only | Any server builder, from Nuxt 4.6 |
| `nitro2` | `h3`, `nitropack/runtime`, **`#imports`** | `nitropack` v2 |
| `nitro3` | `nitro`, `nitro/h3`, alone or alongside `nuxt/server` | The Nitro server builder, v3 |

同一页还有一句"自动导入在某个环境里就是不给你"的硬事实：

> **In playgrounds and test fixtures on Nuxt 5, h3's auto-imported `defineEventHandler` is not available: import it from `nuxt/server`.**

**这也是一条测试环境的陷阱**：dev 里跑得好好的自动导入，在 playground/fixture 里直接不存在。写测试时按官方这句话显式导入，别指望两边一致。

## 为什么这些别名有类型

`typescript` 那一节解释了生成的 tsconfig 里都有什么：

> The generated `tsconfig.json` files inside the `.nuxt` directory include **recommended basic TypeScript configuration** for your project, **references to auto-imports, API route types**, path aliases like **`#imports`**, `~/file`, or `#build/file`, and more.

模块侧还有一句同源的补充（讲的是服务端类型）：

> Builder-specific types, such as `H3Event` for `useRequestEvent()` or the full Nitro config in the `nitro:config` hook, come from the generated `.nuxt/nuxt.d.ts`. **Without it, they resolve to builder-agnostic types** (`RequestEvent` and a reduced Nitro config).

**这解释了 `@ts-expect-error` 为什么会必要**：类型来自生成的那几个文件，而不是别名本身有魔法。生成的东西缺了，类型就退化成通用形状。

## 一张"该从哪写导入"的表

| 你在写的文件 | 推荐导入方式 | 依据 |
| --- | --- | --- |
| `app/` 下的页面、组件、composables | 免写 import（自动导入） | "every function that is not explicitly imported is auto-imported" |
| 想让依赖可见 / 已关掉自动导入 | `from '#imports'` | "still possible to use explicit imports from `#imports`" |
| 发布模块的 `runtime/` 里的 app 侧代码 | **必须**显式 `from '#imports'` | module-anatomy 的 warning |
| 发布模块 `runtime/server/`（要兼容 nitropack v2） | `from '#imports'` + `@ts-expect-error` 那行 | server-compatibility 的 legacy 示例 |
| 新的可移植服务端代码（Nuxt 4.6+） | `from 'nuxt/server'`，且不自动导入 | "These names are not auto-imported from `nuxt/server`" |
| 模块定义文件（`setup` 里） | `from '@nuxt/kit'` | 所有官方模块示例；且不要在那里导入 `nuxt/server`（151） |
| 绕过组件自动注册 | `from '#components'` | components 目录页 Direct Imports（161） |

## 自查

1. 官方给 `#imports` 的定义句里，哪个词说明了覆盖范围？它与 `#build`、`#shared` 的分工分别是什么？
2. 为什么发布模块的 runtime 不能用自动导入？官方给的原因是哪句话里的哪个词？
3. "or alike" 在组件一侧指的是哪个别名？官方那句原文怎么说？
4. `imports.autoImport: false` 与"模块在 `node_modules` 里没有自动导入"是同一件事吗？差别在哪？
5. legacy 服务端文件为什么要加那行 `@ts-expect-error`？注释原文怎么说？
6. 同一份逻辑，可移植版与 legacy 版的 `useRuntimeConfig` 调用有什么写法差异？
7. 三种 handler 变体各自的导入面是什么？`#imports` 出现在哪一行？
8. 官方说在哪个环境下 h3 的自动导入 `defineEventHandler` 不可用？该改成什么？
9. 服务端那些 builder 特定类型从哪个生成文件来？缺了它会怎样？

## 下一篇

155 讲自动导入背后的类型系统：`nuxt prepare` 生成什么、四个 tsconfig 上下文各自管哪片代码、以及为什么类型增强必须放对目录。
