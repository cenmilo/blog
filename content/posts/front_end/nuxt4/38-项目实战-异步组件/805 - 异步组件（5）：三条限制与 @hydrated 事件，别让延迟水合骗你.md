---
url: /posts/eab263b05f6b72483a0e29c2b04bd1c0/
title: 805 - 异步组件（5）：三条限制与 @hydrated 事件，别让延迟水合骗你
date: 2026-10-30T13:00:00+08:00
updated: 2026-10-30T13:00:00+08:00
author: cmdragon

summary:
  摘要：同一页面上并排挂四位同名组件，三种"白挂"的写法看起来都写了 `hydrate-on-visible`，读数也一模一样：接手次数停在 0。区别在于它们其实早就接手了——点一下数字就加；只有正对照那一位是真的没接手，点了不动，滚进视口才动。这一站把官方那条 `::warning` 的三条限制各撞一次，读出 `@hydrated` 不发与"还没接手"根本不是一回事，再用每 50 毫秒点一次的采样看事件和被接住的点击谁先出现。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 异步组件
  - 延迟水合
  - 编译插件
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 四位一样的组件，三行读数在骗人

访问 `http://localhost:3100/docs/getting-started?islandLab=limits`，页面下方并排出现四段"限制面板 LAB_LIMIT_805"，每一段上面都有一行"这块接手次数：N"。四位的挂法只差在名字和属性的写法：

```vue
<LazyLabLimitPanel mark="正对照" hydrate-on-visible @hydrated="记挂法接手('正对照')" />
<LazyLabLimitPanel v-bind="摊开的策略对象" @hydrated="记挂法接手('摊开')" />
<LabLimitPanel mark="无前缀" hydrate-on-visible @hydrated="记挂法接手('无前缀')" />
<!-- 第四位在 `app/components/LabLimitDirect.vue` 里：`import { LazyLabLimitPanel } from '#components'` -->
```

第一眼看过去，四行的数字都是 0——包括那三个已经能正常点的。**"接手次数：0" 在这三位身上不是"还没接手"，是"这个事件根本不发"**。这就是标题里那句"别让延迟水合骗你"。

## 官方那条 `::warning` 一次写了三件事

讲组件目录那一节，`Hydration Strategies` 开头紧跟着的提示块是这么说的：

> Currently Nuxt's built-in lazy hydration only works in single-file components (SFCs), and requires you to define the prop in the template (rather than spreading an object of props via `v-bind`). It also does not work with direct imports from `#components`.

再往下 `Listening to Hydration Events` 那一段：「All delayed hydration components emit a `@hydrated` event when they are hydrated.」——注意主语是 **delayed hydration components**，也就是真的被改写成延迟水合包装的那一种。这一站读到的正是它的反面：没被改写的那三位，事件一次都不发。

## 为什么是这三条：改写的是一位编译插件在做

Nuxt 这一层不是运行时判断，是构建时改写模板。三条限制在源码里一一对得上（`node_modules/nuxt/dist/index.mjs`）：

| 官方那条限制 | 源码里那一句 | 位置 |
| --- | --- | --- |
| 只在单文件组件里生效 | `transformInclude` 只收 `.vue`：`return isVue(id)` | `:4396` |
| 属性必须写在模板上 | 先按整份模板做一次字面量扫描 `TEMPLATE_WITH_LAZY_HYDRATION_RE`，再逐个元素读它自己的属性名 | `:4385`、`:4420` |
| `#components` 直连不生效 | 名字在本文件作用域里有声明就跳过：`if (scopeTracker.getDeclaration(node.name)) return` | `:4416` |

属性那一步是 `for (const attr in node.attributes)` 把属性名转成驼峰再去查一张策略表。`v-bind="对象"` 摊开时，**元素解析出来的属性里没有 `hydrate-*` 这个名字**，所以查不到策略；没策略就不改名，那位组件仍然是普通的 `Lazy` 异步组件——能用、只是不延迟。

