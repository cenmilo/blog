---
url: /posts/9f594ffd2053b8479f2f61f36ff76ca1/
title: 804 - 异步组件（4）：hydrate-after、hydrate-when 与 hydrate-never
date: 2026-10-30T12:00:00+08:00
updated: 2026-10-30T12:00:00+08:00
author: cmdragon

summary:
  摘要：这一站把接手时机交给人来定：`:hydrate-after="1200"` 真的等满了那 1200 毫秒（第 800 毫秒读到的是"计数 0、点一下数字不动"，接手时刻 1382）；`:hydrate-when` 跟着一个布尔走，布尔为假时它一直是 0；`hydrate-never` 等了 2500 毫秒确实一次都没接手，但只改一次 props 就当场翻成 1——官方那条 note 讲的正是这件事。三种档的面板全在视口外，也就顺手读到"这三种与时机无关的东西不牵扯视口"。

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

## 三颗按钮，三种"什么时候能点"

`http://localhost:3100/docs/getting-started?islandLab=after`，页面打开后什么都不用做。文章最后有一段"定时机面板 LAB_HYDRATE_804，记号是 固定记号"，底下一个"点我计数"的按钮，再下面一行"点了 0 次"。上面还有一行是这一站的读数口：`after 这块接手次数：0`。

同一位组件在另外两种档里配的是别的东西：`?islandLab=when` 那里多一颗"把开关打开"的按钮，`?islandLab=never` 那里多一颗"换一次记号"的按钮。三种写法在官方那一节里是三段，每段就一句：

- `hydrate-after`：「Hydrates the component after a specified delay (in milliseconds).」示例是 `:hydrate-after="2000"`。
- `hydrate-when`：「Hydrates the component based on a boolean condition.」示例是 `:hydrate-when="isReady"` 加一个把它设成真的函数。
- `hydrate-never`：「Never hydrates the component.」

`hydrate-never` 看起来最简单，也最容易读错。同节开头有一条 note 直接给它打了折扣：

> Any prop change on a lazily hydrated component will trigger hydration immediately. (e.g., changing a prop on a component with `hydrate-never` will cause it to hydrate)

这一站就是要把这句量出来。

## 新面板要接得住一个 prop

803 那一位面板不接 props，note 里说的"改一次 props"在它身上无处可改，所以这一站另开一位组件 `app/components/LabHydrateTiming.vue`：

```vue
<script setup lang="ts">
defineProps<{ mark?: string }>()
const 点击次数 = ref(0)
</script>

<template>
  <div data-testid="timing-panel">
    <p data-testid="timing-note">定时机面板 LAB_HYDRATE_804，记号是 {{ mark ?? '没传' }}</p>
    <button data-testid="timing-btn" type="button" @click="点击次数 += 1">点我计数</button>
    <p data-testid="timing-value">点了 {{ 点击次数 }} 次</p>
  </div>
</template>
```

它和 803 那一位一样自带一个真按得动的按钮——只看内容在不在分不出接手前后，**数字变不变**才是证据。页面 `app/pages/docs/[slug].vue` 里三档各挂一位，props 都写在模板上（同节 `::warning` 那条限制留给下一站撞）：

```vue
<LazyLabHydrateTiming mark="固定记号" :hydrate-after="1200" @hydrated="记延时接手" />
<LazyLabHydrateTiming mark="固定记号" :hydrate-when="就绪开关" @hydrated="记条件接手" />
<LazyLabHydrateTiming :mark="记号" hydrate-never @hydrated="记从不接手" />
```

三个 `@hydrated` 处理器都是**函数**，不是模板里直接写两句：上一站量到内联处理器只走第一条语句（计数翻了、时刻停在 `-1`），这一站每一位都要读两位，所以统一写成 `记延时接手` 这样的函数。判据在 `test/e2e/async-component-timing.spec.ts`，五条。

## 五条读数，原样打印

