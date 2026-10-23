---
url: /posts/22161290ea8aab5801b0872785a0ed1b/
title: 817 - 载荷收敛与序列化（5）：自定义 reducer 与 reviver，让一个类进得来
date: 2026-11-01T13:00:00+08:00
updated: 2026-11-01T13:00:00+08:00
author: cmdragon

summary:
  摘要：服务层那种带方法的值对象，直接放进负载会让整页变错误页。官方给的解法是注册一对：一个把类换成可序列化的替身，一个在浏览器里把它还原回来。这一站把三种情况都跑一遍——不注册、两侧都注册、只注册一侧——读回来的重点不只是能不能发出去，还有生产构建把类名改掉了这件事，以及判据为什么必须去调一次方法而不是看原型名。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 负载序列化
  - 插件
  - 性能优化
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 上一站那位 500，这一站让它变成 200

上一站把九种值往负载里塞，类实例那一位两条路都是 `{"状态":500,"哨兵在":false}`，服务端日志一句 `Cannot stringify arbitrary non-POJOs`。这一站不换值，还是同一个类实例，只是给它注册一对读写规则：**不注册**是 500，**注册齐**是 200 而且方法还能调，**只注册一半**是最难发现的那种——服务端照常发货，浏览器认不出这个标签。

## 官方给的那对写法

讲 `useNuxtApp` 那一节的 Custom Reducer/Reviver 段（标着 v3.4）给的是一个插件文件、两句注册代码，例子是做日期时间的库：

```ts [app/plugins/date-time-payload.ts]
export default definePayloadPlugin((nuxtApp) => {
  definePayloadReducer('DateTime', (value) => {
    return value instanceof DateTime && value.toJSON()
  })
  definePayloadReviver('DateTime', (value) => {
    return DateTime.fromISO(value)
  })
})
```

那段上面有一句要紧的注释，原文是：「This kind of plugin runs very early in the Nuxt lifecycle, before we revive the payload. **You will not have access to the router or other Nuxt-injected properties.**」——这类插件跑在生命周期的很早一段，比装载荷还原还早，所以里面碰不到路由，也碰不到其它注入进来的东西。想在一个"实验档"里按需注册，是这条约束逼出来的写法。

本站照这个形状写，类放在 `app/utils/period.ts`：

```ts
export class 周期 {
  起始: string
  月数: number

  constructor (起始: string, 月数: number) {
    this.起始 = 起始
    this.月数 = 月数
  }

  /** 页面上要读的就是这个方法：还原回来如果只是个普通对象，这一行就读不出数。 */
  每月额度 (): number {
    return Math.round(1000 / this.月数)
  }

  /** 替身形状：一个字符串（`起始#月数`）。别在替身里放对象——标识名只有一个。 */
  替身 (): string {
    return `${this.起始}#${this.月数}`
  }

  static 还原 (替身: string): 周期 {
    const 段 = 替身.split('#')
    return new 周期(段[0] ?? '', Number(段[1] ?? 0))
  }
}
```

```ts
// app/plugins/payload-period.ts
export default definePayloadPlugin(() => {
  definePayloadReducer('周期', (值) => 值 instanceof 周期 && 值.替身())
  definePayloadReviver('周期', (替身: string) => 周期.还原(替身))
})
```

两个写法细节值得单说。第一，reducer 的返回值是**"命中判定"**：不匹配时返回假值（那句 `值 instanceof 周期 && 值.替身()` 就是这个用法），框架才会去试下一个 reducer。第二，**标识名两头必须一模一样**，官方那句 "Note that the "DateTime" string is the type identifier and must be the same on both the reducer and the reviver" 说的就是这件事，最后一节会看到只注册一半的后果。

## 注册前后

| 情况 | `useState` 那一路 | 取数返回值那一路 | 浏览器里 |
| --- | --- | --- | --- |
| 类没注册（上一站的 `cls`） | 500、哨兵不在 | 500、哨兵不在 | 整页错误页 |
| 两侧都注册（这一站的 `tagged`） | 200、哨兵在 | 200、哨兵在 | 方法能调 |

```text
[reducer·负载] {"state":{"状态":200,"哨兵在":true,
  "那一位":"的状态位\",[\"周期\",32],\"2026-01#12\",[\"Set\"],[\"ShallowReactive\",35],{\"docs"},
 "data":{"状态":200,"哨兵在":true,
  "那一位":",\"登录\",[\"周期\",13],\"2026-01#12\",[\"Island\",15],{\"key\":16,\"params\":17,\""}}
