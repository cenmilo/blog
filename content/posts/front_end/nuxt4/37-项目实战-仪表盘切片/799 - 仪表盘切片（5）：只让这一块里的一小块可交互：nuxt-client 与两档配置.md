---
url: /posts/8b2ec791fe909e3834ddd98e107a3b6f/
title: 799 - 仪表盘切片（5）：只让这一块里的一小块可交互：nuxt-client 与两档配置
date: 2026-10-29T13:00:00+08:00
updated: 2026-10-29T13:00:00+08:00
author: cmdragon

summary:
  摘要：服务端组件整块是静态的，但可以在里面点名某一位组件让它由主客户端 app 接手——这个属性叫 `nuxt-client`，要显式开 `experimental.componentIslands.selectiveClient` 才生效。这一站把三位放在一起量：没标的点不动、标了的点得动、页面递给岛屿的那段 slot 在文档里说"是活的"而本站量出来点不动。顺带量到两件与文档说法不同的形状：没标的那一位代码也进了客户端包，内置组件挂了 `nuxt-client` 之后链接壳留着但文字消失、不报错。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 服务端组件
  - 选择性水合
  - nuxt-client
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一块里，三个按钮，两种命运

打开 `http://localhost:3100/docs/getting-started?islandLab=client`，页面最底下那一段里有三个按钮。挨个点，读数是这样的：

```text
[选择性水合·交互] {"档位":"true","静态":{"前":"1","后":"1","点了几次":1},"客户端":{"前":"3","后":"4","点了几次":1},"页面槽":{"前":"0","后":"0","点了几次":1},"岛屿槽按钮数":0,"控制台":["error:Hydration completed but contains mismatches."]}
```

- 没标的那一位：数字纹丝不动。
- 标了 `nuxt-client` 的那一位：3 变 4，点得动。
- 页面递给岛屿的那一位：0 不动，控制台还留一条水合错位。

官方讲选择性水合那一节的原话是：「The component marked with `nuxt-client` is server-rendered as part of the island, then hydrated by the main client app. Only its chunk is shipped to the client; the rest of the island remains static.」这一句里包着三件事——**标住的活、没标的不活、代码只进标住那位的**。本站量到前两件事成立，第三件事不成立，形状在下面。

## 这一档要动配置：`componentIslands` 第一次被写出来

前四站都不用配：`experimental.componentIslands` 从没出现在 `nuxt.config.ts` 里，默认 `'auto'` 就够——有一个 `.server.vue` 就启用。`nuxt-client` 是这一条线上**第一件必须显式配置**的事，那一节的原话是「This requires `experimental.componentIslands.selectiveClient` to be enabled」。

```ts
experimental: {
  // 780 的实验台，没设环境变量时整位不写，让 Nuxt 用默认值
  ...(process.env.NUXT_PAYLOAD_MODE ? { payloadExtraction: /* … */ } : {}),
  componentIslands: {
    selectiveClient: process.env.NUXT_SELECTIVE_CLIENT === 'deep' ? 'deep' as const : true,
  },
},
```

两件事要留意。写成对象之后 `remoteIsland` 取它自己的默认值 `false`，远程岛屿仍要显式 `'local+remote'` 才开；而 `selectiveClient` 的两档（`true` 与 `'deep'`）不能用一个 `v-if` 切，只能各构建一次，所以本站把它挂在环境变量上：`NUXT_SELECTIVE_CLIENT=deep npm run build`。

新组件长这样：`app/components/IslandInteractive.server.vue`（第三个 `.server.vue`）里放两个子组件，其中一个带 `nuxt-client`：

```vue
<IslandStaticNote />
<IslandClientCounter
  nuxt-client
  :start="3"
/>
```

## 「Only its chunk is shipped」这句在本机不是"只有"

每个子组件的模板里留了一段独有串（`ISLAND_STATIC_799` 与 `ISLAND_CLIENT_799`），编译之后它们只出现在各自那个组件的产物里，于是能问出"谁进了客户端包"：

```text
[选择性水合·产物] {"档位":"true","客户端静态":{"文件数":64,"命中数":1,"命中":["BJjOsYjC.js"]},"客户端交互":{"命中数":1,"命中":["fYKPbfgi.js"]},"服务端静态命中数":1,"服务端交互命中数":1}
```