```text
[定时机·after] {"设定毫秒":1200,"中途计数":0,"中途时刻":" after 接手时刻：-1","中途点了":true,"中途数字":" 点了 0 次 ","位置":{"有这一位":true,"距视口顶":753,"视口高":500,"在视口内":false},"等待毫秒":395,"接手计数":1,"接手时刻":" after 接手时刻：1382"}
[定时机·when] {"计数":0,"点了":true,"开开关":true,"之前":" 点了 0 次 ","开后可计数":1,"时刻":" when 接手时刻：1799","之后":" 点了 1 次 ","点前位置":{"距视口顶":792,"视口高":500,"在视口内":false},"开后位置":{"距视口顶":792,"视口高":500,"在视口内":false}}
[定时机·never] {"计数":0,"时刻":" never 接手时刻：-1","点了":true,"之前":" 点了 0 次 ","面板文本":" 定时机面板 LAB_HYDRATE_804，记号是 初始记号","换记号":true,"改后计数":1,"改后时刻":" never 接手时刻：2780","改后文本":" 定时机面板 LAB_HYDRATE_804，记号是 换过的记号","之后":" 点了 1 次 ","位置":{"距视口顶":829,"在视口内":false},"父级那一行":" 现在传给它的记号：换过的记号"}
[定时机·下载] {"面板文件":["pm7JCbZI.js"],"命中位次":33,"第一趟请求条数":39,"计数":0}
[定时机·关脚本] {"after":{"有那一段":true,"字节":7106},"when":{"有那一段":true,"字节":7162},"never":{"有那一段":true,"字节":7215}}
```

`5 passed`；端到端全量连跑两趟各 `195 passed／57 skipped`。

## after：那 1200 毫秒真的等满了

页面就绪之后先等 600 毫秒、再等 200 毫秒，这一刻读到的是 `中途计数: 0`、`中途时刻: -1`（还没接手，所以那一位没被填过），按钮 `中途点了: true`——事件确实发出去了——但那一行仍是"点了 0 次"。之后计数翻成 1，接手时刻 **1382 毫秒**，比设的 1200 毫秒晚 182 毫秒，那 182 是页面自己的启动开销。

判据钉的是两条：`中途计数` 必须是 0（早于 1200 毫秒不许接手），`接手时刻` 必须不早于 1200。位置那一行 `在视口内: false`（距视口顶 753，视口只有 500 高）顺手说明一件事：**`after` 不牵扯视口**，它到点就接手，跟 803 那一位"不滚进来就永远不动"是两种完全不同的依赖。

## when：跟着一个布尔走

这一档开局 `计数: 0`，点面板数字不动。用 DOM 的方式点一下"把开关打开"（`开开关: true`），计数立刻翻成 1，接手时刻 1799 毫秒；再点面板，"点了 1 次"。

`点前位置` 与 `开后位置` 都是 `距视口顶 792`、`在视口内: false`——这两个数是特意打出来的：官方示例里那个函数（`myFunction` 把 `isReady` 设真）如果靠点一颗看不见的按钮来触发，`locator.click()` 会先把那颗按钮滚进视口，于是"布尔起了作用"和"位置起了作用"就分不开了。这一趟面板从头到尾没进过视口，接手只跟着那个布尔。

## never：它确实不动，直到有人改了一次 props

这一档是这一站的重点，前后两半读数差得很远。

前半：等满 2500 毫秒（比 `after` 那一档还久），`计数: 0`、`时刻: -1`，点面板"点了 0 次"，面板里那一行文字还是服务端渲出来的那一份——`定时机面板 LAB_HYDRATE_804，记号是 初始记号`。「Never hydrates the component」这句在这一半成立。

后半：只点一颗"换一次记号"的按钮，它做的事情就是把传给面板的那个 prop 从 `初始记号` 换成 `换过的记号`，别的什么都没动——没有滚动，页面位置仍是 `距视口顶 829`、`在视口内: false`。400 毫秒之内读回来的是：`改后计数: 1`、`改后时刻: 2780`、面板文本变成 `记号是 换过的记号`、再点面板"点了 1 次"。

**那条 note 逐字兑现了**：改一次 props，`hydrate-never` 也当场接手。所以"never"这个名字在这套实现里说的不是"永远不"，而是"没人动它就不"。给 `hydrate-never` 的那一位传一个会变的 prop，效果与 `hydrate-when` 只差在"由谁决定变"。

还有一位更细的读数值得看：改 prop 那一刻，父级那一行 `现在传给它的记号` 先变成了"换过的记号"，而面板里那一行**当时还是"初始记号"**。父级是活的组件，子级还只是服务端留下的一段静态 HTML——**静态与活的分界线，就在这一站被读出来了**。接手之后子级那一份才被真的组件顶掉。

