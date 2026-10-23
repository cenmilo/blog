---
url: /posts/c24ab414eeafab5f696b479c4a8b3d4e/
title: 798 - 仪表盘切片（4）：客户端导航会多一次岛屿请求，lazy 与 fallback 才不挡
date: 2026-10-29T12:00:00+08:00
updated: 2026-10-29T12:00:00+08:00
author: cmdragon

summary:
  摘要：切成服务端组件之后，站内点进来那一次导航会被岛屿的网络往返挡住，除非给 `lazy` 并配一段回退内容。这一站造了一个"会慢"的岛屿，把这段时间拉到肉眼可数的量级，量出三档读数：非 lazy 时整页被拖到 652 毫秒、`lazy` 时页面 41 毫秒就到而岛屿 641 毫秒才补上、岛屿失败时回退内容一直留在原位并且 `error` 事件真的发出来。这三件东西只有显式用 `<NuxtIsland>` 才拿得到。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 服务端组件
  - 岛屿
  - lazy
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 同一件事，两种屏幕表现

从文档列表页点进详情页，那一块服务端组件的内容比标题晚了多少才出现？本站三档读数（同一个页面、同一次点击，只是目的地那一块挂法不同）：

| 挂法 | 详情页标题出现 | 回退内容出现 | 岛屿内容出现 |
| --- | --- | --- | --- |
| 不带实验位的普通导航（对照） | 65 毫秒 | 没有这一位 | 没有这一位 |
| 非 lazy（默认） | 652 毫秒 | 没有 | 652 毫秒 |
| `lazy` ＋ `#fallback` | 41 毫秒 | 41 毫秒 | 641 毫秒 |

第二行就是官方讲服务端组件那一节写的「islands block on a network round trip during navigation」——标题和岛屿内容**在同一轮**出现，用户点了之后整页都没换。第三行是同一句的后半段：「unless you pass the `lazy` prop (with a `#fallback` slot) to render them non-blockingly」。

## 为什么要先造一个"会慢"的岛屿

本机接口几毫秒就回，挡与不挡看不出差别。所以这一站新增 `app/components/islands/IslandGate.vue`，它按 props 要求决定慢多久、要不要坏：

```ts
const props = defineProps<{ delay?: number, boom?: boolean }>()

if (props.delay && props.delay > 0) {
  await new Promise(resolve => setTimeout(resolve, props.delay))
}

if (props.boom) {
  throw createError({ status: 500, statusText: 'Island gate boom', message: '实验位：这一档就是要让岛屿失败' })
}
```

`delay` 由调用方给，产品自己那两位（正文与探针）从不传。详情页只在查询串带 `islandLab` 时才挂这一段：

```vue
<NuxtIsland
  v-if="实验档"
  name="IslandGate"
  :props="闸门入参"
  :lazy="实验档 === 'lazy' || 实验档 === 'boom'"
  @error="记闸门报错"
>
  <template #fallback>
    <p data-testid="gate-fallback">先给这一段：岛屿还没到</p>
  </template>
</NuxtIsland>
```

入口是列表页那三条 `<NuxtLink>`（`app/pages/docs/index.vue`），必须走链接而不是直接开地址：要量的就是"客户端导航那一趟"，而整页加载时岛屿是内联渲的、根本没有那一次往返。

文件放在 `app/components/islands/` 而不是 `app/components/` 下面，是因为 `<NuxtIsland>` 那一节写得很明白：「By default, component islands are scanned from the `~/components/islands/` directory. So the `~/components/islands/MyIsland.vue` component could be rendered with `<NuxtIsland name="MyIsland" />`.」名字直接写 `IslandGate` 就能取到，端点地址里那串哈希是内部加的（读数里能看到 `IslandGate_c4t…`）。

## 非 lazy 那一档：整页一起等

