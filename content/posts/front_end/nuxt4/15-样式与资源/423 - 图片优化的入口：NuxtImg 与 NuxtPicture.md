---
url: /posts/042bd18201da43ea40b4ff75d9c81742/
title: 423 - 图片优化的入口：NuxtImg 与 NuxtPicture
date: 2027-03-02T09:00:00+08:00
updated: 2027-03-02T09:00:00+08:00
author: cmdragon

summary:
  摘要：`<NuxtImg>` 不是内置组件——它来自 Nuxt Image 模块，**装并启用之后才存在**（`npx nuxt module add image`）。官方给它的定位是原生 `<img>` 的 drop-in replacement，能力五条：用内置 provider 优化本地与远程图片、把 `src` 转成 provider 优化后的 URL、按 `width`/`height` 自动缩放、给了 `sizes` 就生成响应式尺寸、支持和 `<img>` 一样的原生懒加载及其它属性。输出形态官方写得很明确：**直接产出一个原生 `img` 标签、外面不包任何 wrapper**（`<NuxtImg src="/nuxt-icon.png" />` → `<img src="/nuxt-icon.png" />`）。`<NuxtPicture>` 同理是 `<picture>` 的替代品，用法与 NuxtImg 几乎一致，多的是"可能时提供 webp 这类现代格式"。要逐个属性的细节得去 Image 模块自己的文档。

categories:
  - 前端开发

tags:
  - nuxt4
  - NuxtImg
  - NuxtPicture
  - 图片优化
  - 模块
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

新项目里直接写 `<NuxtImg>`，编辑器报"未知组件"、页面上啥也不渲染——因为这两个组件**不在 Nuxt 核心里**，它们属于 Nuxt Image 模块。这一篇讲清入口、能力范围与输出形态。

## 先装模块

两节的 Setup 段是同一句、同一条命令：

> In order to use `<NuxtImg>` you should install and enable the Nuxt Image module:

```bash [Terminal]
npx nuxt module add image
```

`<NuxtPicture>` 的要求一字不差。**没装这个模块，组件就不存在**——这是 424 会展开的边界之一（模块还能用配置键关掉）。

## NuxtImg 的五条能力

官方定义它是"a drop-in replacement for the native `<img>` tag"，随后逐条列了它会做什么：

- Uses built-in provider to optimize local and remote images
- Converts `src` to provider-optimized URLs
- Automatically resizes images based on `width` and `height`
- Generates responsive sizes when providing `sizes` option
- Supports native lazy loading as well as other `<img>` attributes

翻译成人话，五条各自的落点：

| 官方条目 | 意味着 |
| --- | --- |
| 内置 provider 优化本地与远程图片 | 本地文件和外链图都能过优化器 |
| 把 `src` 转成 provider 优化后的 URL | 你写的路径不是最终发出去的 URL |
| 按 `width`/`height` 自动缩放 | 缩放尺寸由这两个属性驱动 |
| 提供 `sizes` 就生成响应式尺寸 | 响应式是 `sizes` 触发的，不是自动猜 |
| 支持原生懒加载和其它 `<img>` 属性 | `<img>` 能写的它基本能写 |

performance 那一节把这五条又复述了一遍，并额外说明 Image 模块"allows resizing and transforming your images using built-in optimizer or your favorite images CDN"——**内置优化器或你自己的图片 CDN** 两条路。

## 输出形态：就是一个 img

官方专门写了这条，因为它影响样式和结构：

> `<NuxtImg>` outputs a native `img` tag directly (without any wrapper around it). Use it like you would use the `<img>` tag:

```html
<NuxtImg src="/nuxt-icon.png" />
```

结果是：

```html
<img src="/nuxt-icon.png" />
```

**没有额外包裹层**。所以按元素/类名写的 CSS 选择器不用为它多算一层——这也符合 383 那条动画要求（要动画的页/布局必须单根）：`<NuxtImg>` 本身就是单个元素。

## NuxtPicture：多的一件事

`<NuxtPicture>` 官方定性是"a drop-in replacement for the native `<picture>` tag"，并给了它与 NuxtImg 的关系：

> Usage of `<NuxtPicture>` is almost identical to `<NuxtImg>` but it also allows serving modern formats like `webp` when possible.

一句概括差别：**用法几乎相同，多的是能在可能时提供 `webp` 这类现代格式**。要 `<picture>` 元素本身的语义细节，官方指向 MDN。

## 属性细节不在这两节

这两节都是**概要级**：各自末尾一条 `::read-more` 指向 Image 模块文档（`image.nuxt.com/usage/nuxt-img`、`.../nuxt-picture`）。所以：

- **不要指望在 nuxt.com 的 4.x 参考里找到 `<NuxtImg>` 的完整属性表**——这两节没有表格，只有上面那五条能力清单。
- 页面属性行为（`format`、`preload`、`fetchpriority` 等）在 performance 那一节的示例里出现过零散用法，但成体系的参数说明归 Image 模块文档。

这个边界很重要：写方案时按"模块文档为准"，而不是拿 Nuxt 核心参考页当属性清单。

## 自查

- `<NuxtImg>`/`<NuxtPicture>` 来自 **Nuxt Image 模块**，要先 `npx nuxt module add image`；没装就没有这两个组件。
- NuxtImg 官方五条能力：内置 provider 优化本地/远程图、`src` 转成优化后 URL、按 `width`/`height` 自动缩放、给 `sizes` 才生成响应式尺寸、支持原生懒加载与其它 `<img>` 属性。
- 输出**原生 `img` 标签、无 wrapper**，像用 `<img>` 一样用。
- NuxtPicture 用法与 NuxtImg 几乎一致，额外能在可能时提供 `webp` 等现代格式。
- 这两节是概要级，**没有属性表**；逐属性细节在 Image 模块自己的文档里。

## 下一篇

下一篇 424 划模块的能力边界：配置键 `image: false` 怎么在图层里关掉它、以及官方写明**默认关闭**的那个 `experimental.prefetchPreloadTags`（转发 `<NuxtImg preload>` 这类提示）为什么默认关。