## 不接手也照样付下载

`[定时机·下载]` 是 `never` 档的：`计数: 0`（一次都没接手）、面板那个文件 `pm7JCbZI.js` 却在第一趟第 **33** 个请求就下来了（整趟 39 条）。

这已经是连续第三次读到同一件事：`Lazy` 前缀（802 的三种 `v-if` 形状）、`hydrate-on-visible`（803）、`hydrate-never`（这一站）——**没有一种能把下载推后**。这三种写法各自管的东西不同，但都不包括"什么时候取这份文件"，那一半归服务端 HTML 头部的 prefetch 名单管。

## 顺手改掉一条写死的判据

这一站给 `/docs` 加了三个实验位链接，结果把另一条不相干的老判据弄红了：`test/e2e/a11y.spec.ts` 那条"键盘走完全站"里，Tab 的次数写死成 14，而 `/docs` 的停靠点本来就是 14 个——多三颗链接之后，按满 14 下焦点还停在这条新链上（`Received: {"href": "/docs/getting-started?islandLab=never", ...}`），于是报"多半是焦点陷阱"。

这条红报得对，但它测的东西被那个常数绑架了：判据要说的是"把所有停靠点走完，焦点会离开文档"，可上限一旦写死，加一位就得改一次常数，迟早把"焦点出不去"与"我没按够"混成同一件事。改成从页面上数：

```ts
const 可停靠数 = await page.evaluate(() => document.querySelectorAll('a[href], button').length)
const stops = await walkTabs(page, 可停靠数 + 3)
```

同一趟读数 `{"可停靠数":14,"stopCount":14}`：走满 14 个停靠点，再按一下焦点就出文档了。**上限是判据的分母，不该由人记得**。

## 排错表

| 想确认的东西 | 这一站用的读法 | 读不到时先怀疑什么 |
| --- | --- | --- |
| 到点之前它是不是真没接手 | 在中途读一次计数＋点一次看数字不 | 中途那一刻是否已经跨过设定延迟 |
| 布尔策略是不是被滚动帮的忙 | 触发前后各读一次 `距视口顶` 与 `在视口内` | 触发按钮用了 `locator.click()` |
| never 是不是真的不动 | 等一段比 `after` 更久的时间再读 | 有别处正在改这一位的 props |
| 改 props 有没有让它接手 | 只改那一个 prop，别的都不动，读计数与时刻 | 判据自己滚了页面 |
| 静态那一份什么时候被顶掉 | 同一时刻分别读父级那一行与面板里那一行 | 以为改了 prop 界面就会跟着变 |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/async-component-timing.spec.ts
```

预期 `5 passed`。手动版最有意思的是 `never` 那一档：访问 `http://localhost:3100/docs/getting-started?islandLab=never`，等三五秒，反复点面板里的"点我计数"——数字一直不动；然后点一下"换一次记号"，那一行的记号变了，此时再去点"点我计数"，它开始加了。这就是官方那条 note 的手感：**它不是"永远不接手"，是"没人动它就没人接手"**。

## 本文依据

- 官方：讲组件目录的那一节（`Hydration Strategies` 里 `hydrate-after`、`hydrate-when`、`hydrate-never` 三段的原话与示例；开头那条关于 prop 变化的 note；`::warning` 那三条限制里"props 要写在模板里"那一条；`Listening to Hydration Events` 那句 `@hydrated`；五条注意里"每种策略都有它针对的场景"那一条）
- 仓库：`app/components/LabHydrateTiming.vue`（新）、`app/pages/docs/[slug].vue`（三档挂载、三个接手计数与三个处理器函数）、`app/pages/docs/index.vue`（`lab-after`、`lab-when`、`lab-never` 三条入口）、`test/e2e/async-component-timing.spec.ts`（新增 5 条）、`test/e2e/a11y.spec.ts`（Tab 上限从写死的 14 改成从页面上数）
- 上一批：[803 - 异步组件（3）：hydrate-on-visible 与 on-idle，什么时候开始接手](/posts/487701fc18420c702a92491d721bb9ad/)
- 下一篇：[805 - 异步组件（5）：三条限制与 @hydrated 事件，别让延迟水合骗你](/posts/eab263b05f6b72483a0e29c2b04bd1c0/)
