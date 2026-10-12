---
url: /posts/3c1723de6b5ff50f42b78a1f70cc7a7c/
title: shared：唯一能被两侧同时导入的位置，以及它的禁令
date: 2026-11-03T09:00:00+08:00
updated: 2026-11-03T09:00:00+08:00
author: cmdragon

summary:
  摘要：shared/ 目录用来在 Vue 应用与 Nitro 服务器之间共享功能，官方给了两条前置事实：这个目录在 Nuxt v3.14+ 才可用，以及一条 important 级别的禁令——shared/ 里的代码不能导入任何 Vue 或 Nitro 代码。原因在"Why You Cannot Mix Vue and Nitro Code"那一节：Nuxt 构建两个独立的包——Vue 应用（客户端与服务端渲染）与 Nitro 服务器（API 路由、服务端中间件、服务端插件），它们各自独立打包并运行在不同上下文里，而 shared/ 里的代码会同时被用在两个包里，所以它两边都不能导入。把 Vue 侧代码引进 Nitro 的后果是组件与组合式函数需要 Vue 应用运行时、通常还需要 Nuxt 上下文（例如 useNuxtApp() 或 useRoute()），而这些在 Nitro 里都不存在，可能引发构建或运行时错误并把 Vue 应用的依赖拉进你的服务端包；反过来把服务端专用代码（Node API、Nitro 工具、服务端路由处理器）引进应用可能破坏客户端构建、在浏览器造成运行时错误，或把服务端逻辑泄漏进客户端包。至于 import type 跨边界"看起来能用"，官方说得很直：类型在编译期被擦除不会把运行时代码拉进另一个包，所以看起来也许可行，但仍要把共享类型放在 shared/types/，因为那里在两个上下文里都被自动导入。

categories:
  - 前端开发

tags:
  - nuxt4
  - shared
  - 边界
  - 自动导入
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


065 那条"两侧不许互引"的禁令，官方把解释留在了这一页。**`shared/` 是唯一的合法共用位置——但它自己有条更严的禁令。**

## 两条前置事实

> 使用 shared/ 目录来**在 Vue 应用与 Nitro 服务器之间共享功能。**

> （note）`shared/` 目录**在 Nuxt v3.14+ 可用。**

> （important）**`shared/` 目录里的代码不能导入任何 Vue 或 Nitro 代码。**

**这三句要连起来读，否则会觉得矛盾**：既然它是"给两侧共用"的位置，为什么它自己谁都不能导入？

答案在下一节。

## 为什么不能混：两套包、两个上下文

官方这一节的标题就是问题本身——"Why You Cannot Mix Vue and Nitro Code"：

> Nuxt 构建**两个独立的包**：Vue 应用（客户端与服务端渲染）与 Nitro 服务器（API 路由、服务端中间件、服务端插件）。**它们独立打包并运行在不同的上下文里。`shared/` 目录里的代码会同时用在两个包里，所以它两边都不能导入。**

**这段把 013 那句"同一份代码跑两个地方"精确化了：** 不是"一份代码两个环境"，而是**两个独立构建产物**，只是中间有一段可以重叠。

**两个方向各自的后果，官方分别写了小节。**

**方向一：把 Vue 侧代码引进 Nitro**

> 组件与组合式函数需要 **Vue 应用运行时**，而且通常还需要 **Nuxt 上下文（例如 `useNuxtApp()` 或 `useRoute()`）**——**这两者在 Nitro 里都不存在。** 把它们导入服务端代码**可能导致构建或运行时错误，并可能把 Vue 应用的依赖拉进你的服务端包。**

**注意最后那半句是包体层面的后果**，不只是"报错与否"：`useRoute()` 在中间件里不准（060）、在 Vue 应用之外的上下文里根本不存在（013/065）。

**方向二：把服务端代码引进 Vue 应用**

> 服务端专用代码（例如 **Node API、Nitro 工具，或服务端路由处理器**）绝不应在浏览器里运行。把它导入你的应用**可能破坏客户端构建、在浏览器里造成运行时错误，或把服务端逻辑泄漏进客户端包。**

**"把服务端逻辑泄漏进客户端包"这一句是最该警惕的**——它的后果不是报错，而是**你的密钥、内部调用逻辑被打包进了用户能下载的 JS 里**（对照 055 那条"不要把密钥放进 `app.config`，它会进客户端包"，是同一族风险）。

## `import type` 那一节：官方自己承认"看起来能用"

这一小节写得很诚实，值得整段读：

> `import type` **在编译期被擦除**，不会把运行时代码拉进另一个包，所以**只导入类型跨越这条边界也许看起来是能工作的。** 尽管如此，**请把共享类型（例如 API 响应类型）保留在 `shared/types/`，那里在两个上下文里都被自动导入。** 这样能保持边界清晰、避免以后把一个类型导入不小心变成一次值导入，并且符合 Nuxt 为 app、server 与 shared 代码准备的、相互独立的[类型上下文]。

**官方在这里给了三条"为什么不图省事"的理由：**

