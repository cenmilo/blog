---
url: /posts/9d6e2485935ffb08ae8397ec4b52ecb5/
title: 157 - 组件与页面的名字怎么生成：路径拼接、分组目录与两个 normalize 开关
date: 2026-12-03T10:00:00+08:00
updated: 2026-12-03T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：标签名不是文件名，官方给的生成规则是："**then the component's name will be based on its own path directory and filename, with duplicate segments being removed.**"——`components/base/foo/Button.vue` 的结果是 `<BaseFooButton />`。官方还有一句建议："**For clarity, we recommend that the component's filename matches its name. So, in the example above, you could rename `Button.vue` to be `BaseFooButton.vue`.**" 目录段可以不带进名字："**If you want to group components in a directory without affecting their name, you can use the parenthesis `(` `)` to name the grouping directory.**"，于是 `base/(foo)/Button.vue` 得到 `<BaseButton />`（"the component's name will **skip the grouping directory**"）。但"标签名"与"Vue 认识的组件名"历史上是两回事，官方在升级指南里把这条列成 v4 的中等影响变更："**Vue will now generate component names that match the Nuxt pattern for component naming.**" 并给了对照：目录 `components/SomeFolder/MyComponent.vue` "In this case, **the component name would be `MyComponent`, as far as Vue is concerned**. If you wanted to use `<KeepAlive>` with it, or identify it in the Vue DevTools, you would need to use this name. **But in order to auto-import it, you would need to use `SomeFolderMyComponent`.**" 迁移步骤点名两类代码："**Ensure that you use the updated name in any tests which use `findComponent` from `@vue/test-utils` and in any `<KeepAlive>` which depends on the name of your component.**"，回退口子是 `experimental.normalizeComponentNames: false`（"**Alternatively, for now, you can disable this behaviour**"）。页面侧另有一个开关 `experimental.normalizePageNames`："**Ensure that page component names match their route names. This sets the `__name` property on page components so that Vue's `<KeepAlive>` can correctly identify them by name.**"，它要解决的问题是"`pages/foo/index.vue` and `pages/bar/index.vue` would both have the component name `index`. This makes name-based `<KeepAlive>` filtering unreliable"，而它的默认状态与组件侧不同："**This flag is enabled when `future.compatibilityVersion` is set to `5` or higher, but you can disable this feature**"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组件命名
  - KeepAlive
  - 实验特性
  - DevTools

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我在测试里写 `findComponent({ name: 'MyComponent' })` 一直找不到组件，DevTools 里那个组件的名字也和我写在模板里的不一样。原因不是一个 bug：**"标签名"、"Vue 认识的组件名"、"路由名"是三件事**，官方给这两个名字各配了一个开关。

## 标签名怎么拼出来

官方给的输入与输出：

```bash [Directory Structure]
-| components/
---| base/
-----| foo/
-------| Button.vue
```

> ... then the component's name will be based on its own path directory and filename, **with duplicate segments being removed**. Therefore, the component's name will be:

```html
<BaseFooButton />
```

规则两半：

| 半句 | 效果 |
| --- | --- |
| based on its own **path directory and filename** | 目录段逐个拼进名字 |
| with **duplicate segments being removed** | 重复的段只算一次 |

`base/foo/Button` → `BaseFooButton` 是"全拼"的例子；重复段的处理在这类路径里才会显形（比如目录本身叫 `base/base-button/Button.vue`，重复的 `base` 只留一次——这一格是按官方那句话推的结果，具体去重边界我没在文档里找到逐条定义，值得实测时用真实目录验一次）。

官方的建议是把文件名对齐：

> For clarity, we recommend that the component's filename matches its name. So, in the example above, you could rename `Button.vue` to be `BaseFooButton.vue`.

**这条建议的实际收益在别处**：文件名 = 组件名后，`import` 出来的符号、DevTools 里显示的名字、模板里的标签三者一致，排查时不用在脑子里做一次路径拼接。

## 分组目录：只想分文件夹，不想改名字

> If you want to group components in a directory without affecting their name, you can use the parenthesis `(` `)` to name the grouping directory.

```bash [Grouping Directory]
-| components/
---| base/
-----| (foo)/
-------| Button.vue
```

> ... then the component's name will **skip the grouping directory**, resulting in:

```html
<BaseButton />
```

`base/(foo)/Button.vue` → `<BaseButton />`：`base` 进了名字，`(foo)` 没进。路由那一侧也有同名机制（`pages/(marketing)/about.vue` → `/about`，官方还给了 `route.meta.groups`，见 116），**两处用的是同一套括号约定，但产出不同**：路由侧把它写进 `groups` 元数据，组件侧只是把它从名字里剔除。

## 组件侧的开关：normalizeComponentNames

实验特性页的定义句很短，但把用途钉死了：

