---
url: /posts/bf553efebbe003620aa9717c02c75376/
title: 063 - composables 与 utils：自动导入的两条来源与它们的边界
date: 2026-11-02T09:00:00+08:00
updated: 2026-11-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：app/composables/ 用来自动导入你的 Vue 组合式函数，app/utils/ 用来自动导入工具函数，官方明确这两个目录的自动导入工作方式与扫描规则完全相同，utils 存在的唯一目的是让你的 Vue 组合式函数与其它工具函数之间有一个语义上的区分。名字来源有两种：具名导出直接用它导出的名字，默认导出则"作为文件名去掉扩展名后的 camelCase 形式可用"（use-foo.ts 或 useFoo.ts 都以 useFoo() 可用）。三条边界最容易撞：一是这两个目录不会给你的代码提供任何额外的响应式能力，响应式来自 Vue 的 Composition API（ref 与 reactive），而且响应式代码并不限于这两个目录；二是它们只在应用内被扫描顶层文件（nested/utils.ts 不被搜索），要嵌套要么从 index.ts 再导出一层（官方标注 recommended），要么用 imports.dirs 配置三种 glob；三是 app/utils 里的东西只在你应用的 Vue 那一部分可用，只有 server/utils 会在 server/ 目录里被自动导入。类型上官方给了对应的三处放置位置：应用专用类型放 app/types/、服务端专用放 server/types/、两侧共享放 shared/types/。

categories:
  - 前端开发

tags:
  - nuxt4
  - 自动导入
  - composables
  - utils
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

061 那节我抄过官方一句建议，强烈推荐改用 composables，而不是提供 helper。这一篇就讲这个被推荐的位置本身，`app/composables/` 与 `app/utils/`。

先给结论，这两条目录的规则是一样的，官方在 utils 页用 tip 写死了这句话。所以这一篇讲一套规则加一条差别。

官方给的两个定位。使用 composables/ 目录来自动导入你的 Vue 组合式函数进你的应用。使用 utils/ 目录来在你的整个应用里自动导入你的工具函数。`app/utils/` 目录的主要目的是在你的 Vue 组合式函数与其它自动导入的工具函数之间，允许一个语义上的区分。

读准这句主要目的，`utils/` 没有额外的技术能力，它只是把你的函数分成两类放。官方在 utils 页的 tip 里再确认了一遍，`app/utils/` 的自动导入如何工作、以及如何扫描，与 `app/composables/` 目录完全相同。

名字从哪来，两种导出方式。

第一种是具名导出，导出的名字就是要用的名字。

```ts [app/composables/useFoo.ts]
export const useFoo = () => {
  return useState('foo', () => 'bar')
}
```

第二种是默认导出，名字来自文件名。

```ts [app/composables/use-foo.ts or composables/useFoo.ts]
// It will be available as useFoo() (camelCase of file name without extension)
export default function () {
  return useState('foo', () => 'bar')
}
```

注释里那句是关键，`use-foo.ts` 或 `useFoo.ts` 两种文件名，都以 `useFoo()` 可用，也就是文件名去掉扩展名后的 camelCase 形式。也就是说连字符版本会被转成驼峰。

用法，`.js`、`.ts`、`.vue` 文件里都能直接用。

```vue [app/app.vue]
<script setup lang="ts">
const foo = useFoo()
</script>

<template>
  <div>
    {{ foo }}
  </div>
</template>
```

`utils/` 那边官方给的两个例子，形态与上面完全对应。

```ts [utils/index.ts]
export const { format: formatNumber } = Intl.NumberFormat('en-GB', {
  notation: 'compact',
  maximumFractionDigits: 1,
})
```

```ts [utils/random-entry.ts or utils/randomEntry.ts]
// It will be available as randomEntry() (camelCase of file name without extension)
export default function (arr: Array<any>) {
  return arr[Math.floor(Math.random() * arr.length)]
}
```

```vue [app/app.vue]
<template>
  <p>{{ formatNumber(1234) }}</p>
</template>
```

第一个例子还顺手说明了具名导出可以是从对象上解构重命名出来的，`format: formatNumber`，自动导入拿到的名字是 `formatNumber`。

一条最重要的澄清，这里没有额外的响应式。官方用 note 写了整段，我全文抄下来，因为它推翻了一个常见假设。`app/composables/` 目录在 Nuxt 中不会为你的代码提供任何额外的响应式能力。相反，composables 内部的任何响应式都是通过 Vue 的 Composition API 机制实现的，例如 `ref` 与 `reactive`。请注意，响应式代码也不受 `app/composables/` 目录的边界限制。你可以自由地在你应用里需要它的任何地方使用响应式特性。

三条结论。

放进这个目录不会让一个函数变成响应式的。009 那篇里 composables 目录的价值是自动导入，不是响应式。

响应式来自 `ref` 与 `reactive`，跟目录无关。

反过来也成立，需要响应式时不必把代码塞进这个目录。

