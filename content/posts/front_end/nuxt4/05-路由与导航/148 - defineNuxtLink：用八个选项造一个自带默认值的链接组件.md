---
url: /posts/ee956f303471aa0bbfa2ccb898213343/
title: 148 - defineNuxtLink：用八个选项造一个自带默认值的链接组件
date: 2026-11-30T10:00:00+08:00
updated: 2026-11-30T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：147 讲了 `<NuxtLink>` 的默认值可以在 `experimental.defaults.nuxtLink` 里改，官方在同一页又给了第二个口子："**You can overwrite `<NuxtLink>` defaults by creating your own link component using `defineNuxtLink`.**"。官方示例连文件位置一起给出，标注是 `app/components/MyNuxtLink.ts`，内容只有一行导出：`export default defineNuxtLink({ componentName: 'MyNuxtLink', /* see signature below for more */ })`。紧接着那句是这一篇的核心，也是最多人记反的地方："**The component is auto-imported by its file name, so you can use `<MyNuxtLink />` as usual with your new defaults. `componentName` only sets the component's internal name (as shown in Vue DevTools); it does not change how the component is used in templates.**"——模板里写什么名字**由文件名决定**，`componentName` 只决定 DevTools 里显示什么。签名官方原样列出：`interface NuxtLinkOptions { componentName?, externalRelAttribute?, activeClass?, exactActiveClass?, trailingSlash?, prefetch?, prefetchedClass?, prefetchOn? (Partial<{ visibility: boolean, interaction: boolean }>) }` 与 `function defineNuxtLink (options: NuxtLinkOptions): Component {}`——**八个键全部可选，返回一个 `Component`**。逐条定义里，三条带了硬行为：`externalRelAttribute` "Defaults to `"noopener noreferrer"`. **Set it to `""` to disable**"；`trailingSlash` "An option to either add or remove trailing slashes in the `href`. **If unset or not matching the valid values `append` or `remove`, it will be ignored.**"；`activeClass`/`exactActiveClass` 被定义为"**A default class to apply** on active links. Works the same as Vue Router's `linkActiveClass` option"，也就是它们只改这个组件的默认值，不是全局。而 `componentName` 那条最后一句补了默认值："**Default is `NuxtLink`.**"

categories:
  - 前端开发

tags:
  - nuxt4
  - defineNuxtLink
  - 组件
  - 默认值
  - NuxtLink

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一节那张默认值表我看完的第一个反应是：如果我只想让**站内面包屑那一类链接**统一用 `interaction` 预取，而其他地方保持默认，`experimental.defaults.nuxtLink` 是全站的，改不了局部。官方给的答案是这一句——自己造一个链接组件。

## 官方给的位置与写法

> You can overwrite `<NuxtLink>` defaults by creating your own link component using `defineNuxtLink`.

```ts [app/components/MyNuxtLink.ts]
export default defineNuxtLink({
  componentName: 'MyNuxtLink',
  /* see signature below for more */
})
```

这两个细节都在示例本身里：

| 细节 | 依据 |
| --- | --- |
| 文件放在 `app/components/` 下、扩展名 `.ts` | 官方代码块的文件标注就是 `app/components/MyNuxtLink.ts` |
| 只写 `default export`，不用 `defineComponent` | 官方示例的整个文件体就是 `export default defineNuxtLink({...})` |
| 不用写 import | 它出现在 `api/components/nuxt-link` 这一节的示例里、没有任何 import 语句；`defineNuxtLink` 属于自动导入范畴（自动导入的完整边界我从 150 起专门讲） |

## 两个名字：模板名来自文件名，DevTools 名来自 componentName

这一句是整节最容易记反的，官方原话：

> The component is auto-imported by its **file name**, so you can use `<MyNuxtLink />` as usual with your new defaults. **`componentName` only sets the component's internal name (as shown in Vue DevTools); it does not change how the component is used in templates.**

`componentName` 那条定义又把同一个意思说了一遍，还补了默认值：

> `componentName`: The component's internal name, as shown in Vue DevTools. **It does not change the name used in templates (that comes from the component's file name). Default is `NuxtLink`.**

所以：

| 你改的 | 影响的 |
| --- | --- |
| 文件名 `MyNuxtLink.ts` | 模板里能写 `<MyNuxtLink />`（自动导入名） |
| `componentName: 'MyNuxtLink'` | Vue DevTools 里显示的组件名 |
| 都不写 | 自动导入名仍是文件名；`componentName` 默认是 `NuxtLink` |

**这里有个可见的坑**：官方示例把文件名和 `componentName` 写成一样的值（都是 `MyNuxtLink`）。如果我改成 `componentName: 'SiteLink'` 而文件仍叫 `MyNuxtLink.ts`，模板里能用的还是 `<MyNuxtLink />`，而 DevTools 里显示的是 `SiteLink`——两边对不上，排查组件时会愣一下。**所以我写这类组件时让两处保持一致**，`componentName` 就当给 DevTools 的注释用。这个一致性要求是我的取向，官方只给了"它不改变模板用法"这一条事实。