> Nuxt updates auto-generated Vue component names to match the full component name you would use to auto-import the component.
>
> If you encounter issues, you can disable this feature.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    normalizeComponentNames: false,
  },
})
```

官方给的对照例子（这一例把"两个名字"讲得最清楚）：

```bash [Directory structure]
├─ components/
├─── SomeFolder/
├───── MyComponent.vue
```

> By default, if you haven't set it manually, Vue will assign a component name that matches the filename of the component.
>
> In this case, **the component name would be `MyComponent`, as far as Vue is concerned**. If you wanted to use `<KeepAlive>` with it, or identify it in the Vue DevTools, you would need to use this name.
>
> **But in order to auto-import it, you would need to use `SomeFolderMyComponent`.**
>
> By setting `experimental.normalizeComponentNames`, these two values match, and Vue will generate a component name that matches the Nuxt pattern for component naming.

**这就是我那次测试找不到的原因**：模板里能写 `<SomeFolderMyComponent />`，而 Vue 内部记的名字在规范化之前是 `MyComponent`（或者反过来，规范化之后变成 `SomeFolderMyComponent`）。

升级指南把这件事作为 v4 的变更列出，影响级别官方标的是 Moderate：

> ### Normalized Component Names
>
> 🚦 **Impact Level**: Moderate
>
> Vue will now generate component names that match the Nuxt pattern for component naming.

迁移步骤点名了两类会受影响的代码，我认为这才是这一节真正要带走的清单：

> Ensure that you use the **updated name** in any tests which use `findComponent` from `@vue/test-utils` and in any `<KeepAlive>` which depends on the name of your component.
>
> Alternatively, **for now**, you can disable this behaviour with: `experimental: { normalizeComponentNames: false }`

**那个 "for now" 有分量**：关档是被官方描述为临时手段的选项，不是长期方案。所以我的做法是**改测试与 `include`/`exclude` 里的名字**，而不是关掉它。

## 页面侧的开关：normalizePageNames

页面组件有自己的一套名字问题，官方定义句里连实现细节都给了：

> Ensure that page component names match their route names. **This sets the `__name` property on page components** so that Vue's `<KeepAlive>` can correctly identify them by name.
>
> By default, Vue assigns component names based on the filename. For example, `pages/foo/index.vue` and `pages/bar/index.vue` would both have the component name `index`. **This makes name-based `<KeepAlive>` filtering unreliable because multiple pages share the same name.**
>
> With `normalizePageNames` enabled, page components are named after their route (e.g. `foo` and `bar`), so you can use `<KeepAlive>` with `include`/`exclude` **without manually adding `defineOptions({ name: '...' })` to each page.**

**`index.vue` 那个例子是这套机制存在的唯一理由**：只要你有两个以上的子目录首页，它们的组件名就全叫 `index`——按名字过滤保活必然误伤。官方给的解法是"按路由命名"，也就是 `foo` 与 `bar`：

```vue [app.vue]
<template>
  <NuxtPage :keepalive="{ include: ['foo'] }" />
</template>
```

关键差别在默认状态。这一项**不默认开**：

> This flag is enabled when **`future.compatibilityVersion` is set to `5` or higher**, but you can disable this feature:

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    normalizePageNames: false,
  },
})
```

（`future.compatibilityVersion` 这个键我在 138 见过它的效果注记——"With `future.compatibilityVersion: 5`, routing is case-sensitive by default to match Nitro"。它的定位句是 `features` 那一页给的："There is also a `future` namespace for early opting-in to new features that will become default in a future (possibly major) version of the framework."）

## 两个开关放一起看

| | `normalizeComponentNames` | `normalizePageNames` |
| --- | --- | --- |
| 管谁 | `components/` 里的组件 | 页面组件 |
| 对齐目标 | **自动导入用的名字**（路径拼接后的那个） | **路由名** |
| 改的是什么 | Vue 生成的组件名 | 页面组件的 `__name` 属性 |
| 默认状态 | v4 起生效（官方标 Moderate 影响，可关，措辞是 "for now"） | 需要 `future.compatibilityVersion: 5` 才默认开 |
| 关掉的写法 | `experimental: { normalizeComponentNames: false }` | `experimental: { normalizePageNames: false }` |
| 谁会被打挂 | `findComponent` 测试、依赖名字的 `<KeepAlive>` | 按名字的 `include`/`exclude`（不开时本来就不可靠） |

两张表的共同指向是同一件小事：**只要代码里出现"按组件名"的字符串，就要先确认此刻名字是哪个体系的**。137 讲 `key`/`pageKey` 决定"要不要重建"，这一篇讲的是"重建之外，能不能被点名"。

## 自查