1. **边界清晰**；
2. **避免日后误改成值导入**（`import type` 去掉 `type` 就成了真导入，那时候才炸）；
3. **与项目引用的类型上下文一致**（031、036 那四份 tsconfig 里的 `shared` 就是这一层）。

紧接着一句把三个 types 目录的分工钉住：

> 只在一个上下文用到的类型可以留在那个上下文旁边：**`app/types/` 只在 Vue 应用里自动导入，`server/types/` 只在 Nitro 服务器里自动导入。当一个类型被两边都需要时，请使用 `shared/types/`。**

**这张表就是 063 那条 tip 的完整版：**

| 类型位置 | 谁能自动导入 |
| --- | --- |
| `app/types/` | 只有 Vue 应用 |
| `server/types/` | 只有 Nitro 服务器 |
| `shared/types/` | **两侧** |

## 用法：一个函数，两侧调用

官方给的两种导出形式（与 063 那套一样）：

```ts [shared/utils/capitalize.ts]
export const capitalize = (input: string) => {
  return input[0] ? input[0].toUpperCase() + input.slice(1) : ''
}
```

```ts [shared/utils/capitalize.ts]
export default function (input: string) {
  return input[0] ? input[0].toUpperCase() + input.slice(1) : ''
}
```

> 现在你可以在你的 Nuxt 应用**与 `server/` 目录里**使用自动导入的工具函数。

然后是两个文件的对照——**这才是 `shared/` 的存在价值：**

```vue [app/app.vue]
<script setup lang="ts">
const hello = capitalize('hello')
</script>

<template>
  <div>
    {{ hello }}
  </div>
</template>
```

```ts [server/api/hello.get.ts]
export default defineEventHandler((event) => {
  return {
    hello: capitalize('hello'),
  }
})
```

**同一份格式化逻辑，浏览器里那份渲染与服务端那个接口用的是同一个函数——不会出现"两边规则不一致"。** 这正是 063 结尾那句"要在 `server/api` 用 `app/utils` 是不行的"之后唯一的正确去处。

## 扫描范围：只有 `shared/utils` 与 `shared/types`

这一节的信息密度很高，官方给的规则与那棵树要一起读：

> **只有 `shared/utils/` 与 `shared/types/` 目录里的文件会被自动导入。** 这两个目录的**嵌套子目录**里的文件不会被自动导入，**除非你把这些目录同时加进 `imports.dirs` 与 `nitro.imports.dirs`。**

```bash [Directory Structure]
-| shared/
---| capitalize.ts        # Not auto-imported
---| formatters
-----| lower.ts           # Not auto-imported
---| utils/
-----| lower.ts           # Auto-imported
-----| formatters
-------| upper.ts         # Not auto-imported
---| types/
-----| bar.ts             # Auto-imported
```

**把这五行读成一个清单：**

| 文件 | 自动导入？ | 为什么 |
| --- | --- | --- |
| `shared/capitalize.ts` | **否** | 不在 `utils/` 或 `types/` 里 |
| `shared/formatters/lower.ts` | **否** | 同上 |
| `shared/utils/lower.ts` | **是** | 位于 `shared/utils/` 顶层 |
| `shared/utils/formatters/upper.ts` | **否** | 在 `utils/` 的**嵌套子目录**里 |
| `shared/types/bar.ts` | **是** | 位于 `shared/types/` 顶层 |

**两个门槛叠在一起：既要进对的那两个目录，又只能是顶层。**

官方在这里还确认了一次与 063 的一致性：

> `shared/utils` 与 `shared/types` 的自动导入如何工作与如何扫描，**与 `app/composables/` 与 `app/utils/` 目录完全相同。**

**注意那句"除非你把这些目录同时加进 `imports.dirs` 与 `nitro.imports.dirs`"里的"同时"：**

- 063 给的例子只改了 `imports.dirs`（应用侧）；
- 这里官方要求**两侧各配一遍**——因为 `shared/` 是被两套构建各自消费的，配置项也是两套（Nitro 那份在 `nitro.imports.dirs` 下）。

**只配一边会出现的症状很典型：页面里能用、`server/api` 里报找不到（或反过来）。**（这个因果是我按官方"同时"二字推出来的。）

## 其余文件怎么引：`#shared` 别名

> 你在 `shared/` 文件夹里创建的其它任何文件，**必须使用 `#shared` 别名手动导入**（由 Nuxt 自动配置）：

```ts [Importing From #shared]
// For files directly in the shared directory
import capitalize from '#shared/capitalize'

// For files in nested directories
import lower from '#shared/formatters/lower'

// For files nested in a folder within utils
import upper from '#shared/utils/formatters/upper'
```

> 这个别名确保了你整个应用里一致的导入，**无论导入方文件在哪里。**

**`#shared` 你也见过两次：051 那张默认别名表里 `"#shared": "/<rootDir>/shared"`，063 那张边界表里它是"两侧共用"的引用入口。**

于是这一节的取舍很干净：