```text
[导航·非lazy] {"时刻":{"标题":652,"回退":null,"闸门":652},"岛屿响应":[{"名":"DocBody_QcL6U4","status":200},{"名":"IslandFacts_Vs","status":200},{"名":"IslandGate_c4t","status":200}],"实验位那一位":"实验档 slow · error 事件 0 次","报错文本":"","闸门文本":"闸门内容（延迟 600 毫秒）"}
```

这一趟发了三个岛屿请求——这一页现在有三个服务端组件（正文、探针、实验位）。**有几个就多几趟**，官方那句「an app with many islands per page makes many requests per navigation」在这个读数上是逐字成立的。

判据钉的是两件事：标题与闸门内容出现时刻之差 ≤120 毫秒（同轮＝整页被挡），以及回退那一位**始终没出现**（非 lazy 不给你看到回退内容）。对照档把"65 毫秒"钉在旁边，否则"652 毫秒"这个数没法说明是挡还是本机本来就慢。

## lazy 那一档：页面先到，岛屿后补

```text
[导航·lazy] {"时刻":{"标题":41,"回退":41,"闸门":641},"岛屿响应":[{"名":"DocBody_QcL6U4","status":200},{"名":"IslandFacts_Vs","status":200},{"名":"IslandGate_c4t","status":200}],"实验位那一位":"实验档 lazy · error 事件 0 次","报错文本":"","闸门文本":"闸门内容（延迟 600 毫秒）"}
```

`lazy` 换到的东西：**导航不再等那一趟往返**（41 毫秒页面就换掉了），代价是要给一段回退内容填着那 600 毫秒。回退内容与页面同一轮出现（41 毫秒），岛屿内容 641 毫秒到位后替换掉它。

`#fallback` 那一段是给"还没到"准备的，这一位在整页加载时你看不到它——下面那一档会量到。

## 岛屿坏掉的时候：回退留在原位，error 事件真的发出来

```text
[导航·boom] {"时刻":{"标题":41,"回退":41,"闸门":null},"岛屿响应":[{"名":"DocBody_QcL6U4","status":200},{"名":"IslandFacts_Vs","status":200},{"名":"IslandGate_c2n","status":500}],"实验位那一位":"实验档 boom · error 事件 1 次 Island gate boom","报错文本":"Island gate boom","闸门文本":"(读不到)"}
```

三条读数的意思：闸门内容**永远没出来**（`null`）、回退内容一直留在原位、页面自己照常 41 毫秒换掉。`error` 事件计数从 0 变 1，携带的短语就是岛屿抛出来的那一句。

这正是限制清单那句的落地形状：「Server components rendered via the auto-generated wrapper do not expose load and error events; use `<NuxtIsland>` directly if you need its `error` event and `refresh()` method」。797 量过自动包装那一位的失败形状——整块静默消失、控制台一声不响。要拿到失败这件事本身，就得换成显式 `<NuxtIsland>`：`#fallback`、`error` 事件、`refresh()` 这三件都只在这条路上。

## 整页加载：没有往返，但那 600 毫秒没消失

```text
[导航·整页] {"status":200,"服务端毫秒":629,"浏览器里岛屿请求数":0,"闸门首屏":"闸门内容（延迟 600 毫秒）","闸门在HTML里":true,"回退在HTML里":false,"标题在HTML里":true}
```

浏览器里岛屿请求数是 0（与 796 档一同一口径），服务端那一趟花了 629 毫秒，回退内容**不在**那份 HTML 里。三句话：

- 首屏没有额外往返，岛屿是内联渲进那份 HTML 的；
- 慢的工作一点没少，只是从"导航之后"挪到了"响应之前"——用户面前那段白屏；
- 回退内容只在"还没到"（lazy 导航中）与"取失败"这两种时刻出现，不是常驻占位。

所以选 `lazy` 不是在"快与慢"之间选，是在**把等待放在哪一段**之间选：放在响应之前（用户看到白屏），还是放在导航之后（用户看到回退内容）。

## 两个形状坑，各挂过一次

读"可能永远不出现"的元素，`innerText()` 的默认超时是 30 秒。`boom` 那一档的闸门内容就是永远不出现，判据因此挂在 `Test timeout of 30000ms exceeded` 上、连断言都没跑到。现在这一位自带 1500 毫秒超时：

