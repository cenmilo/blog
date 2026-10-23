---
url: /posts/9dd2e9cbdb4ac15b28bcefe9e3c0d8f9/
title: 822 - 禁 JS 与 noScripts（4）：岛屿与 nuxt-client 在 noScripts 路上的取舍
date: 2026-11-02T12:00:00+08:00
updated: 2026-11-02T12:00:00+08:00
author: cmdragon

summary:
  摘要：把同一块岛屿放进两条路由，一条发了脚本、一条被 noScripts 盖住，两页屏幕上的数字一模一样。按下按钮才知道差别在哪一位上：只有标了 nuxt-client 的那一位会动，而被规则盖住的那一页连这一位也不动，控制台一声不响。这一节把这两句原文量成读数，也记下我一开始把对照组搭错的那次。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - nuxt-client
  - 服务端组件
  - noScripts
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 先看见数字，再看见差别

上一节留了两条同形的路由：`/no-js-lab` 被 `noScripts` 盖住，`/with-js-lab` 什么都不开。这一节往这两页里各放同一块岛屿，页面上立刻多出三行数字：静态那一位写着 `1`，标了 `nuxt-client` 的那一位写着 `3`，页面递给岛屿那个 slot 写着 `0`。

两页看起来完全一样。于是问题变成一个动作：**按下去**。官方对这件事只有两句话，都在讲服务端组件那一节的末尾。

> Server components that only render static HTML work fine, but components hydrated with `nuxt-client` and interactive island slots rely on an inline script to relocate teleported content, so they will not become interactive on a `noScripts` route.

后面紧跟着一句短的：

> Plain static islands are unaffected.

**teleport**指组件把自己的内容渲染到 DOM 里另一个位置（不是它的父节点下面），所以浏览器接手时需要一段脚本把内容挪回原位——那段脚本正是被省掉的东西之一。

## 同一块岛屿放进两页

```vue [app/pages/no-js-lab.vue（与 with-js-lab.vue 同形）]
<IslandInteractive>
  <template #from-page>
    <button type="button" data-testid="page-slot-button" @click="页面槽计数 += 1">
      页面给岛屿的 slot 计数 <span data-testid="page-slot-value">{{ 页面槽计数 }}</span>
    </button>
  </template>
</IslandInteractive>
```

`IslandInteractive.server.vue` 是第七节那条线立好的探针，它内部同时放着两位：`IslandStaticNote`（**没标** `nuxt-client`，整段留在服务端）与 `IslandClientCounter nuxt-client :start="3"`（标了的那一位，由主客户端 app 接手）。加上页面给的 slot，正好凑成三种命运。

```bash
npm run test:e2e -- test/e2e/no-scripts.spec.ts
# 18 passed (49.4s)
```

## 内容一位不差，动的只有一位

先读 HTML。三个位置（`无脚本页` 是 `noScripts` 那条，`对照页` 是带脚本那条）：

| 读的是哪一位 | `/no-js-lab` | `/with-js-lab` | `/docs/...?islandLab=client` |
| --- | --- | --- | --- |
| 岛屿根在不在 | 在 | 在 | 在 |
| 静态那位的独有串 | 在 | 在 | 在 |
| `nuxt-client` 那位的独有串 | 在 | 在 | 在 |
| 页面给的 slot 那颗按钮 | 在 | 在 | 在 |
| 三位的初始数字 | 1、3、0 | 1、3、0 | 1、3、0 |
| 整页字节 | 2541 | 6360 | 9586 |

内容一位都不差，整页字节差 3819——那 3819 全是脚本与资源提示，跟这一节要问的那一块没关系。这正对上"Plain static islands are unaffected"。

再按下去。同一批按钮，各点一次，读前后两个值：

```
[822·点了动不动] {"无脚本页":{"之前":{"静态":"1","客户端":"3","页面槽":"0"},
  "之后":{"静态":"1","客户端":"3","页面槽":"0"},"水合":false,"错误":[]},
 "对照页":{"之前":{"静态":"1","客户端":"3","页面槽":"0"},
  "之后":{"静态":"1","客户端":"4","页面槽":"0"},"水合":true,"错误":[]}}
```

三行读数各有用处：

- 对照页上 `nuxt-client` 那一位 `3 → 4`，另外两位一动不动（静态 `1 → 1`、slot `0 → 0`）。这与选择性水合那一节立的形状逐字相同——**整块岛屿默认是静态的，只有标出来的那一位归主客户端 app**。
- `noScripts` 那一位停在 `3`。同一位按钮，在对照页 +1，在这一页不动，这就是那条规则真正的代价。
- 两边的 `错误` 都是空数组。这一位必须一起读：点了不动，究竟是"没人接手"还是"抛错被吞掉"，看上去一模一样，实际是两件事。

