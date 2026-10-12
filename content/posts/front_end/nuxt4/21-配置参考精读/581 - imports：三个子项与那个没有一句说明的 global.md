---
url: /posts/cea7a00e72ac97a4e4903c49a5d5f144/
title: imports：三个子项与那个没有一句说明的 global
date: 2027-04-23T11:00:00+08:00
updated: 2027-04-23T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## imports` 的定性句是 "Configure how Nuxt auto-imports composables into your application."，下面**三个子项**：`dirs`（`array`、无默认值、**官方明写它不会覆盖默认目录**）、`scan`（`boolean`、默认 `true`，只管 `app/composables/` 与 `app/utils/`）、以及 `global`——**`global` 这一条只有 `**Type**: boolean` 和 `**Default:** false` 两行，一个字的说明都没有**。另外两个在别的页被当成 `imports` 的子键使用的东西（`presets`、`autoImport`）**在这一节里根本没有条目**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - composable
  - 扫描目录
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

把 `stores/` 里的 Pinia store 也变成不用 import 就能用，或者干脆停掉组合式函数的自动扫描——两件事都在 `imports` 这一节。但这一节的三个子项里，有一个连说明都没有。

## 节本身与三个子项

`## imports` 只有一句定性：**"Configure how Nuxt auto-imports composables into your application."**，加一条 See 指向组合式函数目录那一页。**这一节自己没有 Type 行、没有 Default 行。**

| 子项 | Type | Default | 说明句 |
| --- | --- | --- | --- |
| `dirs` | `array` | **无默认值行** | "An array of custom directories that will be auto-imported. **Note that this option will not override the default directories** (~/composables, ~/utils)." |
| `global` | `boolean` | `false` | **完全没有说明句** |
| `scan` | `boolean` | `true` | "Whether to scan your `app/composables/` and `app/utils/` directories for composables to auto-import. Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled." |

`global` 那一行是这个系列里少见的**零记载条目**：有类型、有默认值，**用途、影响面、和 `components.global` 的关系，一个字都没写**。整套 4.x 文档里也找不到别处解释这个键。要判断它能不能满足你的需求，只能读类型或实测——**不要按名字猜它等于"把所有 composable 变成全局注册"**。

## dirs：它是加法，条目还能写 glob

`dirs` 那句提醒是这一节最容易被忽略的一条规则：**这一项不会覆盖默认目录**（`~/composables`、`~/utils`）。也就是说写了 `dirs`，那两个默认目录**照旧被扫**，你的条目是加上去的。想"只扫我列的目录"，这一项做不到。

官方示例给的是最常见的用法：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    // Auto-import pinia stores defined in `~/stores`
    dirs: ['stores'],
  },
})
```

组合式函数目录那一页还补了**条目可以写 glob**，并给了三种形状：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    dirs: [
      // Scan top-level composables
      '~/composables',
      // ... or scan composables nested one level deep with a specific name and file extension
      '~/composables/*/index.{ts,js,mjs,mts}',
      // ... or scan all composables within given directory
      '~/composables/**',
    ],
  },
})
```

这三行分别对应：**只扫顶层**、**扫嵌套一层的特定入口文件**（名字与扩展名都要对上）、**扫该目录下全部**。那一页同时给了官方更推荐的做法——**在 `app/composables/index.ts` 里再导出（re-export）需要的东西**，示例是 `export { utils } from './nested/utils.ts'`，注释写着"Enables auto import for this export"。**配置里加 glob 是第二条路，不是第一条。**

顺带记一处口径差：`dirs` 那句把默认目录写成 **`~/composables`、`~/utils`**（`~` 别名视角），而 `scan` 那句写的是 **`app/composables/`、`app/utils/`**（srcDir 视角）。两处说的是同一批目录，只是写法不同——**如果你的 `srcDir` 不是默认的 `app`，这两句给你的指向并不一样清楚**，要以 `srcDir` 为准去核对实际路径。

## scan：关掉只影响你这两个目录

`scan` 默认 `true`。关掉之后的范围由那句话限定得很清楚：

- 停止扫描的是 **`app/composables/` 与 `app/utils/`**。
- **"Auto-imports registered by Nuxt or other modules, such as imports from `vue` or `nuxt`, will still be enabled."** —— 框架自己注册的、模块注册的自动导入**不受影响**。

所以 `scan: false` 不是"关掉自动导入"，而是"不再从你自己的这两个目录里收集名字"。真要完全关掉自动导入，文档里出现的写法是另一个键：`imports: { autoImport: false }`（这句来自 Nuxt 3 兼容 API 那一页，配套说明是"这会完全禁用自动导入，但显式 import 仍然可用"）。

**而 `autoImport` 在配置参考这一节里没有条目**——三个子项之外，它和 `presets` 都属于"别处在用、这节不载"的键。

## presets：同样不在这节里，但被当子键使用

自动导入概念页讲第三方包时，给的就是 `imports` 下面的 `presets`：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    presets: [
      {
        from: 'vue-i18n',
        imports: ['useI18n'],
      },
    ],
  },
})
```

用法句是 "Nuxt also allows auto-importing from third-party packages."，另有一条 `::tip` 提醒：**如果你用的包有 Nuxt 模块，多半模块已经替你配好了这些自动导入。**

`presets` 的形状在这里只出现这一处，`from` ＋ `imports` 两个字段。**其余字段（比如别名、类型、`ignore`）配置参考里没有记载。**

## 自查

- `## imports` 只有一句定性 ＋ 一条 See，**节自身无 Type / Default**。
- 三个子项里 **`global` 零说明**（只有 `boolean` ＋ `false`），不要按名字推断它的行为。
- `dirs` **不会覆盖默认目录**，是加法；条目支持三种 glob 形状，但官方更推荐 `app/composables/index.ts` 里 re-export。
- `dirs` 句写 `~/composables`／`~/utils`，`scan` 句写 `app/composables/`／`app/utils/`；改了 `srcDir` 时要自己核对指向。
- `scan: false` 只停你这两个目录的扫描，**框架与模块注册的自动导入照常**；要彻底关是 `imports.autoImport: false`（该键本节无条目）。
- `presets`（`from` ＋ `imports`）与 `autoImport` 都在别处被当作 `imports` 的子键使用，**配置参考本节一个都没列**。

## 下一篇

下一篇 582 看 `## pages`：它的说明句**还写着 "in Nuxt 3"**，没有类型行也没有默认值，只给了一条 `pattern` 示例；而"省略不写时按什么决定开不开"这句里，条件指的是 `app/pages/` 目录存在与否。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## imports` 定性句与 See；`### dirs` 说明句、`array` 无默认、`stores` 示例；`### global` 仅 Type 与 Default 两行；`### scan` 说明句、`boolean`、`true`）
- https://nuxt.com/docs/4.x/directory-structure/app/composables（嵌套目录两条路：re-export 推荐语与 `export { utils }` 示例、`dirs` 三种 glob 形状示例）
- https://nuxt.com/docs/4.x/guide/concepts/auto-imports（"Nuxt also allows auto-importing from third-party packages."、`imports.presets` 的 `vue-i18n`／`useI18n` 示例、那条 `::tip`）
- https://nuxt.com/docs/4.x/bridge/nuxt3-compatible-api（`imports: { autoImport: false }` 与"完全禁用自动导入、显式 import 仍可用"的说明）