顺带读到两个诊断码也在这同一段里：同一位挂两种策略报 `NUXT_B3005`（`:4423`），写了策略却没加 `Lazy` 前缀报 `NUXT_B3006`（`:4432`，只在开发/测试期发）。本站那一位"无前缀"的写法就是后者的形状。

## 六条读数，原样打印

```text
[限制·正对照] {"盒":{"有盒子":true,"距视口顶":679,"在视口内":false,"根上属性":"data-testid=limit-panel"},"计数":0,"点得着":true,"之前":" 点了 0 次 ","之后":" 点了 1 次 "}
[限制·摊开] {"盒":{"距视口顶":829,"在视口内":false,"根上属性":"data-testid=limit-panel hydrateonvisible=true"},"计数":0,"点得着":true,"点了之后":" 点了 1 次 "}
[限制·无前缀] {"盒":{"距视口顶":979,"在视口内":false,"根上属性":"data-testid=limit-panel hydrate-on-visible="},"计数":0,"点了之后":" 点了 1 次 "}
[限制·直连] {"盒":{"距视口顶":1129,"在视口内":false,"根上属性":"data-testid=limit-panel hydrate-on-visible="},"计数":0,"点了之后":" 点了 1 次 "}
[限制·出路] {"盒":{"距视口顶":716,"在视口内":false,"根上属性":"data-testid=limit-panel"},"点得着":true,"之前":" 点了 0 次 ","之后":" 点了 1 次 ","服务端HTML里有那一段":true,"HTML字节":7416}
[限制·时序] {"滚动前条数":0,"日志":[{"类型":"事件:正对照","时刻":581},{"类型":"点击1","时刻":641},{"类型":"点击2","时刻":692},{"类型":"点击3","时刻":742},{"类型":"点击4","时刻":797},{"类型":"点击5","时刻":848},{"类型":"点击6","时刻":899},{"类型":"点击7","时刻":954},{"类型":"点击8","时刻":1018},{"类型":"点击9","时刻":1079},{"类型":"点击10","时刻":1140},{"类型":"点击11","时刻":1203}],"事件是不是第一条":true}
```

`6 passed`；端到端全量连跑两趟各 `201 passed／57 skipped`。

## 三种白挂各自的形状，写在 DOM 上看得见

判据除了读计数，还把面板根元素身上的属性打了出来（`根上属性`）：

- **正对照**：`data-testid=limit-panel`，没有 `hydrate-*` 留在 DOM 上——那一位是包装组件，这个属性被当 prop 吃掉了，这才是"生效"的样子。
- **摊开**：`hydrateonvisible=true` 原样落在 DOM 上。`v-bind` 的对象键是驼峰，落到属性名上就是连写小写；它没进策略表，只是普通 attribute。
- **无前缀**与**直连**：`hydrate-on-visible=`（空值），同样是普通 attribute。这三位的事件计数永远是 0，而**点一下就加一次**——组件早就正常接手了。

所以这一站的判据没有一条只看计数。每条都成对读：`计数` ＋ `那一下点没点动`。只读前者，三种白挂会被读成"延迟生效了"。

## 事件和被接住的点击，谁先出现

`app/lazy/hydration-log.ts` 是一小块共用日志：页面的 `@hydrated` 处理器往里写一笔 `事件:正对照`，面板自己的点击处理器也往里写一笔 `点击N`。判据把正对照那一位滚进视口，然后每 50 毫秒点一次、共 12 次，再回来读整条日志。

读到的形状：日志第一条是 `事件:正对照`（581 毫秒），最早的一笔点击是 `点击1`（641 毫秒）——**12 次点击只留下 11 笔**，最靠前那一下没进日志，因为它落在接手之前。之后 11 次全部被接住。

结论分两层。机制层：包装组件里那句 `emit("hydrated")` 挂在 `onVnodeMounted` 上（`node_modules/nuxt/dist/components/runtime/lazy-hydrated-component.js:23`、`:26`），也就是子节点挂载之后才发——所以**"事件已发但还点不动"这种窗口在这一层不成立**，判据钉的就是"日志第一条必须是事件"。体验层别读反：事件不骗人，但它救不了你，**滚进视口后的第一下点击照样会丢**。要把那一下也接住，只能选不延迟水合的策略。

