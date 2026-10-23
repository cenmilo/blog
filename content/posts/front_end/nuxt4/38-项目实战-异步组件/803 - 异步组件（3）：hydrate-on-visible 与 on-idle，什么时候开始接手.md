---
url: /posts/487701fc18420c702a92491d721bb9ad/
title: 803 - 异步组件（3）：hydrate-on-visible 与 on-idle，什么时候开始接手
date: 2026-10-30T11:00:00+08:00
updated: 2026-10-30T11:00:00+08:00
author: cmdragon

summary:
  摘要：同一个组件挂在文档页文章最后，两种写法各读一遍。挂 `hydrate-on-visible`：内容看得见、按钮点得着，但数字一次都不变，直到把它滚进视口，66 毫秒后计数翻成 1，再点才加。挂 `hydrate-on-idle`：根本没滚动，它在开局第 64 毫秒就接手了（那一页的 `domComplete` 是 35 毫秒）。这一站顺手把上一站留的那条账也结了：延迟水合推不后 HTML 头部那条 prefetch，代码照样在第 32 个请求就下来。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 异步组件
  - 延迟水合
  - 首屏JS
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上一模一样，点下去不一样

访问 `http://localhost:3100/docs/getting-started?islandLab=visible`，滚到文章最后，能看到一段文字"延迟水合面板 LAB_HYDRATE_803：这一段一直看得见"，底下一个按钮"点我计数"，再下面一行"点了 0 次"。

点一下按钮。**那一行还是"点了 0 次"**。按钮不是坏的，事件也不是没写——这一位还没被浏览器接手（水合）而已。上面那行"这块接手次数：0"就是这一站的读数口。

同一位组件换成 `?islandLab=idle`，**根本不用滚动**，它还躺在视口外就已经接手了：那一行变成"这块接手次数：1"，点一下数字就加。

两种写法在官方那里是两条策略：

- `hydrate-on-visible`：「Hydrates the component when it becomes visible in the viewport.」底下那条 note 补了实现：「Under the hood, this uses Vue's built-in `hydrateOnVisible` strategy.」
- `hydrate-on-idle`：「Hydrates the component when the browser is idle. This is suitable if you need the component to load as soon as possible, but not block the critical rendering path.」

同节开头还有一句限定的话：**每个 lazy 组件只能用一种策略**（「Only one strategy can be used per lazy component.」）。

## 为什么每一位都要带一个真按得动的按钮

上一站留了个说法要检验，而这一站最先要解决的是**怎么知道它接手了**。

面板的内容由服务端渲进 HTML，所以 DOM 里 `在页面上` 一直是 1，光看元素在不在分不出水合前后。唯一的硬证据是行为：接手之前点它不动，之后点一次加一次。新组件 `app/components/LabHydratePanel.vue` 就干这件事：

```vue
<script setup lang="ts">
const 点击次数 = ref(0)
</script>

<template>
  <div data-testid="hydrate-panel">
    <p data-testid="hydrate-note">延迟水合面板 LAB_HYDRATE_803：这一段一直看得见</p>
    <button data-testid="hydrate-btn" type="button" @click="点击次数 += 1">点我计数</button>
    <p data-testid="hydrate-value">点了 {{ 点击次数 }} 次</p>
  </div>
</template>
```

页面 `app/pages/docs/[slug].vue` 里两种策略各挂一档，`@hydrated` 事件负责把"接手"这件事变成一行可读的数：

```vue
<template v-if="实验档 === 'visible'">
  <p data-testid="visible-count">这块接手次数：{{ 可见档接手 }}</p>
  <LazyLabHydratePanel hydrate-on-visible @hydrated="可见档接手 += 1" />
</template>
<template v-if="实验档 === 'idle'">
  <p data-testid="idle-count">这块接手次数：{{ 空闲档接手 }}</p>
  <p data-testid="idle-moment">接手时刻：{{ 空闲档时刻 }}</p>
  <LazyLabHydratePanel hydrate-on-idle @hydrated="记空闲接手" />
</template>
```

`/docs` 那一页加了两个入口（`lab-visible`、`lab-idle`）。判据是新文件 `test/e2e/async-component-hydration.spec.ts`，五条。