| 需求 | 做法 |
| --- | --- |
| 一个通用工具函数，两侧都要用、且不想写 import | 放 `shared/utils/` **顶层** |
| 一个共享类型 | 放 `shared/types/` **顶层** |
| 想按目录细分组织（`shared/utils/formatters/upper.ts`） | **写 `import upper from '#shared/utils/formatters/upper'`**（要么就配那两条 dirs） |
| 只想给一侧用的东西 | 别放 `shared/`——放 `app/utils/` 或 `server/utils/` |

## 自查

- [ ] `shared/` 需要什么版本以上才可用？它自己那条 important 禁令是什么？
- [ ] 为什么它两边都不能导入？官方的理由是"两个包"的哪两个性质？
- [ ] 把服务端代码引进应用，官方列的三种后果里最危险的是哪一个？
- [ ] `shared/utils/formatters/upper.ts` 会被自动导入吗？两种解决办法分别是什么？
- [ ] 想让嵌套目录也自动导入，官方要求改哪两个配置？只改一个会怎样？
- [ ] `app/types/`、`server/types/`、`shared/types/` 各自被谁自动导入？

## 下一篇

067 讲 `modules/` 目录：本地模块只认哪两种文件形态、为什么应用侧文件必须放进 `runtime/app/` 才拿得到正确类型，以及 `nuxt/kit` 在本地模块里为什么不用额外安装。

## 本文依据

- `directory-structure/shared`（标题：使用 shared/ 目录在 Vue 应用与 Nitro 服务器之间共享功能；shared/ 允许你共享可同时用于 Vue 应用与 Nitro 服务器的代码；note shared/ 目录在 Nuxt v3.14+ 可用；important shared/ 目录里的代码不能导入任何 Vue 或 Nitro 代码；Why You Cannot Mix Vue and Nitro Code：Nuxt 构建两个独立的包——Vue 应用（客户端与服务端渲染）与 Nitro 服务器（API 路由、服务端中间件、服务端插件），它们独立打包并运行在不同上下文，shared/ 里的代码同时用于两个包所以两边都不能导入；Vue App Code in Nitro：组件与组合式函数需要 Vue 应用运行时且通常需要 Nuxt 上下文（例如 useNuxtApp() 或 useRoute()）而这两者在 Nitro 中都不存在，导入服务端代码可能导致构建或运行时错误并把 Vue 应用依赖拉进服务端包；Nitro Code in the Vue App：服务端专用代码（Node API、Nitro 工具或服务端路由处理器）绝不应在浏览器运行，导入应用可能破坏客户端构建、在浏览器造成运行时错误或将服务端逻辑泄漏进客户端包；Type-Only Imports：import type 在编译期被擦除不会把运行时代码拉进另一个包所以只导入类型跨边界也许看起来能工作，尽管如此仍应把共享类型（如 API 响应类型）放在 shared/types/ 因为那里在两个上下文都被自动导入，这样保持边界清晰、避免日后把类型导入误变成值导入，并符合 Nuxt 为 app、server 与 shared 代码准备的独立类型上下文；只在一个上下文用到的类型可放在该上下文旁边——app/types/ 只在 Vue 应用自动导入、server/types/ 只在 Nitro 服务器自动导入，两侧都需要时用 shared/types/；Usage 两种导出形式（shared/utils/capitalize.ts 的具名导出与默认导出）并说明现在可在 Nuxt 应用与 server/ 目录里使用自动导入的工具函数，示例 app/app.vue 中 const hello = capitalize('hello') 与 server/api/hello.get.ts 中返回 {hello: capitalize('hello')}；How Files Are Scanned：只有 shared/utils/ 与 shared/types/ 里的文件会被自动导入，这两个目录的嵌套子目录里的文件不会自动导入除非把这些目录加进 imports.dirs 与 nitro.imports.dirs，tip 其自动导入与扫描方式与 app/composables/ 和 app/utils/ 完全相同，目录树标注 shared/capitalize.ts Not auto-imported、shared/formatters/lower.ts Not auto-imported、shared/utils/lower.ts Auto-imported、shared/utils/formatters/upper.ts Not auto-imported、shared/types/bar.ts Auto-imported；其它在 shared/ 里创建的文件必须使用由 Nuxt 自动配置的 #shared 别名手动导入并给出 #shared/capitalize、#shared/formatters/lower、#shared/utils/formatters/upper 三例，该别名确保无论导入方文件在哪里都能有一致的导入）
- `directory-structure/server`（对照用：important 不要在服务端路由或工具里导入 Vue 应用侧代码也不要在应用里导入服务端专用代码，且 read-more 指向本页的 Why You Cannot Mix Vue and Nitro Code；server/types 只在服务端上下文自动导入且只扫描该目录顶层文件）
- `directory-structure/app/utils`（对照用：utils 只在应用的 Vue 部分可用、只有 server/utils 在 server/ 目录自动导入；类型三处放置 tip；以及 shared/types 作为两侧共享类型的位置）
- `api/nuxt-config`（对照用：默认别名包含 "#shared": "/<rootDir>/shared"；imports.dirs 与 nitro.imports.dirs 分属两侧配置）
