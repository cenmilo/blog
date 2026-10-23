---
url: /posts/b14eea783e90f8b774cb15ab2db1e3c3/
title: 427 - 内联样式的粒度：features.inlineStyles 的取舍
date: 2027-03-03T10:00:00+08:00
updated: 2027-03-03T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：先纠一个名字：**4.x 里没有 `inlineSSRStyles`**——那是 3.x 时期的键名，4.x 管内联粒度的是 `features.inlineStyles`（在 `features` 节，不在 experimental 里）。官方定义："Inlines styles when rendering HTML. This is currently available **only when using Vite**."，默认值是 `(id) => id.includes('.vue')`，还可以传一个"接收 Vue 组件路径、返回布尔"的函数逐组件决定。为什么要改默认：升级指南里那条 "More Granular Inline Styles"（影响级别 Moderate）说得很清楚——Nuxt 现在**只内联 Vue 组件的样式、不再内联全局 CSS**；此前是全部 CSS 都内联并删掉指向独立 CSS 文件的 `<link>`。官方的理由是平衡：初始加载仍不会有 per-page/per-component 的独立 `.css` 请求，同时**能缓存一个全局 CSS 文件**、并减小首屏文档体积。要退回旧行为就设 `inlineStyles: true`。

categories:
  - 前端开发

tags:
  - nuxt4
  - inlineStyles
  - 内联样式
  - features节
  - 升级
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

搜"inlineSSRStyles"会发现一件尴尬事：在 4.x 文档里**一个字都搜不到**。这个键名是 3.x 时代的，4.x 换成了 `features` 下面的 `inlineStyles`。这一篇把新键讲清，顺带说明默认值为什么长成那样。

## 先改名：4.x 的键在哪

`inlineStyles` 位于 `nuxt.config` 的 **`features`** 节，不是 experimental（57 个实验开关里没有它）：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  features: {
    inlineStyles: false, // or a function to determine inlining
  },
})
```

官方定义两句：

> Inlines styles when rendering HTML. This is currently available only when using Vite.

第二句是个硬约束：**只在用 Vite 时可用**（换 webpack/rspack 构建器这条路不生效）。

## 默认值本身是个函数

同一节给出的默认值很关键：

> You can also pass a function that receives the path of a Vue component and returns a boolean indicating whether to inline the styles for that component.
> It defaults to `(id) => id.includes('.vue')`.

也就是说默认不是布尔，而是一个"路径含 `.vue` 就内联"的判定函数。这个默认值正好解释了下一节的变化。

## 为什么默认只内联组件样式

升级指南里这一条的影响级别是 **Moderate**，标题 "More Granular Inline Styles"，变化一句话：

> Nuxt will now only inline styles for Vue components, not global CSS.

变化前后的对照官方写得很完整：

> Previously, Nuxt would inline all CSS, including global styles, and remove `<link>` elements to separate CSS files. Now, Nuxt will only do this for Vue components (which previously produced separate chunks of CSS). We think this is a better balance of reducing separate network requests (just as before, there will not be separate requests for individual `.css` files per-page or per-component on the initial load), as well as allowing caching of a single global CSS file and reducing the document download size of the initial request.

拆成三段读：

1. **以前**：全部 CSS 内联（含全局样式），并移除指向独立 CSS 文件的 `<link>`。
2. **现在**：只对 Vue 组件这么做（这些组件原本会产出独立 CSS chunk）。
3. **官方的取舍理由**：初始加载仍然**不会**有 per-page / per-component 的独立 `.css` 请求（这点和以前一样省），同时换来两个好处——**一个全局 CSS 文件可以被缓存**、**首屏文档的下载体积更小**。

第 3 点里的对称关系值得记住：把全局 CSS 从文档里挪出去、变成可缓存的独立文件，代价是首屏多一个 CSS 请求，收益是后续访问命中缓存且 HTML 更小。官方认为这个平衡更好，所以设成默认。

## 三个取值对应三种结果

| 取值 | 结果 |
| --- | --- |
| 默认 `(id) => id.includes('.vue')` | 只内联 Vue 组件样式，全局 CSS 走独立文件（可缓存） |
| `true` | 回到旧行为：全局 CSS 也内联（升级指南的迁移写法） |
| `false` | 关闭这个内联行为 |
| 自定义函数 | 逐组件决定内不内联（参数是组件路径） |

升级指南给的"退回旧行为"写法就是显式设 `true`：

> This feature is fully configurable and you can revert to the previous behavior by setting `inlineStyles: true` to inline global CSS as well as per-component CSS.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  features: {
    inlineStyles: true,
  },
})
```

注意这个键的语义是**布尔或函数**，默认值已经是函数——直接写 `true` 是"全部内联"，不是"恢复默认"。想恢复默认就删掉这一行。

自定义函数的用途也就清楚了：按组件路径分档，比如把某类只出现在折叠区下方的组件样式排除在内联之外（参数是 id 字符串，判定规则你自己写）。

## 和 noScripts 的关系

424/353 那条要连起来看：`features.noScripts` 关掉脚本时，**CSS 不受影响**——样式表链接与内联样式照常渲出，页面外观不变，只是永不水合（353 的 features 那节写了这一点）。反过来说，`inlineStyles` 管的是"样式放哪"，`noScripts` 管的是"脚本报不报"，两者互不覆盖。

流式渲染那边也有一条相关口径（324）：路由样式是流式吐出的，壳先发时 head 里只有入口 chunk 的样式与提示，页面/布局模块注册后其 CSS 紧跟壳之后流出（开了 `inlineStyles` 就内联成 `<style>`，否则是样式表链接）——所以这个键也影响流式下的样式落点。

## 自查

- **4.x 没有 `inlineSSRStyles`**；控制内联粒度的是 `features.inlineStyles`，在 `features` 节、**不在 experimental 里**。
- 硬约束：目前**只有用 Vite 时可用**。
- 默认值是函数 `(id) => id.includes('.vue')`，即只内联 Vue 组件样式、全局 CSS 不内联。
- 变化理由：初始加载仍无 per-page/per-component 的独立 `.css` 请求，换来"一个可缓存的全局 CSS 文件"+"首屏文档更小"。
- 想连全局 CSS 一起内联（旧行为）→ 显式 `inlineStyles: true`；关掉→ `false`；逐组件决定→传函数；删掉该行→回默认。
- 它与 `noScripts` 互不覆盖（后者不动 CSS）；流式渲染下它决定路由样式是内联 `<style>` 还是样式表链接。

## 下一篇

下一篇 428 讲把主题值放进 `app.config`：官方示例本身就是 `theme.primaryColor`，配合 `useAppConfig()` 两端可读与 `updateAppConfig` 运行期改，但要注意它会进客户端包、不能放密钥。