到这一步，会员区那句"禁 JS 时 islands 区块仍出内容"就闭合了：内容在（表一）＋ 不动的只是交互（表二），两半都要有读数，缺一条都算没验。

## 机制那一段，本机没读到

官方把原因归到"一小段内联脚本"。我在判据里把两页 HTML 里所有内联脚本的开头片段并排列出来，想找到那段：

```
[822·内联脚本对照] {"无脚本页片段数":0,"对照页片段数":1,
  "对照页片段":["window.__NUXT__={};window.__NUXT__.config={public:{siteUrl:\"http://localhost:300"],
  "像挪位那段":{"无":0,"有":1}}
```

那位"像挪位"的 1 条是我自己写的关键词判据误算进去的——它匹配 `__nuxt`，命中的其实是那段烤运行时配置的脚本，跟挪位无关。所以本机能确认的只到这一层：**被规则盖住那页一个内联脚本都没有，对照页有一个，而那个不是挪位脚本**。机制细节（挪位脚本长什么样、由谁发出）在这一版里读不到，记成未闭合，不写成结论。

还有一位读法值得单独说。`#hint` 那个 slot（只在 `selectiveClient: 'deep'` 下才递给 `nuxt-client` 组件）我按"HTML 里有没有那段独有串"来读，三页都读到"在"——可上一节那条线量的却是"水合之后 DOM 里没有那颗按钮"。**同一段字符串存在，不等于它在客户端接手后还在**。这条判据因此只打印不断言：字符串存在性读不出它想读的那件事，读法得换成数 DOM 里的元素。

## 那次把对照组搭错的弯路

第一轮跑出来的读数是这样的：

```
[822·摸形状] {"无脚本页":{"岛屿根":1,…,"岛屿请求":[]},
 "对照页":{"水合":true,"岛屿根":0,"静态那颗":0,"客户端那颗":0,"页面槽那颗":0,"岛屿请求":[]}}
```

对照页整块岛屿不见了，而且没有一条岛屿请求、控制台一声不响。第一反应是框架在服务端组件这一步上有什么条件把它吞了。查下去是最普通的错：**我忘了给对照页加上那块组件**——只加到了被规则盖住那一页。

这条弯路值得留一句话：**"对照组什么都没有"的时候，先怀疑对照组本身没搭好，再怀疑被测对象。** 判据这次帮了忙——它把两页分别读数、并排打印，缺位一眼就看得见；如果只断言"实验页不动"，这一轮会绿着通过，而整条线的结论其实没被验过。

## 容易读错的几处

| 你看到的 | 实际是什么 | 该怎么办 |
| --- | --- | --- |
| 两页数字一样，就判断这页不需要脚本 | 一样的是内容，不一样的是能不能按 | 内容位与交互位分开各读一次 |
| 被规则盖住那页点了不动，以为抛错了 | 这一版里 `pageerror` 一条都没有 | 把"不动"与"有没有报错"两位一起读 |
| 岛屿里那位数字动了，以为整块岛屿水合了 | 只有标 `nuxt-client` 的那一位归主 app | 三种命运各留一位读数 |
| HTML 里搜到那段字符串，就断定它在页面上能用 | 服务端渲出的字符串可能是客户端接手后要换掉的 | 数 DOM 里的元素，别只搜字符串 |
| 想证明"挪位脚本被省掉" | 本机 HTML 里找不到那段脚本本身 | 只报到"这一页没有内联脚本"这一层，机制记未闭合 |

## 这一站的三条结论

内容不受影响：`noScripts` 那页整块岛屿连同 `nuxt-client` 那一位与页面给的 slot 都在 HTML 里，初始数字 1、3、0 一位不差。
交互的代价只落在标出来的那一位：对照页 `3 → 4`，`noScripts` 页停在 `3`；而静态那位在两边都不动——它本来就不水合。
会员区那句"禁 JS 时 islands 区块仍出内容"到这里有完整两半的读数，可以结案。

下一站收这条线：四步各留一句、判据按用例数现跑一张分档表、会员区三条验收逐句对照，并把这几站攒下的"本机读不到"逐条列明。

## 本文依据

- `guide/concepts/server-components`（`nuxt-client` 与岛屿 slot 依赖那段内联脚本；"Plain static islands are unaffected"；以及"Only its chunk is shipped to the client; the rest of the island remains static"那句）
- `api/components/nuxt-island`（`<NuxtIsland>` 的 `name`／`props`／`#fallback` 用法，本轮 HTML 里 `data-island-uid`、`data-island-component` 这些位与它同源）
- `guide/going-further/features` 的 noScripts 一段（与服务端组件的相互作用、命中时省掉的四类）
- `guide/recipes/mostly-static-sites`（静态岛屿在 `noScripts` 路上仍工作、`prerender` 的组合写法）
