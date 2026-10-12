---
url: /posts/8a935b379ebb03385c1bf1b065f06907/
title: '#imports 类型：显式导入时的补全'
date: 2027-02-23T11:00:00+08:00
updated: 2027-02-23T11:00:00+08:00
author: cmdragon

summary:
  摘要：自动导入可以完全不写 import，但有些场合你必须显式写——而显式写的时候，来源是 `#imports` 这个别名。官方原话："Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed"，示例就是 `import { computed, ref } from '#imports'`。三类必须或最好显式写的场景：发布的模块（`node_modules` 里不启用自动导入，出于性能原因）、把 `imports.autoImport` 设成 `false` 之后（官方明说"disable auto-imports completely but it's still possible to use explicit imports from `#imports`"）、以及想消歧义同名符号时。因为 `#imports` 指向的就是那张自动导入表本身，所以显式导入同样带类型和补全，不会退回"手动引了就没提示"。

categories:
  - 前端开发

tags:
  - nuxt4
  - imports别名
  - 显式导入
  - 自动导入
  - 类型补全
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`imports.autoImport: false` 之后满屏报"找不到 `ref`"，有人以为类型没了、要回到手写 import 的老日子。其实官方在这条设置后面就给了出路：`#imports`。

## `#imports` 是什么

自动导入那页的 Explicit Imports 一节：

> Nuxt exposes every auto-import with the `#imports` alias that can be used to make the import explicit if needed:

```vue
<script setup lang="ts">
import { computed, ref } from '#imports'

const count = ref(1)
const double = computed(() => count.value * 2)
</script>
```

关键词 **every auto-import**：这张表把全部自动导入项都暴露出来，所以从 `#imports` 显式引进来和自动导入用的是**同一个符号来源**——类型、补全都在，不是"降级成手动引"。

## 场景一：关掉 autoImport 之后

官方给的关闭方式与后果：

> If you want to disable auto-importing composables and utilities, you can set `imports.autoImport` to `false` in the `nuxt.config` file.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  imports: {
    autoImport: false,
  },
})
```

紧接着一句是重点：

> This will disable auto-imports completely but it's still possible to use explicit imports from `#imports`.

也就是 `autoImport: false` 关掉的是"自动注入"，**没有**关掉那张表。全项目改成显式风格的人，靠 `#imports` 一样有类型。

## 场景二：发布的模块代码

`node_modules` 里不启用自动导入——官方在模块那页写得很直白：

> Published modules cannot leverage auto-imports for assets within their runtime directory. Instead, they have to import them explicitly from `#imports` or alike.
> Auto-imports are not enabled for files within `node_modules` (the location where a published module will eventually live) for performance reasons.

所以在 `app/composables` 里不 import 就能用的写法，放进要发布的模块 `src/runtime/` 里就会报找不到——必须显式：

```ts
import { useRuntimeConfig } from '#imports'
```

这是写模块时最容易忽略的一条，原因是它在自己项目的 dev 里往往看不出来（154 那条同源）。

## 场景三：`shared/` 用 `#shared`

同类的还有共享目录：`shared/` 里不在自动扫描范围内的文件，要用 `#shared` 别名手动引（066/312）：

```ts
import capitalize from '#shared/capitalize'
import lower from '#shared/formatters/lower'
```

官方说明这个别名"ensures consistent imports across your application, regardless of the importing file's location"——路径不随引用者位置变化。`#imports` 管自动导入那张表，`#shared` 管 `shared/` 目录，是两个不同的口子。

## 顺带一条容易忽略：scan 关掉时 `ref` 还在

有人以为"想只停掉自己代码的自动导入"要关 `autoImport`，其实官方给的是更细的一档：

> If you want framework-specific functions like `ref` to remain auto-imported but wish to disable auto-imports for your own code (e.g., custom composables), you can set the `imports.scan` option to `false`.

`scan: false`＝框架的（`ref` 这些）继续自动导入，只是不扫你自己的 `app/composables`、`app/utils`。要"半自动"用这个，别用 `autoImport: false`。

## 自查

- `#imports` 暴露**全部**自动导入项，显式导入同样带类型与补全。
- `imports.autoImport: false` 关掉自动注入但**不影响** `#imports` 显式导入。
- 发布模块必须显式引（`node_modules` 内不启用自动导入，性能原因）。
- `shared/` 用 `#shared`，与 `#imports` 是两个不同别名。
- 只想停自有代码、保留 `ref` 这类框架函数 → `imports.scan: false`，不是 `autoImport: false`。

## 下一篇

下一篇 405 谈组件的 props 与 emits 在 Nuxt 文档里的真实覆盖范围：哪些地方官方明确说"完全定型"（布局 props、页面自定义 props、error 页），哪些属于 Vue 文档、Nuxt 页只当编译器宏类比。

## 本文依据

- "Nuxt exposes every auto-import with the `#imports` alias"与 `import { computed, ref } from '#imports'` 示例、`imports.autoImport: false` 及"disable auto-imports completely but it's still possible to use explicit imports from `#imports`"、`imports.scan: false`（框架函数仍自动导入、只停自有代码）（`guide/concepts/auto-imports`）；发布模块不能用自动导入、须从 `#imports` 显式引与 `node_modules` 因性能不启用自动导入（`guide/modules/module-anatomy`）；`#shared` 别名与"consistent imports regardless of the importing file's location"（`directory-structure/shared`）。
