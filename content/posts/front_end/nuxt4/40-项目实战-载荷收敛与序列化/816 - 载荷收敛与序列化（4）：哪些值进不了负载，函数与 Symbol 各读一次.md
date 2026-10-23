---
url: /posts/594d5ca632dcedbf1872eeba239f5a57/
title: 816 - 载荷收敛与序列化（4）：哪些值进不了负载，函数与 Symbol 各读一次
date: 2026-11-01T12:00:00+08:00
updated: 2026-11-01T12:00:00+08:00
author: cmdragon

summary:
  摘要：官方讲 `useState` 的那一节只给了一句报错名，并举了类实例这个例子。真把九种值依次塞进负载，报错的原文有四种各不相同，函数与 Symbol 那两句文档里根本没出现。这一站把九种值乘两条路一次读全，看清哪几种整页发不出去、哪几种能进去且水合回来还是原样，以及那几种进去的值在负载里被摊成了什么形状。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 负载序列化
  - useState
  - 错误处理
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 文档只给了一个报错名，而这里住着四类值

打开讲 `useState` 那一节，最底下有个 Troubleshooting 小节，标题就是一种值：`Cannot stringify arbitrary non-POJOs`，正文两句——「This error occurs when you try to store a non-serializable payload with `useState`, such as class instances.」，然后指向 `definePayloadPlugin` 那个逃生口。上面还有一句 important：「Because the data inside `useState` will be serialized to JSON, it is important that it does not contain anything that cannot be serialized, such as classes, functions or symbols.」

它一口气点了三类值（类、函数、Symbol），却只给了**一个**报错名。真写起来是这样：往 `useState` 里放一个函数，页面不是"少显示一点"，而是整条响应变成错误页。这一站把九种值挨个塞进去，两条路各塞一遍，读回来的是一张三行的表和四句各不相同的原文。

## 实验台一次只试一位

新增一档 `payload`，地址长这样：`/docs/getting-started?islandLab=payload-state-fn`、`...?islandLab=payload-data-map`。**一次请求只试一位**——两位一起炸就分不出是谁炸的，也读不到"另一种值是不是照样进得去"。

```ts
const 载荷候选: Record<string, () => unknown> = {
  fn: () => function 载荷函数 (参数: number) { return 参数 },
  symbol: () => Symbol('载荷标记'),
  symbolkey: () => ({ [Symbol('载荷键')]: 1 }),
  cls: () => new 载荷类('实例'),
  map: () => new Map([['键', 1]]),
  set: () => new Set([1, 2]),
  date: () => new Date('2026-01-02T03:04:05.000Z'),
  bigint: () => BigInt('9007199254740993'),
  regexp: () => /载荷/g,
}
// `useState` 在 setup 里跑一次并留住这个 ref：放进 computed 里再调，取的就不是当前组件上下文了。
const 载荷状态位 = 载荷实验 ? useState<unknown>(`lab:${载荷实验.位}`, () => 载荷实验.走 === 'state' ? 载荷候选[载荷实验.位]?.() : '这一档不写状态') : null
const { data: 载荷返回值 } = await useAsyncData('docs:payload-lab', () =>
  Promise.resolve(载荷实验?.走 === 'data' ? 载荷候选[载荷实验.位]?.() : '这一档不返回值'))
```

模板里那段只把"形状"打成一行字，不抛错——要炸也是编码器在发载荷那一步炸，页面自己不该抢先：

```vue
<p data-testid="payload-keep">LAB_PAYLOAD_816：这一段还在，说明这一趟响应发得出去</p>
```

那个 `LAB_PAYLOAD_816` 是哨兵：响应体里它在不在，比看状态码更能说明"页面渲到了那一段"。

## 九种值乘两条路

判据把 18 次请求跑成一张账，读回来的只有两种结果：

| 放进去的值 | 写进 `useState` | 从取数返回值给出去 | 服务端日志里那句原文 |
| --- | --- | --- | --- |
| 函数 | 500 | 500 | `Cannot stringify a function` |
| Symbol | 500 | 500 | `Cannot stringify a Symbol primitive` |
| 带 Symbol 键的普通对象 | 500 | 500 | `Cannot stringify POJOs with symbolic keys` |
| 类实例 | 500 | 500 | `Cannot stringify arbitrary non-POJOs` |
| `Map`／`Set`／`Date`／`BigInt`／`RegExp` | 200 | 200 | 没有报错 |

四行失败里，**只有最后一行的报错名是文档写过的那句**。服务端日志里那一段按请求顺序打出来，正好是四句（每一句都成对出现，一句错误一句 `cause`）：

