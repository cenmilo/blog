---
url: /posts/72dcf1804d3bff0adb3dedf0e2bcf452/
title: 687 - 模块的版本兼容检查：七个 Kit 函数与那句 may be removed in v5
date: 2027-05-29T09:00:00+08:00
updated: 2027-05-29T09:00:00+08:00
author: cmdragon

summary:
  摘要：Kit 的兼容页开场就把范围写死了：这些工具能用在 Nuxt 3、用 Bridge 的 Nuxt 2，甚至不用 Bridge 的 Nuxt 2。页面给了七个函数，三种检查（返回问题数组、抛错、返回布尔值）加四个更细的判断。真正要用记住两点：`bridge` 那个约束的对象是**构建器**（`vite`、`webpack`、`rspack`），设成 `false` 表示禁用；`isNuxt2` 和 `isNuxt3` 都挂着同一句"This may be removed in @nuxt/kit v5 or a future major version"。

categories:
  - 前端开发

tags:
  - nuxt4
  - 模块开发
  - 版本兼容
  - nuxt-kit
  - 发布节奏
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 三种检查方式，差别只在失败时怎么告诉你

这一节开头的分类是这样的：`checkNuxtCompatibility`、`assertNuxtCompatibility`、`hasNuxtCompatibility` 用来确认当前 Nuxt 版本是否满足你给的约束；`isNuxt2`、`isNuxt3`、`getNuxtVersion` 用来做更细的判断。页面后面还多了一个 `isNuxtMajorVersion`，一共七个。

| 函数 | 返回 | 失败时 | 文档原话要点 |
| --- | --- | --- | --- |
| `checkNuxtCompatibility` | `Promise<NuxtCompatibilityIssues>` | 返回一条消息的数组 | Nuxt 2 的版本**同时检查 `bridge` 支持** |
| `assertNuxtCompatibility` | `Promise<true>` | **抛出错误**，错误里是问题列表转成的字符串 | 断言约束必须满足 |
| `hasNuxtCompatibility` | `Promise<boolean>` | 返回 `false` | Nuxt 2 的版本同时检查 `bridge` 支持 |
| `isNuxtMajorVersion` | `boolean` | 返回 `false` | 参数是 `major: number` |
| `isNuxt3` | `boolean` | 返回 `false` | 检查当前版本是否为 3.x，带 removal 提示 |
| `isNuxt2` | `boolean` | 返回 `false` | 检查当前版本是否为 2.x，带 removal 提示 |
| `getNuxtVersion` | `string` | —— | 返回当前 Nuxt 版本 |

七个函数最后一个参数都是同一个：`nuxt?: Nuxt`，说明也一致——不提供时通过 `useNuxt()` 从上下文取。

## 约束对象只有两个键

`checkNuxtCompatibility` 的参数表里，`constraints` 的完整形状只列了两行，其余几个函数都指回这张表：

| 属性 | 类型 | 必填 | 描述 |
| --- | --- | --- | --- |
| `nuxt` | `string` | `false` | semver 格式的 Nuxt 版本。版本可以按 Node.js 的写法来定义，例如：`>=2.15.0 <3.0.0` |
| `bridge` | `Record<string, string \| false>` | `false` | 指定版本约束，**或为特定的 Nuxt 构建器（例如 `vite`、`webpack`、`rspack`）关闭兼容性**。用 `false` 表示禁用 |

第二行是这个页面最容易被读错的地方。名字叫 `bridge`，但它管的不是"Nuxt 2 到 Nuxt 3 的那座桥"这一件事——文档写的是"为特定构建器禁用兼容性"，值可以是版本约束字符串，也可以是 `false`。所以它同时承担两种语义。

官方示例给的是 `nuxt` 那一半：

```ts
import { checkNuxtCompatibility, defineNuxtModule } from '@nuxt/kit'

export default defineNuxtModule({
  async setup (_options, nuxt) {
    const issues = await checkNuxtCompatibility({ nuxt: '^2.16.0' }, nuxt)
    if (issues.length) {
      console.warn('Nuxt compatibility issues found:\n' + issues.toString())
    } else {
      // do something
    }
  },
})
```

`hasNuxtCompatibility` 的示例则是把结果当开关用：

```ts
const usingNewPostcss = await hasNuxtCompatibility({ nuxt: '^2.16.0' }, nuxt)
if (usingNewPostcss) {
  // do something
} else {
  // do something else
}
```

（这两段示例里的版本号都还是 `^2.16.0`，是文档原样。**页面没有给出面向 4.x 的示例约束写法**，真要限制"至少 4.x"得自己按 semver 拼。）