1. `components/base/foo/Button.vue` 的标签名是什么？官方规则的两半分别怎么说？
2. "duplicate segments being removed" 在哪类路径上才会显形？文档有没有逐条定义去重边界？
3. 官方为什么建议文件名与名字一致？它原话里的两个理由词是什么（"For clarity…"）？
4. `(foo)` 目录在组件侧与路由侧分别产出什么？
5. `components/SomeFolder/MyComponent.vue` 这个例子里，"Vue 认为的名字"和"自动导入用的名字"分别是什么？`normalizeComponentNames` 做的对齐是哪一边向哪一边靠？
6. 这个变更的迁移步骤点名了哪两类代码？关档那句里的时间限定词是哪个？
7. `normalizePageNames` 具体设置的是页面组件的哪个属性？它解决的 `index.vue` 问题是什么形状？
8. 两个开关的默认状态差别在哪一句里？`normalizePageNames` 需要什么前提才默认开？
9. `<NuxtPage :keepalive="{ include: ['foo'] }" />` 为什么在开了 `normalizePageNames` 之后才靠得住？

## 下一篇

158 把三种"人为干预命名"的手段摊平对照：`pathPrefix: false`、`prefix`、以及分组目录各自改变名字的哪一段，以及同一目录被列两次时会发生什么。

## 本文依据

- `https://nuxt.com/docs/4.x/directory-structure/app/components`
  - 原文（Component Names）："If you have a component in nested directories such as: `-| components/ ---| base/ -----| foo/ -------| Button.vue`"；"... then the component's name will be based on its own path directory and filename, with duplicate segments being removed. Therefore, the component's name will be: `<BaseFooButton />`"；note "For clarity, we recommend that the component's filename matches its name. So, in the example above, you could rename `Button.vue` to be `BaseFooButton.vue`."
  - 原文（分组目录）："If you want to group components in a directory without affecting their name, you can use the parenthesis `(` `)` to name the grouping directory."；目录示例 `-| components/ ---| base/ -----| (foo)/ -------| Button.vue`；"... then the component's name will skip the grouping directory, resulting in: `<BaseButton />`"
- `https://nuxt.com/docs/4.x/guide/going-further/experimental-features`
  - 原文（normalizeComponentNames）："Nuxt updates auto-generated Vue component names to match the full component name you would use to auto-import the component."；"If you encounter issues, you can disable this feature." + `experimental: { normalizeComponentNames: false }`；"By default, if you haven't set it manually, Vue will assign a component name that matches the filename of the component."；目录示例 `├─ components/ ├─── SomeFolder/ ├───── MyComponent.vue`；"In this case, the component name would be `MyComponent`, as far as Vue is concerned. If you wanted to use `<KeepAlive>` with it, or identify it in the Vue DevTools, you would need to use this name."；"But in order to auto-import it, you would need to use `SomeFolderMyComponent`."；"By setting `experimental.normalizeComponentNames`, these two values match, and Vue will generate a component name that matches the Nuxt pattern for component naming."
  - 原文（normalizePageNames）："Ensure that page component names match their route names. This sets the `__name` property on page components so that Vue's `<KeepAlive>` can correctly identify them by name."；"By default, Vue assigns component names based on the filename. For example, `pages/foo/index.vue` and `pages/bar/index.vue` would both have the component name `index`. This makes name-based `<KeepAlive>` filtering unreliable because multiple pages share the same name."；"With `normalizePageNames` enabled, page components are named after their route (e.g. `foo` and `bar`), so you can use `<KeepAlive>` with `include`/`exclude` without manually adding `defineOptions({ name: '...' })` to each page."；"This flag is enabled when `future.compatibilityVersion` is set to `5` or higher, but you can disable this feature:" + `experimental: { normalizePageNames: false }`；示例 `<NuxtPage :keepalive="{ include: ['foo'] } />`（`app.vue`）
- `https://nuxt.com/docs/4.x/getting-started/upgrade`
  - 原文（Normalized Component Names 条目）：标题 "Normalized Component Names"、"🚦 Impact Level: Moderate"、"Vue will now generate component names that match the Nuxt pattern for component naming."；"What Changed" 段落与 `components/SomeFolder/MyComponent.vue` 例子同上；"Migration Steps" — "Ensure that you use the updated name in any tests which use `findComponent` from `@vue/test-utils` and in any `<KeepAlive>` which depends on the name of your component."；"Alternatively, for now, you can disable this behaviour with:" + `experimental: { normalizeComponentNames: false }`
- `https://nuxt.com/docs/4.x/api/components/nuxt-page` 与 `https://nuxt.com/docs/4.x/api/utils/define-page-meta`
  - 对照用（见 092、137、119）：`keepalive` 的 `include`/`exclude` 与 `key`/`pageKey` 的分工
- `https://nuxt.com/docs/4.x/directory-structure/app/pages`
  - 对照用（见 116）：路由组的括号目录与 `route.meta.groups`
