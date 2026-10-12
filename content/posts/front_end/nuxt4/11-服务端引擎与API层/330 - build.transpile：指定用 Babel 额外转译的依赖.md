---
url: /posts/5b296f9c17e87a7f34c7dda2343c3c69/
title: build.transpile：指定用 Babel 额外转译的依赖
date: 2027-01-30T09:00:00+08:00
updated: 2027-01-30T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：node_modules 里的依赖默认不额外转译，偶尔某个依赖的代码太新、或你希望它也走一遍项目的转译管线，就用 build.transpile 把它列进去。这一篇按 nuxt-config 那节讲它的条目形状：数组里的每一项可以是包名、函数、字符串或匹配依赖文件名的正则对象；也能给一个函数按条件转译，函数收到一个上下文对象、返回要转译的包名或 false。示例 `transpile: [({ isLegacy }) => isLegacy && 'ky']` 就是"只有 legacy 构建时才转译 ky"。要如实说明：这一节文档很薄、函数参数的字段官方写成空括号没列全。

categories:
  - 前端开发

tags:
  - nuxt4
  - build.transpile
  - Babel
  - 转译
  - nuxt.config
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

`build` 是 nuxt.config 里的构建配置节，`transpile` 是它下面一个数组。这一节文档写得很短，我按原样把它能确定的讲清，不确定的地方如实标出来。

## 它是干什么的

官方这一节的全文（`build.transpile`）：

> If you want to transpile specific dependencies with Babel, you can add them here. Each item in transpile can be a package name, a function, a string or regex object matching the dependency's file name.
>
> You can also use a function to conditionally transpile. The function will receive an object ().

类型 `array`。

把它翻成能操作的规则：

- 用途：**让指定的依赖也经过 Babel 转译**（默认这些第三方依赖不被额外转译）。
- 数组每一项可以是四种之一：**包名**、**函数**、**字符串**、或**匹配依赖文件名的正则对象**。
- 还能**按条件**转译：给一个函数，它收到一个上下文对象，返回要转译的包名或 `false`。

典型场景：某个 npm 包用了很新的语法/特性、或你希望它享受你项目里的某些转译（比如特定的语法降级、路径处理），就把它列进 transpile。

## 官方示例

nuxt-config 给的例子正好演示"函数条件转译"：

```ts
export default defineNuxtConfig({
  build: {
    transpile: [({ isLegacy }) => isLegacy && 'ky'],
  },
})
```

读法：这个函数收到一个对象（示例里解构了 `isLegacy`），当 `isLegacy` 为真时返回字符串 `'ky'`（即"legacy 构建才转译 `ky` 这个包"），否则返回假值、不转译。

## 一处要如实说明的空白

官方那句 "The function will receive an object ()" 的**括号是空的**——文档在这一节没有把函数收到的对象有哪些字段列出来。示例里只出现了 `isLegacy` 这一个。所以：

- **能确定的**：函数参数是个对象、示例用到 `isLegacy`（是否 legacy 构建）。
- **文档没写的**：这个对象除 `isLegacy` 外还有哪些字段、函数返回值除"包名/false"外还能是什么。我不替它编字段清单——要确认真实可用的上下文，去项目里打印这个对象、或查 Nitro/Vite 相关转译文档。

这一节整体就是很薄的一节，我不把它扩成一张其实不存在的"字段表"。

## 和相邻的转译/构建配置别混

`build` 节下还有别的键（比如 `transpile` 同级），但这一篇只讲 transpile。相关坐标记一下：

| 键 | 管什么 |
| --- | --- |
| `build.transpile` | 指定哪些**第三方依赖**额外走 Babel 转译（本篇） |
| `sourcemap` | 服务端/客户端产物是否生成 sourcemap（另节，默认 server:true / client:false） |
| `nitro` | 服务端引擎 Nitro 的配置（320） |

依赖默认不转译这件事，和 071 讲的"node_modules 里不启用自动导入"是同一类"第三方默认不处理"的取向——你要某个依赖特殊处理，就显式列进 transpile。

## 自查

- 某个 node_modules 依赖要额外转译 → 把它的包名加进 `build.transpile`（数组，每项可以是包名/函数/字符串/正则）。
- 按条件转译给函数：返回要转译的包名或假值；示例是 `({ isLegacy }) => isLegacy && 'ky'`。
- 这一节文档很薄，函数收到的对象除 `isLegacy` 外没列全——别当它有完整字段表，需要时自己在项目里验。

## 下一篇

下一篇 331 讲 `.output` 目录与 preset：`nuxt build` 产出独立于 node_modules 的 `.output`，靠 preset 选目标运行环境；并说清 4.x 没有逐厂商的 deployment 文档目录。

## 本文依据

- nuxt-config build.transpile 节（`api/nuxt-config`）：全文两句（"If you want to transpile specific dependencies with Babel ... Each item in transpile can be a package name, a function, a string or regex object matching the dependency's file name."；"You can also use a function to conditionally transpile. The function will receive an object ()."）；类型 array；示例 `transpile: [({ isLegacy }) => isLegacy && 'ky']`。函数参数对象字段官方未列全（如实标注）。
- nuxt-config sourcemap / nitro 节：相邻构建配置对照（见 320）。
