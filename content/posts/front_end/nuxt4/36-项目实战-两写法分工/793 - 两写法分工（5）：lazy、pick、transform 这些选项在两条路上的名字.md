---
url: /posts/02b5c6d7d18d09efba3de6f838725a82/
title: 793 - 两写法分工（5）：lazy、pick、transform 这些选项在两条路上的名字
date: 2026-10-28T13:00:00+08:00
updated: 2026-10-28T13:00:00+08:00
author: cmdragon

summary:
  摘要：同一批选项在两条取数口子上叫什么、写在第几个位置，官方四节各说了一遍。这一站把 lazy、pick、transform、default 四样各跑一趟，读数是当场读出来的：名字档与选项档在这一刻不等价，而 pick 少的是负载不是请求。另外那条"这两个别一起用"只写在声明文件的注释里。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 取数与缓存
  - 选项等价性
  - 测试分层
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 换个名字，界面就慢了一拍

账户页上写着订阅者是谁。有人为了让首屏早点出来，把 `useFetch` 改成 `useLazyFetch`——官方那两行 note 明写这两个是同一件事：

> 「`useLazyFetch` has the same signature as `useFetch`.」「`useLazyFetch` is equivalent to `useFetch` with `lazy: true` option set.」

改完数字仍然在，只是先空一下再出现。写成 `{ lazy: true }` 就没有这个空档。这一站量的就是这一格差别，顺便把 `pick`、`transform`、`default` 在两条口子上的名字各跑一遍。

先说位置，这是两条口子最直观的差别：

```ts
// 路子一：地址在前，选项对象是第二位，键写在选项对象里
await useFetch(url, { key: 'me:probe-picked', pick: ['hasSid'] })

// 路子二：键是第一位，函数第二位，选项对象是第三位
await useAsyncData('me:probe-bare', () => $fetch(url), { lazy: true })
```

## 名字档与选项档，这一刻不等价

在运行时层（把组件挂进 Nuxt 运行时，跑的是浏览器那一条腿）拿四只会记数、能延迟落定的假脚各跑一次，读数原样：

```text
[lazy的名字] {"名字档":{"那一刻":"(没值)","status":"idle","趟数":0},"选项档":{"那一刻":"byKind,metrics,note","status":"success","趟数":1},"另一条口的名字档":{"那一刻":"byKind,metrics,note","status":"success","趟数":1},"名字档关掉watch":{"那一刻":"byKind,metrics,note","status":"success","趟数":1},"落定之后":{"名字档":"(没值)","选项档":"byKind,metrics,note","另一条口的名字档":"byKind,metrics,note"},"趟数":{"名字档":0,"选项档":1,"另一条口的名字档":1}}
```

四件事按顺序读：

- **名字档**（`useLazyFetch`）那一刻 `idle`、假脚 0 趟；**选项档**（`useFetch` ＋ `{ lazy: true }`）当场 1 趟、状态已是 `success`。
- 另一条口子上的 `useLazyAsyncData` 当场就发，**没有这个分别**。
- 名字档再多等 200 毫秒仍是空的。它不是"慢一点就发"，那一趟要等**挂载**那一步（下一段就是这个读数）。
- 给名字档补一个 `watch: false`，立刻与选项档同形。

最后那一条是关键。包内 `fetch.js:88` 那一行把**选项对象自己也加进了 watch 源**：

```js
watch: watchSources === false ? [] : [...watchSources || [], _fetchOptions]
```

所以"名字档不发"不是 `lazy` 的语义，是这一处多出来的那一位改变了第一次执行的时机。这一层的根因本轮没追到底，能确定的是形状：**关掉 watch 就等价，开着就不等价**。

挂载之后再现读一次同一位（`test/nuxt/fetch-option-names.test.ts` 第二段判据）：

```text
[lazy挂载之后] {"setup那一刻":{"status":"idle","data":"(没值)"},"挂载之后":{"status":"success","data":"byKind,metrics,note"},"趟数":1}
```

界面表现就是那个"先空一下"。这一位别写成"那一刻读到什么就是什么"——把 `status` 抄成字符串存下来的第一版，抄下去的那一刻会一直 `idle`，要现读那两位才看得见它变成 `success`。

## default 两条口子都能顶住空值

`default` 是工厂函数（每次要起步值时调一次），不是给一个值。两边都写在选项对象里，只是那一位的**位置**不同：