两位**都**在客户端包里各命中一个文件（服务端那两位也各自命中，这是正对照，防止"扫描找不到东西"报出假结论）。行为上没标的那位确实不活，可它的代码照样下载了。

再看整页的代价（同一份基线判据，798 之后与 799 之后）：

| 读数 | 开 `selectiveClient` 之前 | 开之后 |
| --- | --- | --- |
| 这一页下载的脚本条数 | 15 | 31 |
| 下载原始字节合计 | 322,143 | 327,594 |
| 客户端 chunk 数 | 20 | 64 |
| 整页响应字节 | 5,736 | 6,973 |
| 那段正文的字节 | 255 | 255 |

字节只涨 1.7%，但**请求数翻倍、chunk 从 20 涨到 64**：开这一档把打包方式改了，不是把某一块拿掉。795 那条"渲染器在客户端 0 命中"仍然成立，所以这句话要这样带走——**服务端组件真正省的是"没被引进来的依赖"，不是"岛屿里那几个组件的代码"**。

## 岛屿给 `nuxt-client` 组件传的 slot：响应里有，DOM 里没有

`'deep'` 那一档换到的是这件：「Setting `selectiveClient: 'deep'` additionally allows passing slots to `nuxt-client` components. Those slots are rendered on the server and are not interactive on the client.」于是给那个可交互子组件传了一个 `#hint` slot，并分三层去问它在不在：

```text
[选择性水合·slot] {"档位":"true","DOM按钮数":0,"HTML里有按钮":true,"HTML里有独有串":true,"服务端产物命中":["build\\IslandInteractive.server-Ba110yj0.mjs"]}
```

服务端渲了（产物与那份 HTML 都带着它），浏览器接手之后 DOM 里没有。换成 `'deep'` 再跑一次，读数一字不差：

```text
[选择性水合·slot] {"档位":"deep","DOM按钮数":0,"HTML里有按钮":true,"HTML里有独有串":true,"服务端产物命中":["build\\IslandInteractive.server-DAb72rHf.mjs"]}
```

所以本站这一位**量不出 `true` 与 `'deep'` 的差别**——那句 "additionally allows" 在这个形状下没兑现。判据钉的是读到的形状（`DOM按钮数: 0`），注释里留着那句文档：哪天它渲出来了，判据会红，红了就回来改这里的说法。

## 页面递给岛屿的 slot：在 DOM 里，也点不动

同一节的 Slots 那段说得是反方向的：「Slot content is provided by the parent, so it belongs to the main client app and **is** interactive (it is wrapped in a `<div>` with `display: contents;`)。」

```text
[选择性水合·页面slot] {"档位":"true","前":"0","后":"0","点了几次":1}
```

这一位确实渲出来了、也在岛屿那个根里面、父级的 `display` 也正是 `contents`，但点三次还是 0；整页加载那一趟还留一条 `Hydration completed but contains mismatches.`。换 `'deep'` 再点，同样不动。**"内容看得见"与"事件挂得上"是两件事**，这一位就是只有前者。

## 内置组件挂 `nuxt-client`：链接壳留着，文字没了

两处文档都在拦这一件事。选择性水合那节的 warning：「Use `nuxt-client` only on local `.vue` SFCs. Built-ins like `<NuxtLink>` skip the islands transform. After client navigation you may see `Failed to locate Teleport target`, or the link disappears with no error.」讲 experimental 开关那节的 note：「Skip `nuxt-client` on non-SFC components such as `<NuxtLink>`.」

这一档必须故意写错才量得到，所以是临时变异（加一行 `<NuxtLink to="/docs" nuxt-client>…</NuxtLink>`，跑完就删）。量出来比"消失"更具体：

```text
[内置·整页] {"链接数":1,"点之后地址":"/docs","行":["error:Hydration completed but contains mismatches."]}
[内置·点进来] {"链接数":1,"形状":{"href":"/docs","标签":"A","在岛屿内":true,"可见":true},"点击结果":"点击报错：locator.click: Timeout 2000ms exceeded.","跳没跳":"没跳","补测":{"尺寸":"0x21","中心命中的标签":"(命不中)","父级display":"contents"}}
```