类型从哪来，`.nuxt/imports.d.ts`，以及那个报错。官方原话，在底层，Nuxt 会自动生成文件 `.nuxt/imports.d.ts` 来声明这些类型。请注意，你必须运行 `nuxt prepare`、`nuxt dev` 或 `nuxt build`，才能让 Nuxt 生成这些类型。

官方跟着一条 note，如果你在没有运行开发服务器的情况下创建一个 composable，TypeScript 会抛出一个错误，例如 `Cannot find name 'useBar'.`

这就是 031、035 那条链在这两个目录上的具体表现。类型住在 `.nuxt/imports.d.ts`，`.nuxt` 是生成目录，052 讲过。生成时机是 `prepare`、`dev`、`build` 三条，035 讲过。症状被官方点名了，刚写完 `useBar()` 就报 Cannot find name，先跑一次 `nuxt prepare`，别怀疑你代码写错了。

扫描范围，只有顶层。官方这一节标题就是 How Files Are Scanned，规则与那棵树一起给。Nuxt 只扫描 `app/composables/` 目录顶层的文件。

```bash [Directory Structure]
-| composables/
---| index.ts     // scanned
---| useFoo.ts    // scanned
---| nested/
-----| utils.ts   // not scanned
```

官方说，只有 `app/composables/index.ts` 与 `app/composables/useFoo.ts` 会被搜索以用于导入。

注意这里与 060、061 的措辞差异，中间件与插件那两节说的是顶层文件或子目录里的 index 文件，而这一节的树里 `nested/utils.ts` 明确标 not scanned，并且 `index.ts` 被扫描的位置是顶层。别把三处的规则类推成完全一样。

要嵌套也能自动导入，官方给了两条路，并标了推荐顺序，要让嵌套模块的自动导入工作，你可以重新导出它们，推荐，或者配置扫描器以包含嵌套目录。

路线一，从顶层 `index.ts` 再导出一层。

```ts [app/composables/index.ts]
// Enables auto import for this export
export { utils } from './nested/utils.ts'
```

路线二，用 `imports.dirs` 扩扫描范围。

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

这三行注释正好是三种粒度，从窄到宽，只扫顶层、只认往下层的 index 文件且限定四种扩展名、全递归。

`**` 那一档我建议你谨慎，它把整个子树都变成自动导入来源，名字冲突面会随之变大，156、159 讲冲突与扫描顺序。

两个官方示例，套娃与取插件注入。

组合式函数里用另一个组合式函数，靠的就是自动导入本身。

```ts [app/composables/test.ts]
export const useFoo = () => {
  const nuxtApp = useNuxtApp()
  const bar = useBar()
}
```

在组合式函数里访问插件注入的东西。

```ts [app/composables/test.ts]
export const useHello = () => {
  const nuxtApp = useNuxtApp()
  return nuxtApp.$hello
}
```

这第二条正好把 061 那条建议落地了，官方在插件那一节让你别到处 `provide`，而如果确实有一个注入项要读，这一节给的做法是把它包进一个 composable，`useHello()` 内部读 `nuxtApp.$hello`，调用侧见到的仍然是一个明确的函数。

边界那条 important，utils 只在 Vue 那一侧。官方用 important 给了一句限制，它的分量比看起来重，这些 utils 只在你的应用的 Vue 那一部分可用。只有 `server/utils` 会在 `server/` 目录里被自动导入。

对照着读这三行就够了。`app/composables/` 与 `app/utils/` 里的东西，应用的 Vue 侧能自动导入，组件、组合式函数、插件、页面。`server/utils/` 里的东西，服务端能自动导入，也就是 `server/` 目录。`shared/` 里的东西，两侧都能，066 专讲。

我在 `server/api` 里调用刚写的 `app/utils` 函数，报找不到，按官方这句，这就是正确行为，不是配置坏了。需要两侧共用的东西，位置在 `shared/`。

类型也分三处放。utils 页最后那条 tip 我单独拎出来，因为它回答了一个非常具体的问题，我的类型该放哪。类型可以用同样的方式自动导入。把应用专用类型放在 `app/types/`，服务端专用类型放在 `server/types/`，两侧共享的类型放在 `shared/types/`。

这三个位置与上面那张谁能导入表是一一对应的。

## 自查

- [ ] `use-foo.ts` 用默认导出时，你在页面里写的函数名是什么？规则怎么说？
- [ ] 把函数放进 `app/composables/` 会让它变响应式吗？官方怎么说？
- [ ] 刚写完一个 composable 就报 `Cannot find name 'useBar'`，缺的是哪一步？
- [ ] `composables/nested/utils.ts` 会被扫描吗？官方推荐的两条解法是什么？
- [ ] `server/api` 里想用一个 `app/utils` 的函数，能自动导入吗？正确位置在哪？
- [ ] `app/types/`、`server/types/`、`shared/types/` 分别给谁用？

## 下一篇

064 讲两个「放静态东西」的位置：`app/assets/` 与 `public/`。官方给的判据是能读出来的——`public` 里的文件「在根路径提供且不被构建过程修改」，而 `assets` 是「构建工具会处理的」，这直接决定了同一个 `src` 写法在两边的结果完全不同。