```text
[default的位置] {"一条口那一刻":"byKind,metrics,note","二条口那一刻":"byKind,metrics,note","键位":{"一条口":"选项对象里","二条口":"第一位"}}
```

这一条是四条里唯一在读数上完全对称的：两份起步值都被读到，形状一模一样。

## pick 与 transform：顺序、互斥那句话在哪、数组上会怎样

先给四个读数：

```text
[pick与transform] {"只选":"metrics,note","只改":"备注,总数","两个都":"总数","数组上":{"形状":"title","是不是数组":false,"序列化":"{}"},"一条口的选":"metrics"}
```

```ts
const 只选 = await useAsyncData('opt:pick-only', async () => 原文, { pick: ['metrics', 'note'] })
const 两个都 = await useAsyncData('opt:both', async () => 原文, {
  transform: (输入: 计量) => ({ 总数: 输入.metrics.total, 备注: 输入.note }),
  pick: ['总数'],
})
```

- `pick` 与 `transform` **在两条口子上都写在选项对象里**，键名一字不差（`一条口的选` 那一位是走 `useFetch` 的）。
- 同给时不是"其中一个被忽略"。包内 `asyncData.js:355-356` 两行是顺序跑的：先 `transform`，再把它的返回值交给 `pick`。所以 `两个都` 读到的是 `总数`——**`pick` 挑的是改过之后的键名**。
- "这两个别一起用"那句不在参数表里，在声明文件的注释里：`asyncData.d.ts` 两处各写了一句「Do not use it along with the `pick` option.」。文档的参数表只说各自做什么。
- 结果不是对象而是数组时，`pick` 的实现（`newObj[key] = obj[key]`）交回一个**键留着、值是空的**新对象：`序列化` 那一位是 `{}`。界面上就是"这一位存在但没内容"。

## pick 少的是负载，不是请求

官方讲 data fetching 那一节的原话是：

> 「Both `pick` and `transform` don't prevent the unwanted data from being fetched initially. But they will prevent unwanted data from being added to the payload transferred from server to client.」

**负载**指的是服务端渲 HTML 时顺手塞进页面里的那份数据（浏览器接手时先从它拿值，不再重新取一遍）。这句话的两半得分两层看，用账户页那三位探针——同一个 `/api/session-state`，两位不带 `pick`，一位带：

```text
[pick的负载] {"负载里键名数":{"hasSid":3,"sidPrefix":2,"hasCsrf":2,"csrfLength":2},"界面读到":"hasSid","字节数":3160}
[pick的请求] {"接口回的键":["csrfLength","hasCsrf","hasSid","sidPrefix"],"界面读到":"hasSid"}
```

`pick的负载` 那一行数的是那份内联数据表里每个键名出现几次：三位探针都带 `hasSid` 所以是 3，而另外三个键名只在没写 `pick` 的两位里出现，所以是 2。**这条判据是可红的**：`pick` 一旦不生效，那三个键名就会都变成 3。

`pick的请求` 那一行是客户端导航时浏览器自己收到的接口原文：四位一个都没少。合起来正好是官方那两半——**少的那一份在页面里，不在请求里**。想让请求也少，得接口自己配合（只回你要的那几位）。

## 键名写错，编译器当场拦

`pick` 的键名不是字符串随便写。本站账户页那一位故意写错一次，`npm run typecheck` 打出来的三行是：

```text
app/pages/me.vue(63,101): error TS2769: No overload matches this call.
  The last overload gave the following error.
    Type '"nope"' is not assignable to type '"hasSid" | "sidPrefix" | "hasCsrf" | "csrfLength"'.
```

第二句里那串联合类型就是推出来的形状——这一位与 791 那一站讲的是同一件事：地址认得，类型就推得出来。数组那一位也是同样地被拦住（`Type 'string' is not assignable to type 'never'`），上一节的运行时读数是绕开这道检查之后量到的。

## 那六个名字是编译器保留的

官方在四节里各写了一句同样的警告（讲 `useFetch`、`useAsyncData`、`useLazyFetch`、`useLazyAsyncData` 的那四节），原话形状是「… is a reserved function name transformed by the compiler, so you should not name your own function …」。

这条警告**没有编译错误兜着**。包内 `useFetch`／`useLazyFetch` 这两个导出本身是占位（`compiler/runtime/index.js:8-16`，真被直接调用会抛 `NUXT_E1007`），实际生效的是编译器改写调用点后的那一份。你自己再导出一个同名的，构建照样过——只是那一处的调用从此不被改写，那些烤进改写里的行为（比如键的生成）就没了。783 量过这件事的形状：改写只发生在导出声明那一行。