```

负载里那一位是 `["周期",32],"2026-01#12"`：标签后面跟的是下标，替身字符串自己在同一条数组里占第 32 位（813 讲的那套一维数组加下标引用，自定义类型走的也是同一条路）。上一站的内置类型 `["Map",32,33]`、`["Date","2026-01-02T03:04:05.000Z"]` 长得一模一样——**自定义标签不是特殊通道，只是往同一张表里多加了一个名字**。

还有一点：`["周期", …]` 里那个标识名是**数据**，不是代码里的标识符。生产构建把类名改了，标签一个字没动。

## 还原回来的是不是它：别看原型名，调方法

```text
[reducer·控制台] {"页外错误":[],"方法那一行有几个":1,"这一段还在不在":1}
[reducer·浏览器] {"之前":"发出去之前：object|e|可枚举 2",
  "之后":"水合之后手上：object|e|可枚举 2",
  "方法":"方法还能调吗：每月额度 83"}
```

`方法还能调吗：每月额度 83` 这行是判据主体。至于原型名——注意它读出来是 **`e`**，不是 `周期`：两侧都是。生产构建会重命名标识符，类名跟着变成 `e`，`constructor.name` 因此**不能当判据**（上一站那几位显示的是 `Map`／`Date`／`RegExp`，因为那些是内置类型的名字，压缩不动）。所以还原是不是成功，唯一可靠的读法是**调一个只有那个类才有的方法**，把它返回值当判据。

`可枚举 2` 也对得上：`起始` 与 `月数` 两个字段。

## 只注册一半：页面卡住，而且控制台没话

把 `definePayloadReviver` 那一行注释掉，只剩 reducer，重新构建跑同一批判据：

```text
[reducer·负载] {"state":{"状态":200,"哨兵在":true,"那一位":"…[\"周期\",32],\"2026-01#12\"…"},
              "data":{"状态":200,"哨兵在":true,…}}
[reducer·控制台] {"页外错误":[],"方法那一行有几个":0}
    Error: page.waitForFunction: Test timeout of 30000ms exceeded.