## 出路：不走模板属性，走 Vue 那一层

第一条限制说"只在单文件组件里生效"，那 `.ts` 里定义的组件对象怎么办？答案就是 Nuxt 那个包装自己做的事：`defineAsyncComponent({ hydrate, loader })`。`app/lazy/lab-object-panel.ts` 照同一个形状手写一次：

```ts
import { defineAsyncComponent, hydrateOnVisible } from 'vue'

export const LabObjectPanel = defineAsyncComponent({
  loader: () => import('~/components/LabLimitPanel.vue'),
  hydrate: hydrateOnVisible(),
})
```

页面里用 `<component :is="LabObjectPanel" mark="出路" />` 挂上，`[限制·出路]` 那一行读回来：开局在视口外（距视口顶 716）、点一下"点了 0 次"、滚进来之后点得动 ⇒ **这条路确实做到了同样的延迟**。另外单独读了一件容易漏的事：`服务端HTML里有那一段: true`（那一趟响应 7,416 字节）——它绕开了 Nuxt 那层服务端的记账，但内容仍然由服务端渲出来，看内容的一方不需要为此做任何额外处理。

## 排错表

| 想确认的东西 | 这一站用的读法 | 只读一个数会怎么错 |
| --- | --- | --- |
| 延迟水合到底生没生效 | 计数 ＋ 那一下点击有没有被接住 | 只看计数：白挂的三位也是 0 |
| 属性去了哪儿 | 打面板根元素上的属性名 | 不看会以为它被组件吃掉了 |
| 事件与可交互谁先 | 两边写同一条日志，按毫秒排序 | 只看截图会以为"事件先发=一定点得动" |
| 非 SFC 想延迟 | 用 Vue 的 `hydrate` 选项自己组一次 | 不试就以为只能放弃延迟 |
| 内容会不会因为延迟而少 | 不执行脚本读一次 HTML | 把延迟水合当 `server: false` |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/async-component-limits.spec.ts
```

预期 `6 passed`。手点版最有说服力：打开 `?islandLab=limits`，页面不动，从上往下依次点四段面板里的"点我计数"。第一段的数字不动，后三段全都立刻加——而它们上面那行"这块接手次数"全都写着 0。这时再滚到底，第一位才跟着动。想验证 `@hydrated` 与点击的顺序，就把 DevTools 的 Console 打开，手动把第一位滚进视口的同时连续快速点它几下：早于接手的那几下不会留下任何痕迹。

## 本文依据

- 官方：讲组件目录的那一节（`Hydration Strategies` 开头那条 `::warning` 三句、`Listening to Hydration Events` 那句 `@hydrated`、五条注意里"每种策略各有所长"那一条）
- 包内源码：`node_modules/nuxt/dist/index.mjs`（`:4385` 模板字面量扫描、`:4396` 只收 `.vue`、`:4416` 本地声明即跳过、`:4420` 逐元素读属性名、`:4423` 与 `:4432` 两个诊断码）、`node_modules/nuxt/dist/components/runtime/lazy-hydrated-component.js`（`:20-26`，`emit("hydrated")` 挂在 `onVnodeMounted`）
- 仓库：`app/components/LabLimitPanel.vue`（新，四种挂法共用的面板）、`app/components/LabLimitDirect.vue`（新，直连那一位）、`app/lazy/hydration-log.ts`（新，接手/点击同一条时间轴）、`app/lazy/lab-object-panel.ts`（新，Vue 那一层的出路）、`app/pages/docs/[slug].vue`（`limits`／`escape` 两档）、`app/pages/docs/index.vue`（`lab-limits`、`lab-escape`）、`test/e2e/async-component-limits.spec.ts`（新增 6 条）
- 上一批：[804 - 异步组件（4）：hydrate-after、hydrate-when 与 hydrate-never](/posts/9f594ffd2053b8479f2f61f36ff76ca1/)
- 下一篇：[806 - 异步组件（6）：这条线的读数收口，与没闭合的那几行](/posts/021a930fcc2d8e1760acb3140e731140/)