这里有个坑值得单独说：判据里点按钮用的是 `el.click()`，不是 Playwright 的 `locator.click()`。后者会先把元素滚进视口，**那一滚本身就触发了 `hydrate-on-visible`**——测出来的"点了没反应"就不是"没接手"造成的了。

## 五条读数，原样打印

```text
[延迟水合·视口外] {"位置":{"有这一位":true,"距视口顶":716,"视口高":500,"在视口内":false},"接手":" 这块接手次数：0","点得着":true,"点了之前":" 点了 0 次 ","点了之后":" 点了 0 次 ","面板文件":["CJR9hfi9.js"],"第一趟请求条数":38}
[延迟水合·滚进来] {"位置":{"有这一位":true,"距视口顶":387,"视口高":500,"在视口内":true},"接手":" 这块接手次数：1","等待毫秒":66,"之前":" 点了 0 次 ","之后":" 点了 1 次 "}
[延迟水合·空闲] {"刚就绪":" 这块接手次数：1","接手":" 这块接手次数：1","等待毫秒":5,"时刻":" 接手时刻：64","位置":{"有这一位":true,"距视口顶":753,"视口高":500,"在视口内":false},"关键节点":{"domComplete":35,"domInteractive":12},"点了":" 点了 1 次 ","第一趟请求条数":39}
[延迟水合·下载] {"面板文件":["CJR9hfi9.js"],"modulepreload条数":29,"prefetch条数":4,"面板在prefetch里":true,"面板在modulepreload里":false,"命中位次":32,"第一趟请求条数":38,"位置":{"有这一位":true,"距视口顶":716,"视口高":500,"在视口内":false},"接手":" 这块接手次数：0"}
[延迟水合·关脚本] {"可见档有那一段":true,"空闲档有那一段":true,"可见档HTML字节":6970,"空闲档HTML字节":7005}
```

`5 passed`；全量端到端连跑两趟都是 `190 passed／57 skipped`。

## 视口外那一次点击为什么丢了

第一组是可见档。开局 `距视口顶 716`、`视口高 500` ⇒ 这一位在视口外，`接手` 是 0，`点得着: true`（按钮确实存在、`click()` 也确实发出去了），但 `点了之后` 还是"点了 0 次"。事件处理器还不存在，那一次点击就掉在地上了。

把它滚进视口（滚完 `距视口顶 387`、`在视口内: true`），计数在 **66 毫秒**内翻成 1；再点一次，"点了 1 次"。同一位组件、同一个按钮，差别只有一件事：它有没有进过视口。

这一条差点把自己骗过去。第一次跑的时候判据直接红在"开局必须在视口外"上，读数是 `距视口顶 716`／`视口高 720`——**默认视口只比它矮 4 像素**，`IntersectionObserver` 一上来就判它可见，于是接手早就发生了，我看到的"没接手"根本不存在。处理是把这一份判据的视口压到 500 高（`test.use({ viewport: { width: 1280, height: 500 } })`），比去改页面内容便宜，也不动别人的读数。规矩落在这儿：**凡是结论挂在"在不在视口内"上的判据，必须把 `距视口顶` 与 `视口高` 一起打出来**，只打一个布尔值的话，4 像素这种形状看不出来。

## 空闲档不等视口，只等空闲

第二组里 `位置.在视口内: false`（距视口顶 753），没有滚动，但第一次读 `刚就绪` 那一行就已经是"这块接手次数：1"，`点了` 是"点了 1 次"。它自己填的接手时刻是 **64 毫秒**，而这一页的 `domComplete` 是 35 毫秒、`domInteractive` 是 12 毫秒 ⇒ 接手排在"页面已经可用"之后 29 毫秒。满载那一趟读到的是 79 对 47，形状一样。

诚实的边界要说清楚：本机这一页太小，空闲窗口来得比一次点击还快，**"不挡关键渲染路径"这半句量不出挡不挡**，只量出"它确实排在 domComplete 之后"。真要看出代价差异，得在一整屏都是这种组件的页面上测，本站没有那种页。这一条在下面排错表里也标了。

顺带记一个模板层的形状：接手时刻这一位一开始一直读到 `-1`。原因是内联处理器写成两句（`空闲档接手 += 1; 空闲档时刻 = …`）之后，**第二句没有生效，只有第一句被走完**（计数翻了、时刻没动）。改成 `@hydrated="记空闲接手"` 调一个函数，两位才对上。判据里那条"接手时刻要读得到一个非负数"就是替这件事站岗的。

