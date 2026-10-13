---
url: /posts/d2745a9467b6a25dbb3a91c74832441f/
title: 646 - Kit 兼容性七函数：三个 check 系列与那两个该换掉的 isNuxt2、isNuxt3
date: 2027-05-15T10:00:00+08:00
updated: 2027-05-15T10:00:00+08:00
author: cmdragon

summary:
  摘要：判断"当前这套 Nuxt 能不能跑我这个模块"，这一节给了**七个函数**，分三档：`checkNuxtCompatibility` **返回问题列表**（不合适时数组里是消息）、`assertNuxtCompatibility` **直接抛错**、`hasNuxtCompatibility` **返回布尔**。约束对象只有两项：`nuxt`（semver，也可以写成 Node.js 那种 `>=2.15.0 <3.0.0`）与 `bridge`（**一个 `Record<string, string \| false>`，管的是 vite、webpack、rspack 这些构建器的版本约束或直接用 `false` 关掉**）。另外四个是细粒度判断：`isNuxtMajorVersion`、`isNuxt3`、`isNuxt2`、`getNuxtVersion`。**后两个官方让你换成 `isNuxtMajorVersion`，并说可能在 v5 或将来某个大版本移除**；而 `isNuxt3` 那一节给的重定向建议**写的却是 `isNuxtMajorVersion(2, nuxt)`**，与它自己检查的 3.x 不符。

categories:
  - 前端开发

tags:
  - nuxt4
  - 兼容性判断
  - NuxtKit
  - 版本约束
  - Bridge
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

一份模块要同时伺候好几代 Nuxt。判断版本这件事如果靠字符串比较，很容易在 pre-release 或者 Bridge 那种半新半旧的形态上判错——这一节的七个函数就是为它准备的。

## 先确认这套工具覆盖哪些代

开场第一句划了范围：**Kit 的工具可以用在 Nuxt 3、用 Bridge 的 Nuxt 2，甚至可以用在没有 Bridge 的 Nuxt 2**。

然后给出函数分组：为保证模块在所有版本上可用，**可以用 `checkNuxtCompatibility`、`assertNuxtCompatibility`、`hasNuxtCompatibility`，它们会检查当前 Nuxt 版本是否满足你提供的约束**；此外**可以用 `isNuxt2`、`isNuxt3`、`getNuxtVersion` 做更细粒度的判断**。

数一下这句：三个 check ＋ 三个细粒度 ＝ 六个。**但页面上的 `##` 条目有七个**，多出来的是 `isNuxtMajorVersion`——它没出现在开场句里，却是这一节里唯一"官方推荐拿来替代前两个"的那个。

## 三个 check 的差别只在"不合适时发生什么"

| 函数 | 返回类型 | 约束不满足时 |
| --- | --- | --- |
| `checkNuxtCompatibility` | `Promise<NuxtCompatibilityIssues>` | **返回一个消息数组**，你自己决定怎么处理 |
| `assertNuxtCompatibility` | `Promise<true>` | **抛出一个错误**，错误内容是以字符串形式给出的问题清单 |
| `hasNuxtCompatibility` | `Promise<boolean>` | 返回 `false` |

三条描述里重复出现的一句是：**Nuxt 2 的版本也会检查 `bridge` 支持**（`checkNuxtCompatibility` 与 `hasNuxtCompatibility` 两节的原话都带这句）。

`checkNuxtCompatibility` 的官方用法就是"拿数组长度当开关"：

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

`hasNuxtCompatibility` 的用法是同一个约束换成布尔判断：

```ts
const usingNewPostcss = await hasNuxtCompatibility({ nuxt: '^2.16.0' }, nuxt)
if (usingNewPostcss) {
  // do something
} else {
  // do something else
}
```

**要分支执行选 `has`，要把原因说给用户听选 `check`，要让构建停下来选 `assert`**。这三个动词的差别就是这一节的全部设计。

## 约束对象只有两项

```ts
function checkNuxtCompatibility (constraints: NuxtCompatibility, nuxt?: Nuxt): Promise<NuxtCompatibilityIssues>
```