所以这一条用一个静态判据守着，读数是：

```text
[保留名的导出] {"扫到":[],"扫过的文件数":15}
```

里面有一条正控制（同一条正则喂两句样例，必须扫出 `useFetch` 与 `useAsyncData`），否则"扫到 0 条"与"扫描坏了"是同一种形状。往 `app/composables/use-console-metrics.ts` 里临时加一句 `export function useFetch () { return null }`，这一条立刻红：`扫到:[{"文件":"app/composables/use-console-metrics.ts","名字":"useFetch"}]`。

## 排错表

| 症状 | 先查哪一位 | 为什么是它 |
| --- | --- | --- |
| 换成 `useLazyFetch` 后界面先空一下 | 那一刻是哪条腿、发没发 | 名字档要等挂载才发第一趟 |
| 加了 `pick` 但接口流量没变 | 看请求还是看负载 | `pick` 收的是负载那一份 |
| `pick` 之后界面读到"有这一位但没内容" | 结果是不是数组 | 实现是把键抄进新对象 |
| `pick` 之后拿不到 `transform` 改过的键 | 两者的顺序 | 先 `transform`，`pick` 从它的结果里选 |
| 键名写错就红 | 推出来的形状 | 键名按结果的键类型检查 |
| 同一份数两处读数不同 | 两条口子是不是共用一个键 | 同键才共享，`pick`／`transform` 必须一致 |
| 自己写了个 `useFetch` 能用但行为怪 | 那一处还是不是编译器改写的那一份 | 保留名，同名导出会让改写失效 |

## 自己动手验一遍

```bash
npm run test:nuxt -- test/nuxt/fetch-option-names.test.ts
E2E_PORT=3263 npm run test:e2e -- test/e2e/pick-payload-door.spec.ts
npm run test:unit -- test/unit/data-fetch-surface.test.ts
```

预期读数：第一条打出四行 `[lazy的名字]`／`[lazy挂载之后]`／`[default的位置]`／`[pick与transform]`；第二条两档全过，`负载里键名数` 是 3／2／2／2；第三条打出 `扫到:[]`。想复现那两位类型错误，把 `app/pages/me.vue` 里的 `pick: ['hasSid']` 改成 `pick: ['nope']` 跑一次 `npm run typecheck`，改回来即可。

794 收这条线：六步各留一句，加上本线判据的现跑分档表与还量不到的那几行。

## 本文依据

- 官方：讲 `useLazyFetch` 的那一节（同签名的 note、等价的 note、那两条 warning、返回值表七位）；讲 `useLazyAsyncData` 的那一节；讲 `useAsyncData` 的那一节（选项表 14 行、「`lazy: false` 底层用 `<Suspense>` 阻塞路由」那句、Shared State and Option Consistency 那两列清单）；讲 `useFetch` 的那一节（`transform` 与 `pick` 两行、保留名那句）；讲 data fetching 那一节的 Minimize payload size 小节（`pick` 与 `transform` 都不减少第一次取数）
- 包内：`node_modules/nuxt/dist/app/composables/fetch.js:56-66` 与 `:88`（参数怎么读、选项对象进 watch 源那一行）、`asyncData.js:355-356` 与 `:281-285`（`transform` 先跑、`pick` 的实现）、`asyncData.d.ts:31-34` 与 `:74`（互斥那两句注释）、`compiler/runtime/index.js:8-16`（占位与 `NUXT_E1007`）
- 仓库：`app/pages/me.vue`（第三位探针 `pick: ['hasSid']`）、`test/nuxt/fetch-option-names.test.ts`（新增，4 条）、`test/e2e/pick-payload-door.spec.ts`（新增，2 条）、`test/unit/data-fetch-surface.test.ts`（第 3、4 条：认人地址的形状清单与保留名清单）、`test/e2e/cookie-forward-door.spec.ts`（上一站的判据，本轮按三位探针改了取数次数那一条）
- 上一站（服务端那一趟带不带浏览器 cookie、开关是地址形状）：[792 - 两写法分工（4）：服务端那一趟带不带浏览器 cookie，两条口子不一样](/posts/c11de876de91e2635e5834147d352a4b/)
- 下一篇：[794 - 两写法分工（6）：这条线的读数收口，与还是量不到的那几行](/posts/66f5ba07295efbbb43e43c504a03c5c7/)