## 把上一站那条账结掉：延迟水合推不后下载

`[延迟水合·下载]` 那一行是给 802 收尾的：面板还挂在视口外、`接手` 明确是 0，此时它的文件已经在这趟请求的第 **32** 个下来了（整趟 38 条），HTML 头部那 4 条 `rel="prefetch"` 里就有它，而 29 条 `modulepreload` 里没有它。

也就是说：加了 `hydrate-on-visible`，**下载一次都没少**。三站合起来是一张分工表：

| 你写的东西 | 它管的事 | 它不管的事 |
| --- | --- | --- |
| `Lazy` 前缀 | 代码单独成一个文件（801 那一位量到 950 字节，本站这一位 4 条 prefetch 里也有它） | 什么时候下载 |
| `v-if` | 渲不渲、跑不跑 | 什么时候下载 |
| `hydrate-on-visible` / `-on-idle` | 什么时候开始能交互 | 什么时候下载、内容在不在 |

这条线读到这儿，"少发点 JavaScript"这个目标还没有任何一种写法兑现过——它归打包与入口依赖管，不归这三种写法管。806 收口时按这张表结账。

## 关掉脚本，内容一行都不少

最后一条判据不跑浏览器脚本，直接读服务端回的那串字节：`可见档有那一段: true`、`空闲档有那一段: true`，两档分别是 6,970 与 7,005 字节。延迟水合延迟的是交互能力，**不延迟内容**——爬虫那边收不收得到先放到一边，用屏幕阅读器的人、脚本被扩展拦掉的人，读到的都是完整文字。

## 排错表

| 想确认的东西 | 这一站用的读法 | 读不到时先怀疑什么 |
| --- | --- | --- |
| 组件是不是真没接手 | 点一下看数字变不变，别只看元素在不在 | 点击会不会先把元素滚进视口 |
| 它在不在视口内 | `距视口顶` 与 `视口高` 一起打 | 只打布尔值会漏掉"露出 4 像素"这种形状 |
| 什么时候接的手 | `@hydrated` 里记 `performance.now()`，再与 `domComplete` 比 | 内联处理器只走第一句 |
| 延迟水合有没有省下载 | 该位的请求位次＋HTML 的 prefetch 名单 | 只数总条数（别的 chunk 也在动） |
| 内容有没有被延迟掉 | 不执行脚本读一次 HTML | 把延迟水合当成 `server: false` |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/async-component-hydration.spec.ts
```

预期 `5 passed`。手动版：开两个标签，一个 `?islandLab=visible`、一个 `?islandLab=idle`，都停在页面顶部。可见档那个先点按钮（它还在视口外），数字不动；滚到底，"这块接手次数"变 1，再点就加了。空闲档那个不滚，直接点，数字立刻就动。想看 4 像素那个坑，把窗口拉高到刚好让面板露头，可见档会在你滚动之前就先接手。

## 本文依据

- 官方：讲组件目录的那一节（`Hydration Strategies` 开头"每位 lazy 组件只能用一种策略"那句、`hydrate-on-visible` 与 `hydrate-on-idle` 两段的原话与两条 `Under the hood` note、`::warning` 那三条限制、`Listening to Hydration Events` 那句 `@hydrated`、五条注意里的"首屏内容别用延迟水合"）；Vue 文档的 `hydrateOnVisible`／`hydrateOnIdle` 两个策略名（本站只引到实现层名字，没引它们的选项细节）
- 仓库：`app/components/LabHydratePanel.vue`（新）、`app/pages/docs/[slug].vue`（两种策略、两个计数与 `记空闲接手`）、`app/pages/docs/index.vue`（`lab-visible`、`lab-idle` 入口）、`test/e2e/async-component-hydration.spec.ts`（新增 5 条）
- 上一批：[802 - 异步组件（2）：只加前缀不够，没有条件渲染它照样提前下载](/posts/092009a90d9d7b0a3aec6cb84c3213a4/)
- 下一篇：[804 - 异步组件（4）：hydrate-after、hydrate-when 与 hydrate-never](/posts/9f594ffd2053b8479f2f61f36ff76ca1/)