```text
[WebServer]  DevalueError: Cannot stringify a function
[WebServer]  DevalueError: Cannot stringify a Symbol primitive
[WebServer]  DevalueError: Cannot stringify POJOs with symbolic keys
[WebServer]  DevalueError: Cannot stringify arbitrary non-POJOs
```

函数、Symbol、带 Symbol 键的对象各报自己那一句，位置在编码器里（`node_modules/devalue/src/stringify.js` 的 `:131`、`:133`、`:365`、`:361`）。也就是说，按文档那一句去搜代码是搜不到函数那一类的——搜 `Cannot stringify arbitrary non-POJOs` 只会命中类实例。

两条路的结果一模一样：**"值是哪里来的"不影响能不能进负载**，`payload.state` 与 `payload.data` 用的是同一个编码器。

## 进去的那五种，在负载里长这样

把负载原文里那一位截出来（813 讲过这是一维数组加下标引用）：

```text
"$slab:map":31},null,"页面写的状态位",["Map",32,33],"键",1
"$slab:set":31},null,"页面写的状态位",["Set",32,33],1,2
"$slab:date":31},null,"页面写的状态位",["Date","2026-01-02T03:04:05.000Z"]
"$slab:bigint":31},null,"页面写的状态位",["BigInt","9007199254740993"]
"$slab:regexp":31},null,"页面写的状态位",["RegExp","载荷","g"]
```

三件事值得记住。第一，这些标签（`Map`／`Set`／`Date`／`BigInt`／`RegExp`）**不是 Nuxt 注册的**——813 读到的 Nuxt 那七个只管 `ref`、`reactive`、`NuxtError`、岛屿这些，类型位是编码器自带的。第二，`Date` 与 `BigInt` 进去以后都变成**字符串**：一个是 ISO 串，一个是十进制串；`RegExp` 拆成"源"和"修饰符"两位；`Map` 的键和值摊成同级的两个位置。第三，正因为摊平了，**大 `Map` 会真的按成员数吃字节**——这一路的账要连着 814、815 那两站一起看。

## 水合回来还是不是原来那个东西

浏览器里读那一位的形状（`typeof`、原型名、成员数），五种全部前后一致：

```text
[载荷·水合形状] {"map":{"之前":"发出去之前：object|Map|大小 1","之后":"水合之后手上：object|Map|大小 1"},
  "set":{"之前":"发出去之前：object|Set|大小 2","之后":"水合之后手上：object|Set|大小 2"},
  "date":{"之前":"发出去之前：object|Date|可枚举 0","之后":"水合之后手上：object|Date|可枚举 0"},
  "bigint":{"之前":"发出去之前：bigint|9007199254740993","之后":"水合之后手上：bigint|9007199254740993"},
  "regexp":{"之前":"发出去之前：object|RegExp|可枚举 0","之后":"水合之后手上：object|RegExp|可枚举 0"}}
```

`new Map()` 到浏览器手里还是 `Map`、`大小 1` 也对得上，不是退化成普通对象。所以"负载只能放 JSON 能放的东西"这句常见说法是错的：能放进去的比 JSON 多，只是函数、Symbol、类实例这三类真的不行。

## 让它变一下，才知道标签在做什么

把 `date` 那一位换成内容一模一样的字符串 `'2026-01-02T03:04:05.000Z'`，两处判据各红一次：

```text
Error: Date 进负载是 ISO 串，不是时间戳数字
Expected substring: "[\"Date\",\"2026-01-02T03:04:05.000Z\"]"
Received string:    "…\"$slab:date\":31},null,\"页面写的状态位\",\"2026-01-02T03:04:05.000Z\",[\"Set\"]…"
Error: Date 回来还是 Date 实例
Expected: "object|Date|可枚举 0"
Received: "string|String|可枚举 24"
```

负载里那一段的字面内容几乎一样，**差别全在那个类型位上**：没有 `["Date", …]`，浏览器手里就是个字符串，`new Date(值)` 那类调用会静默拿到一个不同的东西（`string|String|可枚举 24`——字符串的"可枚举键"是它的下标位）。改回 `new Date(...)` 之后两条判据重绿。

顺带把上一条线的口径补全：这一站 18 次请求里那 8 次失败，**全部是整条响应变错误页**（状态码 500、哨兵不在）。它发生在服务端发载荷那一步，而 811 量过——`NuxtErrorBoundary` 的捕获注册整段包在 `if (import.meta.client)` 里。所以别指望页面上一处边界能接住"某位值进不了负载"，它接的是渲染期的错，不是编码器发不出货。

## 判据与读数