地址栏直接进这一趟，链接是好的，点了跳到 `/docs`。**从列表页点进来之后**，那个 `<a>` 还在、`href` 还是 `/docs`，但 Playwright 报的点击超时里带着解析到的节点：`locator resolved to <a class="" href="/docs" data-testid="builtin-link"></a>`——**标签里是空的**，尺寸 `0x21`，正中心根本命不中它。也就是说文档那句"链接消失"在本机的形状是：**壳留着、文字没了、点不到、一声不响**（控制台这一趟没有任何报错）。

绕法就是文档给的那句：只把 `nuxt-client` 挂在自己的 `.vue` 文件上——要一个可交互的链接，就写一个自己的 `MyLink.vue` 包住 `<NuxtLink>`，把 `nuxt-client` 挂在那层包装上。

## 排错表

| 撞上的现象 | 先查这里 | 为什么 |
| --- | --- | --- |
| 岛屿里的按钮点了没反应 | 那一位有没有标 `nuxt-client` | 默认整块静态，事件不挂 |
| 标了还是不活 | `experimental.componentIslands.selectiveClient` 有没有开 | 不开时 `nuxt-client` 这个属性不认 |
| 客户端包反而变大 | chunk 数与请求数，而不是只看字节 | 开这一档改的是打包切分方式 |
| slot 内容看不见 | 分三层问：产物、响应 HTML、DOM | 服务端渲了不等于浏览器留得住 |
| 链接还在但点不到 | 那个 `<a>` 里面还有没有文字 | 内置组件跳过 islands 改写，slot 搬不过来 |
| 只有 `nuxt-client` 那段能点 | 用的是内置组件还是自己的 SFC | 文档明写只支持本地 `.vue` |

## 没量到的那一段

同一节还有一句：在带 `noScripts` 的路由上，那段搬移 teleport 内容的内联脚本会被省掉，于是完整的 `nuxt-client` 组件不会水合。本站没有任何 `noScripts` 路由，这条本轮没验，不能当已证结论用。

## 自己动手跑一遍

```bash
npm run build
E2E_PORT=3319 npm run test:e2e -- test/e2e/island-selective-client.spec.ts
NUXT_SELECTIVE_CLIENT=deep npm run build
E2E_PORT=3320 npm run test:e2e -- test/e2e/island-selective-client.spec.ts
```

两趟各预期 `4 passed`。手动看的话：开 DevTools 的 Network，进 `?islandLab=client` 那一页，先点"点静态位"（数字不动），再点"点客户端位"（数字动）。想看链接那一档就得自己复现那次变异：在 `IslandInteractive.server.vue` 的根里加一行 `<NuxtLink to="/docs" nuxt-client>内置组件挂 nuxt-client</NuxtLink>`，重新构建，然后分别从"地址栏直接进"与"从列表页点进来"两次——前者点了能跳，后者那个 `<a>` 里没有文字、点不下去，控制台也不报错；看完把那行删掉。

## 本文依据

- 官方：讲服务端组件的那一节（选择性水合那一小节的三句原话、`selectiveClient` 与 `'deep'` 的差别、Slots 那段"父级提供的 slot 是活的、包在 `display: contents` 的 div 里"、内置组件挂 `nuxt-client` 的 warning 与包一层的绕法、`noScripts` 那条交互）；讲 experimental 开关的那一节（`componentIslands` 的取值形状，以及那句「Skip `nuxt-client` on non-SFC components such as `<NuxtLink>`」）；讲 `<NuxtIsland>` 的那一节（`props` 与 `context` 走查询串）
- 仓库：`nuxt.config.ts`（本站第一次显式写 `componentIslands`）、`app/components/IslandInteractive.server.vue`、`app/components/IslandStaticNote.vue`、`app/components/IslandClientCounter.vue`、`app/pages/docs/[slug].vue`（`client` 那一档）、`app/pages/docs/index.vue`（第四条实验链接）、`test/e2e/island-selective-client.spec.ts`（新增 4 条）、`test/e2e/console-baseline.spec.ts`（那位名单更新为五位）
- 上一批：[798 - 仪表盘切片（4）：客户端导航会多一次岛屿请求，lazy 与 fallback 才不挡](/posts/c24ab414eeafab5f696b479c4a8b3d4e/)
- 下一篇：[800 - 仪表盘切片（6）：这条线的读数收口，与要真会话才能补的那几行](/posts/231ea1560f3f1ce2151b45c4cbf4c49d/)