## 签名：八个可选键，返回一个组件

```ts
interface NuxtLinkOptions {
  componentName?: string
  externalRelAttribute?: string
  activeClass?: string
  exactActiveClass?: string
  trailingSlash?: 'append' | 'remove'
  prefetch?: boolean
  prefetchedClass?: string
  prefetchOn?: Partial<{
    visibility: boolean
    interaction: boolean
  }>
}
function defineNuxtLink (options: NuxtLinkOptions): Component {}
```

三条从签名里就能读出的信息：

- **八个键全是 `?`**：只传想改的那几个就够。
- **返回值类型是 `Component`**：所以它就是一个组件工厂，产物按普通组件用（放 `app/components/` 里作为默认导出即被自动导入）。
- **`prefetchOn` 是 `Partial<{ visibility, interaction }>`**：可以只写一个键，比如 `{ interaction: true }`。

## 逐条默认值与硬规则

| 键 | 官方定义 | 默认值 |
| --- | --- | --- |
| `componentName` | "The component's internal name, as shown in Vue DevTools. It does not change the name used in templates (that comes from the component's file name)." | `NuxtLink` |
| `externalRelAttribute` | "A default `rel` attribute value applied on external links. **Set it to `""` to disable**" | `"noopener noreferrer"` |
| `activeClass` | "A default class to apply on active links. Works the same as Vue Router's `linkActiveClass` option." | `"router-link-active"` |
| `exactActiveClass` | "A default class to apply on exact active links. Works the same as Vue Router's `linkExactActiveClass` option." | `"router-link-exact-active"` |
| `trailingSlash` | "An option to either add or remove trailing slashes in the `href`. **If unset or not matching the valid values `append` or `remove`, it will be ignored.**" | 未设（被忽略） |
| `prefetch` | "Whether or not to prefetch links by default." | （与 `<NuxtLink>` 默认为开一致，见 147 那张 `defaults` 表里的 `prefetch: true`） |
| `prefetchOn` | "Granular control of which prefetch strategies to apply by default." | `<NuxtLink>` 那侧默认是 `{ visibility: true }` |
| `prefetchedClass` | "A default class to apply to links that have been prefetched." | 未设 |

三处值得单独记：

**1）`externalRelAttribute` 的关闭方式是空字符串，不是 `false`。** 官方只写了 `Set it to "" to disable`。这一项和 126 那条默认 `rel` 是同一件事的两个入口：那个是 `<NuxtLink>` 的全局默认，这里是**你这个组件**的默认。

**2）`trailingSlash` 只有两个合法值，且非法值不会报错。** 原文："If unset or not matching the valid values `append` or `remove`, **it will be ignored**." 传 `'append '` 之类的错拼是**静默失效**的，这类"配了没反应"的字段只能靠看渲染出来的 `href` 验。

**3）两个 class 是"这个组件的默认值"。** 官方给它们的话与 `<NuxtLink>` props 那节不同：这里写的是 "**A default class** to apply"，props 那节写的是 "A class to apply"。所以这一层设的值仍然是**可被单个链接的 `active-class` prop 覆盖**的默认值——这跟 147 末尾那张"三处可配"表是同一套结构。

## 能配到什么程度：官方没写的那部分

这一节只给了 options 到 `Component` 的签名，**没有列出造出来的组件支持哪些 props**，也没有说明它与 `<NuxtLink>` 的 props 是否完全一致。官方唯一的表述是那句"you can use `<MyNuxtLink />` **as usual with your new defaults**"——"照常使用，只是默认值换了"。

我不敢替文档补一层"它继承了全部三组 props"的结论。实际做法：把新组件当 `<NuxtLink>` 用（`to`、`external`、`prefetch` 这些先按 147 的表写），然后**看编辑器给的类型提示与属性检查**是否通过；这条验证成本很低，一次就够。

## 什么时候值得造一个

| 你的需求 | 是否用 `defineNuxtLink` |
| --- | --- |
| 全站所有链接统一某个 active class / 预取策略 | 不用，改 `experimental.defaults.nuxtLink`（147），或 `router.options.linkActiveClass`（142） |
| **某一类链接**要不同的默认行为（比如只 hover 才预取） | ✅ 造一个，另一类继续用 `<NuxtLink>` |
| 站内链接一律带同一套 `trailingSlash` 处理 | ✅ 造一个，把 URL 归一规则收在这层 |
| 只想给单个链接换 `rel` | 不用，直接写 `rel` 或 `no-rel` prop（126） |
| 想在 `<NuxtLink>` 外面再包一层样式类 | 官方口子只有这八个键；样式类不在其中，那属于你自己封装组件的事 |