| 属性 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `nuxt` | `string` | `false` | Nuxt 版本，**semver 格式**；版本也可以按 Node.js 的方式写，例如 `>=2.15.0 <3.0.0` |
| `bridge` | `Record<string, string \| false>` | `false` | 指定版本约束，或**为特定 Nuxt 构建器关闭兼容性**，可点名 `vite`、`webpack`、`rspack`；**用 `false` 表示禁用** |

第二项容易被读错：**`bridge` 这个名字让人以为它只管 Bridge，但描述里说的是"针对具体的 Nuxt builder"**。也就是说这一项是**按构建器给约束或关掉兼容**，值形态是"构建器名 → semver 或 `false`"。

`>=2.15.0 <3.0.0` 这种**空格分隔的复合范围**是文档明确允许写的，不必自己拼两个条件。

## 细粒度的那四个

**`isNuxtMajorVersion`** 定性为"检查当前 Nuxt 实例是否属于指定的大版本"：

```ts
function isNuxtMajorVersion (major: number, nuxt?: Nuxt): boolean
```

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

它是这一节**唯一接收 `major: number` 的判断函数**，也就是把"哪一代"当参数而不是当函数名——这正是它被推荐给前两个的原因。

**`isNuxt3`** 与 **`isNuxt2`** 各自只有一句定性（检查当前版本是不是 3.x / 2.x），然后挂同一条 `::note`：

> Use `isNuxtMajorVersion(2, nuxt)` instead. This may be removed in @nuxt/kit v5 or a future major version.

这里有个必须点出来的问题：**同一条 note 被复制到了两节里，其中 `isNuxt3` 那一节写的替代是 `isNuxtMajorVersion(2, nuxt)`**。`isNuxt3` 检查的是 3.x，**照抄那句建议会把它替换成"检查是不是 2"**。两处中只有一处是自洽的（`isNuxt2` 那节）。**替代写法应取 `isNuxtMajorVersion(3, nuxt)`**，这是按各自章节的检查内容推出来的，而不是文档字面给的。

移除时间也只给了模糊范围：**"在 @nuxt/kit v5 或将来某个大版本"**，没有版本号可以盯。

**`getNuxtVersion`** 最简单：**返回当前 Nuxt 版本**，类型是 `(nuxt?: Nuxt) => string`。页面**没有说这个字符串的形态**（是否含 pre-release、是否带 build 元数据），要拿它自己比较时得先打印一次看看。

## 七个函数共用的一条取实例规则

`nuxt` 这个参数在多个条目里重复了同一句：**不提供时，通过 `useNuxt()` 调用从上下文获取**。所以在已经拿到实例的地方直接传，在没拿到的地方可以省掉这个参数——**两种写法等价，前提是上下文里确实有实例**（这正是 `tryUseNuxt` 存在的理由，见上下文那一讲）。

## 它和 meta.compatibility 的分工

模块定义里也能写版本约束（`meta.compatibility.nuxt`），两者不是重复：

| 方式 | 生效时机 | 不满足时的表现 |
| --- | --- | --- |
| `meta.compatibility` | 模块加载阶段由 wrapper **自动检查** | 控制台 **WARN，并且整个模块被禁用** |
| 这一节的函数 | 你在 `setup` 里主动调用 | **由你决定**：警告、分支、或抛错停下 |

所以判据很清楚：**整个模块在新版本才有意义 → 写声明**（用户什么也不用做）；**模块里某一段功能要按版本换实现 → 用函数**（模块照常装，只是走不同分支）。

## 收尾自查

- 有没有用 `isNuxt2` / `isNuxt3`？**换成 `isNuxtMajorVersion(2 或 3, nuxt)`**，注意别照抄 `isNuxt3` 那节给的示例参数。
- 用了 `assertNuxtCompatibility`？**它在 `setup` 里抛错会让构建停下来**，这通常是库不该做的决定。
- `constraints` 里到底要不要管构建器？**只有当某功能只在某个 builder 下可用（或明确不可用）时才写 `bridge`**，值是构建器名加 `false` 或 semver。

下一篇讲自动导入那一节的三个函数：`addImports` 的八个字段、`addImportsDir` 与 `addImportsSources`。

## 下一篇

下一篇 647：Kit 自动导入三函数：addImports 的八字段、Dir 与 Sources

## 本文依据

- https://nuxt.com/docs/4.x/api/kit/compatibility