```

三件事叠在一起看：

- 服务端那半**完全正常**——两条路都 200、标签与替身都在负载里。也就是说这一半注册不会给任何人报警。
- 浏览器那半拿不到还原函数，页面停住了：方法那一行个数是 0，判据等到超时。
- 而 `page.on('pageerror')` 收到的是空的——没有任何未捕获异常冒出来。

这一条是本站最阴的一种坏法：状态码 200、HTML 里内容都在、控制台干净，症状只有"某个只在水合后才出现的读数没出现"。判据里那句 `方法那一行有几个` 与超时，就是为了这种情况留的：别只断"页面还在"。

出处写在这里供按图索骥：`node_modules/devalue/src/parse.js:226` 那句是 `throw new Error(\`Unknown type ${type}\`)。**但本轮没读到它冒出来**——它被吞在哪一层（装载荷那一段是不是有内部捕获、还是这一位根本没被访问）没钉到底，记进本线未闭合。

## 一个新加的文件不会自动被认下来

这一站还撞了一件与负载无关但很实在的事：类放在 `app/utils/period.ts`，页面与插件里都直接写 `周期`，typecheck 报的是：

```text
app/pages/docs/[slug].vue(169,21): error TS2304: Cannot find name '周期'.
app/plugins/payload-period.ts(11,50): error TS2304: Cannot find name '周期'.
```

看 `.nuxt/imports.d.ts` 那份自动导入清单，`utils` 那几行只有 `shared/utils/` 里的三个函数，新加的 `app/utils/period.ts` 不在里面。⇒ 别假设"文件放进目录就自动能用"，改完 `nuxt prepare` 之后要么在清单里查到这个名字，要么像这一站一样在两个使用处各写一行显式 `import { 周期 } from '~/utils/period'`。

## 什么时候该注册

三条口径，按代价从低到高：

- **能拍扁就先拍扁**。`周期.替身()` 那种"服务端把值摊成字符串或数字"的做法（815 的 `transform` 就是它的通用形态）比注册一对省事：不需要两侧同步，也不怕某个环境里少注册了一半。
- 要带行为（还原回来的对象要能调方法）才注册一对，而且把这一对放在同一个插件文件里，让"只写一半"在代码上就不成立。
- **函数永远不要用这条路**。上一站读到的 `Cannot stringify a function` 不是"格式没配对"，而是这个东西不该跨进程；要留在浏览器的行为就挂在组件作用域里。

## 判据

`test/e2e/payload-shape.spec.ts` 加到 **10 条**，本步新增基线⑩（两条路的 HTTP 账 ＋ 浏览器里的三个读数）：

```bash
E2E_PORT=3453 npm run test:e2e -- test/e2e/payload-shape.spec.ts
#   10 passed (21.1s)
```

```ts
expect(账[走]?.状态, `${走}-tagged：注册之后这一位该发得出去`).toBe(200)
expect(账[走]?.那一位, `${走}-tagged：负载里该出现 ["周期", 下标]`).toContain('["周期",')
expect(浏览器.方法, '还原成功的判据是方法真能调，不是看着像').toBe('方法还能调吗：每月额度 83')
expect(去前缀(浏览器.之后), '类名被压缩改掉了：这里不该再出现 `周期` 两个字').not.toContain('周期')
expect(页外错误.length, '两侧都注册时，浏览器在装载荷那一步不该抛错').toBe(0)
```

最后那条 `not.toContain('周期')` 看着别扭，它钉的是"类名不可读"这件事实：哪一天构建不再改名（比如换了压缩配置或加了保留名字的配置），这条会红，红了你就能确认判据的来源是名字而不是行为。

## 对不上表

| 看到的现象 | 大概率是 | 怎么确认 |
| --- | --- | --- |
| 页面变错误页、日志 `Cannot stringify arbitrary non-POJOs` | 类实例没注册 reducer | 只补 reducer，看状态码是否变 200 |
| 状态码 200、HTML 都在，但某个只该在水合后出现的读数没有 | 注册了一半（缺 reviver） | 数那一行的个数；`pageerror` 会是空的 |
| 原型名读出来是一个字母 | 生产构建改了标识符名 | 别用 `constructor.name` 当判据，调方法 |
| `TS2304: Cannot find name 'X'`，可文件明明在 `app/utils/` 下 | 自动导入清单里没有它 | 打开 `.nuxt/imports.d.ts` 查；没有就显式 import |
| 负载里那位变成 `["标签", 数字]`，你以为是 bug | 正常的下标引用，替身自己在数组里占位 | 按下标去那一位看内容 |

下一站 818 是这条线收官：六步各留一句、判据按用例数重跑一张分档表、把 `serialize` 与 `middleware` 这两个本机版本还没有的选项记进版本账，再把一路攒下来的未闭合逐条列出来。

## 本文依据

- 讲 `useNuxtApp` 那一节的 Custom Reducer/Reviver 段（`definePayloadPlugin`／`definePayloadReducer`／`definePayloadReviver` 与插件时机那句）：https://nuxt.com/docs/4.x/api/composables/use-nuxt-app
- 讲 `useState` 那一节（会被 JSON 序列化那句 important，上一站）：https://nuxt.com/docs/4.x/api/composables/use-state
- 标签写法、命中判定的返回值、`Unknown type` 那句：官方文档未逐条记载，本节引的是本机 `node_modules/devalue/src/stringify.js` 的 `:83`、`:122-125`（自定义 reducer 那段，`stringified[index] = \`["${key}",${flatten(value)}]\``）与 `node_modules/devalue/src/parse.js` 的 `:80`、`:226`