```ts
async function 文本 (页面: Page, 选择器: string): Promise<string> {
  try {
    return (await 页面.locator(选择器).innerText({ timeout: 1500 })).trim()
  } catch {
    return '(读不到)'
  }
}
```

另一条不在判据里而在扫描里：`readdirSync(dir, { recursive: true })` 在 Windows 上**把目录本身也返回**，所以那位"岛屿形状的文件名单"里会多出一行 `"islands"`。`console-baseline` 那条名单已经按读到的值更新成四位，并在注释里写清哪两位是文件、哪一行是目录。

## 排错表

| 撞上的现象 | 先查这里 | 为什么 |
| --- | --- | --- |
| 点了链接整页不动 | 目的地有没有非 lazy 的岛屿 | 默认会挡在往返上，直到落定才换页 |
| 岛屿内容晚了一大截 | `lazy` ＋ `#fallback` 有没有配 | `lazy` 把等待挪到导航之后 |
| 首屏白屏时间变长 | 岛屿内部那一次取数花多久 | 内联渲时那一段时间在响应之前 |
| 岛屿坏掉但没人知道 | 用的是自动包装还是显式 `<NuxtIsland>` | `error` 事件与 `refresh()` 只在显式那条路上 |
| 判据挂在超时而不是断言 | 读缺失元素有没有自带短超时 | 默认 30 秒会先吃掉整个测试 |
| 文件名单里多出一个目录名 | `readdirSync` 的 `recursive` | Windows 上目录本身也在返回值里 |

## 量不到的那一段

`<NuxtIsland>` 那一节还有一句与预取有关：「When a `<NuxtLink>` to a server-only page is prefetched, its island response is prefetched too」。本站没有任何 server-only page（整站路由都是普通页面），这一条本轮**没量**，不能拿去当已验证的说法用；要验它得先把某一位页面标成只在服务端渲染。

## 自己动手跑一遍

```bash
npm run build
E2E_PORT=3299 npm run test:e2e -- test/e2e/island-lazy-fallback.spec.ts
```

预期 `5 passed`，四组时刻读数与上面一致（`652`／`641` 这类数字会随机器抖，但对照档应该在三位数以内、带延迟的那两档一定在 `delay` 之上）。想自己在浏览器里看：先开 DevTools 的 Network，从 `/docs?` 那三条实验链接分别点一次——非 lazy 那一档会看到**页面换掉之前**那三个岛屿请求还在跑，`lazy` 那一档先换页、再看到闸门内容替换掉那句回退文案。

## 本文依据

- 官方：讲服务端组件的那一节（客户端导航那一节的原话：每个岛屿都要多一次取数、`lazy` 与 `#fallback` 才不挡、"an app with many islands per page makes many requests per navigation"、`<NuxtLink>` 预取 server-only page 时把岛屿一起预取；当前限制清单里"自动包装那位不暴露 load 与 error 事件"那条）；讲 `<NuxtIsland>` 的那一节（`components/islands/` 的扫描规则与 `name` 用法、`#fallback` slot 的两种出现时机、`error` 事件、`refresh()`）
- 仓库：`app/components/islands/IslandGate.vue`（新增实验岛屿）、`app/pages/docs/[slug].vue`（三档挂法与 `error` 计数）、`app/pages/docs/index.vue`（那三条实验链接）、`test/e2e/island-lazy-fallback.spec.ts`（新增 5 条）、`test/e2e/console-baseline.spec.ts`（那位名单更新为四位）
- 上一批：[797 - 仪表盘切片（3）：岛屿拿不到页面状态，只能用 props 递进去](/posts/3ec9b7ce1d854bc6cdc108e30c2f05b1/)
- 下一篇：[799 - 仪表盘切片（5）：只让这一块里的一小块可交互：nuxt-client 与两档配置](/posts/8b2ec791fe909e3834ddd98e107a3b6f/)