`test/e2e/payload-shape.spec.ts` 加到 9 条，本步是基线⑧（矩阵）与基线⑨（水合形状）：

```bash
E2E_PORT=3442 npm run test:e2e -- test/e2e/payload-shape.spec.ts
#   9 passed (22.0s)
```

```ts
expect(取(`${走}-${位}`).状态, `${走}-${位}：这一位进不了负载，整页该失败`).toBe(500)
expect(取(`${走}-${位}`).哨兵在, `${走}-${位}：哨兵所在那一段压根没发出去`).toBe(false)
expect(取('state-regexp').那一位, 'RegExp 拆成源与修饰符两位').toContain('["RegExp","载荷","g"]')
expect(剥(一行.之后), `${位}：水合之后手上的形状该与发出去之前一致`).toBe(剥(一行.之前))
```

全量端到端 `E2E_PORT=3440` 跑回 **233 passed／57 skipped**，本线前面几站的判据（814 的 `pick`、815 的 `transform`、813 的三页字节账）都没被这一档碰坏——新加那一位在不带 `islandLab` 时只是一个短字符串，约二十来字节，落在 `payload.data` 里。

## 这三类值各自是从哪里进来的

矩阵读完了，更实用的一张表是"它们怎么进来的"——三类值各有各的来路，而且都不是有人故意往负载里塞函数。

函数这一类最常见的一条路是上一站那个选项：`transform` 之后的形状才是进负载的形状（815 钉过），所以谁在 `transform` 里顺手挂了一个回调、或者把一个带方法的对象当结果返回，报错就落在发载荷那一步，而代码看上去只是"改了个形状"。第二条路是给页面存行为：把 `关闭()` 这类回调存进 `useState`，服务端那一趟它跟着写进 `payload.state`，于是整页发不出去。

Symbol 这一类几乎只有一种来路：**拿它当映射表的键**（`const 内部键 = Symbol('内部')`，然后 `{ [内部键]: … }`）。它报的是第三句原文，与"值本身是 Symbol"那句不同名，看日志时容易认错对象。

类实例这一类最容易出现的地方是服务层的值对象——金额、时长、日期时间这类"带方法的对象"，下一站那个自定义写法用的例子正是做日期时间的库。ORM 返回的带原型的行对象也算这一类。

三条口径按代价从低到高排：**能拍扁就先拍扁**（`toPlain()`、`toISOString()`、`[...map.entries()]`，服务端做完，客户端只拿数据）；拍扁会丢掉含义的（比如你要的就是一个能 `plus()` 的对象），才用下一站的自定义 reducer 与 reviver；函数没有第二种解法——它不是"形状特殊"，是根本不该跨进程走，要留在浏览器里就该挂在组件的作用域里，而不是负载里。

## 对不上表

| 看到的现象 | 大概率是 | 怎么确认 |
| --- | --- | --- |
| 页面直接变错误页，模板里找不到抛错的地方 | 负载里塞进了函数、Symbol 或类实例 | 看服务端日志那句 `DevalueError`，四种值四句原文 |
| 搜文档那句 `Cannot stringify arbitrary non-POJOs` 搜不到自己的错 | 你放的不是类实例 | 按日志原文搜，或去编码器里看那几处分支 |
| `Map` 到浏览器手里变成普通对象 | 中间某层把它当普通对象转了一遍（`{...值}`、`JSON.stringify`） | 水合后读原型名，别只看成员数 |
| 日期到浏览器手里不能 `.getTime()` | 接口那边给的就是 ISO 字符串，不是 `Date` | 比负载里有没有那个类型位 |
| 加了 `Map`/`Set` 之后负载明显变大 | 成员被摊平成一个个位置 | 读负载字节数与成员数，按这一路的账收敛 |

下一站 817 补最后一块：**让一个类进得来**。官方给的是 `definePayloadPlugin` 加一对 `definePayloadReducer`／`definePayloadReviver`，那段例子用的库是 Luxon；这一站读的是"不注册"与"注册"两种情况下，浏览器手里还是不是那个类的实例。

## 本文依据

- 讲 `useState` 那一节（important 那句与 Troubleshooting 那段）：https://nuxt.com/docs/4.x/api/composables/use-state
- 讲 `useNuxtApp` 那一节（`payload` 三个键、Custom Reducer/Reviver 一节，下一站）：https://nuxt.com/docs/4.x/api/composables/use-nuxt-app
- 四句报错原文与类型标签：官方文档未逐条列出，本节引的是本机 `node_modules/devalue/src/stringify.js` 的 `:131`、`:133`、`:365`、`:361`