## 那句 removal 提示挂在两个函数上

`isNuxt3` 和 `isNuxt2` 两节都挂着同一条 note：

> Use `isNuxtMajorVersion(2, nuxt)` instead. This may be removed in @nuxt/kit v5 or a future major version.

对照着读会发现一处对不上：`isNuxt3` 那一节讲的是"当前版本是不是 3.x"，但它给的替代写法是 `isNuxtMajorVersion(2, nuxt)`，也就是查 2。这条提示在 `isNuxt2` 那节读得通，在 `isNuxt3` 这节读不通；**文档没有说明这是笔误还是别的意图**。安全的写法是别用这两个，直接按你要判断的主版本调 `isNuxtMajorVersion(major, nuxt)`：

```ts
import { defineNuxtModule, isNuxtMajorVersion } from '@nuxt/kit'

export default defineNuxtModule({
  setup () {
    if (isNuxtMajorVersion(3)) {
      // do something for Nuxt 3
    } else {
      // do something else for other versions
    }
  },
})
```

文档的示例判断的是 3，注释写"否则给其他版本做别的事"。这一节没有任何一处示例判断 4。

## 版本线本身：模块作者的依赖上限在哪

兼容检查只能判断"现在跑的是什么"。要决定"我声明支持到哪"，得看发布节奏那一节的几条承诺：

- 从 2023 年 1 月起采用固定发布节奏，遵循 semver；目标是**每年一个大版本**，patch 大约每周、minor 大约每月。
- 关键的那句：这些版本"**不应该包含破坏性变更，除非是在明确标记为 `experimental` 的选项范围内**"。
- 4 与 5 有一次计划中的调整：Nuxt 4 是一个以稳定为主的版本，包含所有 `compatibilityVersion: 4` 的特性；之后很快由 Nuxt 5 接上，包含升级到 Nitro v3 和更多变更。理由是"把破坏性变更拆成可管理的阶段"。
- 支持期承诺：**每个主版本至少支持到下一个主版本发布后六个月**。
- 当前活跃版本是 v4，npm 上以 `latest` 标签发布；Nuxt 3 已于 2026 年 7 月 31 日结束生命周期，不再收错误修复与安全补丁。

那张生命周期表一共五行，逐行落地：

| 版本 | 状态 | 首次发布 | 结束支持 | 文档 |
| --- | --- | --- | --- | --- |
| **5.x** | scheduled | Q4 2026（估计） | TBA | —— |
| **4.x** | stable | 2025-07-16 | 5.x 发布后 6 个月 | nuxt.com 的 4.x 入门页 |
| **3.x** | unsupported | 2022-11-16 | 2026-07-31 | nuxt.com 的 3.x 入门页 |
| **2.x** | unsupported | 2018-09-21 | 2024-06-30 | v2.nuxt.com |
| **1.x** | unsupported | 2018-01-08 | 2019-09-21 | —— |

状态词的定义也给了表，五个：Unsupported = 不再维护、不会收到安全补丁；Maintenance = 只收安全补丁；Stable = 正在为其开发、会收到安全补丁；Development = 可能不稳定；Scheduled = 还不存在但已在计划中。

把两张表叠起来看，模块作者的实际约束是这样的：主版本一年一个、上一版本只保半年，意味着一个假定"Nuxt 3 用户"的模块，在 2026 年 7 月之后已经没有官方补丁通道了；而 `experimental` 里的东西按承诺是可以随时改的，所以依赖实验开关的模块要预期被打断。

## 落地检查三件事

- 版本要求写在 `meta.compatibility` 里，别只在 README 写一句"支持 Nuxt 3"。最佳实践那一节对措辞还有明确建议：用"X for Nuxt"而不是"X for Nuxt 3"，避免生态分裂。
- 需要按版本走不同分支时用 `isNuxtMajorVersion`，不要用挂着 removal 提示的 `isNuxt2` / `isNuxt3`。
- 需要按构建器分情况时用 `bridge` 那个对象，并且清楚它的值既可以是版本约束也可以是 `false`；文档没有说明"某个构建器被禁用之后模块被拒绝"时用户看到的具体报错，要确认得实际跑一次。

下一篇把散在几页里的发布要求合一份检查清单：十条最佳实践、四类测试、`npm run release` 里跑的六步。

## 下一篇

下一篇 688：模块发布检查清单：九条实践各自的验证动作与三类测试

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/compatibility
- https://nuxt.com/docs/4.x/community/roadmap
- https://nuxt.com/docs/4.x/guide/modules/best-practices