一句话：**`defineNuxtLink` 是"作用域收窄版"的默认值覆盖**。`experimental.defaults.nuxtLink` 管全站，`defineNuxtLink` 管一类，prop 管一条。

## 自查

1. 官方给 `defineNuxtLink` 的定位句是什么？它和 `experimental.defaults.nuxtLink` 那层的差别在哪？
2. 官方示例文件放在哪个目录、叫什么扩展名？文件里除了 `export default defineNuxtLink({...})` 还有别的吗？
3. 模板里 `<MyNuxtLink />` 这个名字是从哪来的？`componentName` 改的又是什么？它的默认值是什么？
4. 八个键里有哪几个是可选的？返回值类型是什么？这说明它是什么角色的函数？
5. 想彻底关掉外部链接的默认 `rel`，在这一层怎么写？在哪一层写会关掉全站链接的？
6. `trailingSlash` 传了非法值会怎样？官方原话怎么说？
7. `activeClass` 在这一层设了之后，单个链接还能不能覆盖？依据是哪句措辞差别？
8. 造出来的新组件支持哪些 props，文档说清了吗？你会怎么低成本确认？

## 下一篇

149 给 110-149 收尾：把一次导航经过的全部环节按顺序排出来——从 `<a href>` 被点下，到中间件、`validate`、布局与过渡、`page:loading:*` 钩子，再到 chunk 预取在哪一步发生。

## 本文依据

- `https://nuxt.com/docs/4.x/api/components/nuxt-link`
  - 原文（Custom Link Component）："You can overwrite `<NuxtLink>` defaults by creating your own link component using `defineNuxtLink`."；示例（文件标注 `app/components/MyNuxtLink.ts`）：`export default defineNuxtLink({ componentName: 'MyNuxtLink', /* see signature below for more */ })`
  - 原文（两个名字）："The component is auto-imported by its file name, so you can use `<MyNuxtLink />` as usual with your new defaults. `componentName` only sets the component's internal name (as shown in Vue DevTools); it does not change how the component is used in templates."
  - 原文（Signature）：`interface NuxtLinkOptions { componentName?: string; externalRelAttribute?: string; activeClass?: string; exactActiveClass?: string; trailingSlash?: 'append' | 'remove'; prefetch?: boolean; prefetchedClass?: string; prefetchOn?: Partial<{ visibility: boolean, interaction: boolean }> }` 与 `function defineNuxtLink (options: NuxtLinkOptions): Component {}`
  - 原文（逐条定义）："`componentName`: The component's internal name, as shown in Vue DevTools. It does not change the name used in templates (that comes from the component's file name). Default is `NuxtLink`."；"`externalRelAttribute`: A default `rel` attribute value applied on external links. Defaults to `"noopener noreferrer"`. Set it to `""` to disable"；"`activeClass`: A default class to apply on active links. Works the same as Vue Router's `linkActiveClass` option. Defaults to Vue Router's default (`"router-link-active"`)"；"`exactActiveClass`: A default class to apply on exact active links. Works the same as Vue Router's `linkExactActiveClass` option. Defaults to Vue Router's default (`"router-link-exact-active"`)"；"`trailingSlash`: An option to either add or remove trailing slashes in the `href`. If unset or not matching the valid values `append` or `remove`, it will be ignored."；"`prefetch`: Whether or not to prefetch links by default."；"`prefetchOn`: Granular control of which prefetch strategies to apply by default."；"`prefetchedClass`: A default class to apply to links that have been prefetched."
  - 原文（In Nuxt Config 的默认值块，用于对照）：`experimental.defaults.nuxtLink` 八个键与注释 `// default values`、`componentName: 'NuxtLink'`、`externalRelAttribute: 'noopener noreferrer'`、`activeClass: 'router-link-active'`、`exactActiveClass: 'router-link-exact-active'`、`prefetchedClass: undefined // can be any valid string class name`、`trailingSlash: undefined // can be 'append' or 'remove'`、`prefetch: true`、`prefetchOn: { visibility: true }`；important "These options will likely be moved elsewhere in the future, such as into `app.config` or into the `app/` directory."
  - 原文（props 那节措辞对照）：`activeClass` — "A class to apply on active links."；`prefetchOn` — "Possible options are `interaction` and `visibility` (default)."
  - 原文（页尾）：`::link-example{to="https://nuxt.com/docs/examples/routing/pages"}`；Source 位于 `packages/nuxt/src/app/components/nuxt-link.ts`
- `https://nuxt.com/docs/4.x/guide/recipes/custom-routing`
  - 对照用（见 142）：`router.options` 里"只有 JSON 可序列化"的七个选项含 `linkActiveClass`、`linkExactActiveClass`
- `https://nuxt.com/docs/4.x/api/advanced/hooks`
  - 对照用（见 145）："`link:prefetch` | `to` | Client | Called when a `<NuxtLink>` is observed to be prefetched."
