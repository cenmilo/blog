---
url: /posts/c11de876de91e2635e5834147d352a4b/
title: 792 - 两写法分工（4）：服务端那一趟带不带浏览器 cookie，两条口子不一样
date: 2026-10-28T12:00:00+08:00
updated: 2026-10-28T12:00:00+08:00
author: cmdragon

summary:
  摘要：账户页上加两行字，各用一种写法去问同一个接口，问的都是"服务端这一次收到会话号没有"。一份 HTML 里读出两个答案。差别不在选哪个组合式函数，而在服务端自己发出去的那一次内层请求有没有把浏览器的请求头带上——开关是地址的形状。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 取数与缓存
  - 请求头转发
  - 测试分层
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上先出现两行互相矛盾的话

反馈通常只有两句：「我明明登录了，账户页写着未登录」，或者「本地是好的，部署上去会员区的数字就是空的」。这两句在代码里指向不同的地方，凭话本身分不出来。

上一站量的是类型从哪来，这一站把另一件事摆到屏幕上：在 `app/pages/me.vue` 里加两行，用两种写法问同一个接口，答案直接印进 HTML。加完之后账户页上有这么两句话：

```text
走 useFetch 那一路：带上了（这一趟由服务端发）
走 useAsyncData 包住 $fetch 那一路：没带上（这一趟由服务端发）
```

同一次渲染、同一个服务端、同一个接口，两个答案。这一节就讲这个差别从哪来、开关到底在哪一位上。

## 服务端渲一次页面，其实发了两次请求

先说清一件事，不然后面没法读。**内层请求**指的是：浏览器来要这一页的 HTML，服务端在拼这页 HTML 的过程中，自己去调了本站的接口。那一次调用是服务端进程发出来的 HTTP 请求，不是浏览器发的。

浏览器发请求会自动带上 cookie——这是它自己的规矩，脚本挡不住也补不上。内层那一次既然是服务端发的，浏览器那条 cookie 就不在它身上。官方在讲 `useRequestFetch` 的那一节写的正是这个：

> 「When making a client-side fetch request, the browser automatically sends the necessary headers. However, when making a request during server-side rendering, due to security considerations, we need to forward the headers manually.」

同一节末尾的 `::tip` 给出了两条口子的分工：**`useFetch` 底层用的就是 `useRequestFetch`**，替你把手动转发那一步做了；而同一节的对照示例里，`useAsyncData` 包住裸 `$fetch` 那一位写的是「This will NOT forward anything」。

## 本站的读数：一份 HTML 里两个答案

加在页面上的就三行：

```ts
interface SessionProbe { hasSid: boolean }

const { data: forwardProbe } = await useFetch<SessionProbe>('/api/session-state', { key: 'me:probe-forward' })
const { data: bareProbe } = await useAsyncData('me:probe-bare', () =>
  $fetch<SessionProbe>('/api/session-state'))
```

问的接口 `/api/session-state` 是本站现成的，只回答"这一次请求里有没有 `sid` 那条 cookie"，回一个布尔值，凭据原文一个都不回页面。这一位不需要连数据库——本站有个演示接口专门发一条假会话号（`server/api/session-demo.post.ts`），而 `server/middleware/auth.ts` 在没配库时按无人登录处理、一次库都不查。

发布物上用真浏览器跑，三档读数（每一档都是工具原样打印的）：

```text
[cookie两口子·档一 匿名直接进] {"转发那条":"没带上（这一趟由服务端发）","裸调那条":"没带上（这一趟由服务端发）","字节数":2981}
[cookie两口子·档二 带会话号直接进] {"浏览器存的":["csrf","sid"],"转发那条":"带上了（这一趟由服务端发）","裸调那条":"没带上（这一趟由服务端发）"}
[cookie两口子·档三 点进来] {"页面发的取数":["/api/session-state","/api/session-state"],"转发那条":"带上了（这一趟由浏览器发）","裸调那条":"带上了（这一趟由浏览器发）","水合告警数":0}
```

三档读出来的是一条规矩的三个面：

- 档一两行都是「没带上」。这一档排除的是"这行字恒为真"：浏览器没有会话号时它必须说没带上。
- 档二就是这一站的全部：**只有服务端那一路带得上**。裸 `$fetch` 那一行不是写错了，它本来就不带。
- 档三换成浏览器发请求，两种写法一起变成「带上了」，而 `页面发的取数` 里真的出现了两次 `/api/session-state`。

也就是说：**同一个地址在两种情况下会有两种答案**，取决于这一次渲染走了哪条腿、用了哪种写法。

## 开关不在函数名上，在地址形状上

如果真是"选错函数就带不上"，这规矩倒好记。实际决定它的是地址长什么样，那个头也能自己补。先把 `useFetch` 那一位的地址换成绝对写法，读数立刻翻回去：

```text
[cookie两口子·档二 带会话号直接进] {"浏览器存的":["csrf","sid"],"转发那条":"没带上（这一趟由服务端发）","裸调那条":"没带上（这一趟由服务端发）"}
```

这一档判据红了：`Expected: "带上了（这一趟由服务端发）"`、`Received: "没带上（这一趟由服务端发）"`。原因在包内那一条判断上（`node_modules/nuxt/dist/app/composables/fetch.js:108`，编译后的形状）：

```js
if (typeof _request.value === "string" && _request.value[0] === "/" && (!toValue(fetchOptions.baseURL) || toValue(fetchOptions.baseURL)[0] === "/")) _$fetch = useRequestFetch();
```

三位条件：地址是字符串、地址以 `/` 开头、`baseURL` 没写或者也以 `/` 开头。三位全中才把内层请求器换成 `useRequestFetch`。所以：

- 本站相对路径 ⇒ 自动带；
- 本站接口写成绝对地址 ⇒ 不带；
- 第三方接口 ⇒ 永远不带。

反过来也一样：给裸 `$fetch` 自己补上那个头，它又带了。这是官方在讲 data fetching 那一节给过的写法（`useRequestHeaders`）：

```ts
const { data } = await useAsyncData('me:probe-bare', () =>
  $fetch<SessionProbe>('/api/session-state', { headers: useRequestHeaders(['cookie']) }))
```

同一档读数变成 `{"浏览器存的":["csrf","sid"],"转发那条":"带上了（这一趟由服务端发）","裸调那条":"带上了（这一趟由服务端发）"}`，那条判据同样红了一次。所以这一站的结论不是"useFetch 好、useAsyncData 坏"，而是：服务端那一趟要不要带请求头，是一句你自己得说清的话——`useFetch` 只是替你把默认值填成了"带"。

浏览器那一条腿上这两个东西各自是什么，也量了一次（`test/nuxt/request-leg-client.test.ts`）：

```text
[浏览器那条腿] {"头":{},"头键数":0,"是同一个脚":true}
```

`useRequestHeaders` 给空对象、`useRequestFetch()` 返回的就是全局那个 `$fetch`——官方那两句话（「In the browser, `useRequestHeaders` will return an empty object.」「In the browser during client-side navigation, `useRequestFetch` will behave just like regular `$fetch`.」）在运行时层成立。

## 转发的是哪些头，漏掉的是哪些

官方给了一份"不打算被转发的头"清单：`transfer-encoding`、`connection`、`keep-alive`、`upgrade`、`expect`、`host`、`accept`，句子是「Headers that are not meant to be forwarded will not be included in the request.」

包内那份集合多一位、也不解释（`node_modules/h3/dist/index.mjs:1138-1147`，这一条读的是包内源码，文档未列全）：`Set` 里是 `transfer-encoding`、`accept-encoding`、`connection`、`keep-alive`、`upgrade`、`expect`、`host`、`accept` 八项，而 `host` 在地址是相对路径时又被放回来（同文件 1268-1270 行那句 `host: typeof req === "string" && req.startsWith("/")`）。

这一站只验了 `cookie` 这一位，其余头没有各自的读数；想知道你的站到底转发了什么，看包内那个集合比看文档更全。

反方向也成立：既然相对路径会自动把入站请求头转出去，就别顺手把绝对地址写成相对的那种。讲 useFetch 那一节给过一份不要转发的头清单（`host`、`content-length`、`x-forwarded-*`、`cf-*` 那一串）——亲手补头时把整包请求头转给第三方是最省事的错法。

## 这条规矩会连带坏掉的两件事

**缓存**。这一页的内容现在跟着人变。777 那一站量出来：`swr` 写在页面路径上观察不到效果，只有写在 `/**` 这一种注册路由的键上才生效；一生效，第一个访问者那份 HTML 就发给后面所有人。所以 `nuxt.config.ts` 里 `/me` 那行是一个缓存位都不给：

```ts
'/me': { ssr: true, prerender: false },
```

这两档读数本轮没重量（记在前面的站里）。规矩落在这一站：取数跟着请求头变的页面不能进共享缓存。

**水合**。页面上那句「这一趟由服务端发」是服务端写下来、跟着负载进浏览器的：

```ts
const probeLeg = useState<'服务端' | '浏览器'>('me:probe-leg', () => import.meta.server ? '服务端' : '浏览器')
```

第一版不是这么写的，是在模板里现问 `import.meta.server`。服务端渲好、浏览器接手时那一问变成"假"，同一个位置被改成另一种说法——控制台报水合不匹配。判据里那条 `水合告警数` 就是钉这件事的（本轮读数 `0`）：凡是"服务端才知道"的信息，都得由服务端写进负载。

## 判据层三个坑，都会伪装成结论

写这三条判据的过程里，工具给了三次假信号。

`page.request` 与测试夹具里的 `request` 不是一回事。后者是另一个请求上下文，不带浏览器上下文里那条 cookie。用 `request.get('/me')` 读这一页，两行都会是「没带上」，看着正好印证"另一种写法不带"——其实两行一次都没被带过。换成 `page.request` 才出现档二那种一行带、一行不带。

静态扫描里"地址前 200 字符"这种窗口会串味。有一条判据要从地址往回找它被哪个取数入口包着，账户页那两条语句挨着写，前一条的 `useFetch` 落进了后一条的窗口，第二条读数变成三个名字。改成按顶格的行切边界，才各剩一个入口。

`0` 命中与"这里没有"长得一模一样。那份清单判据第一版把入口写成 `/\buseFetch\(/`，而账户页那位写的是 `useFetch<SessionProbe>(…`，泛型实参卡在名字与括号之间，扫出来是空列表——判据照样绿。暴露它的是里面那条"清单必须不为空"的正控制。

## 排错表

| 症状 | 先查哪一位 | 为什么是它 |
| --- | --- | --- |
| 页面写着未登录，接口却回 200 | 服务端那一趟带没带 cookie | 内层请求默认只由 `useFetch` 那一路转发 |
| 本地对、线上错 | 是不是浏览器那条腿才带 | 预渲染或关掉服务端渲，这一趟就不带 |
| 同一个页面点进去对、直接进错 | 两条腿各自的读数 | 两条腿的取数来源不同，别只看一条 |
| 数字偶尔是别人的 | `/**` 上有没有开 `swr` | 带请求头的页面被缓存会发给所有人 |
| 模板里那句在控制台报水合不匹配 | 那一位是不是服务端写进负载的 | 浏览器接手时 `import.meta.server` 是假 |

## 自己动手验一遍

在本站的仓库里，这三步都能跑出上面那些读数：

```bash
E2E_PORT=3260 npm run test:e2e -- test/e2e/cookie-forward-door.spec.ts
npm run test:nuxt -- test/nuxt/request-leg-client.test.ts
```

预期值：三档全过；档二那一行带、一行不带；最后一行打出 `"是同一个脚":true`。端到端那一趟自己先构建、再起发布物服务（`playwright.config.ts` 的 `webServer`）。

这一站立的规矩一句话讲完：服务端自己发出去的那一次内层请求，默认谁都不带——`useFetch` 加本站相对路径是替你把它带上了，写绝对地址就又不带，裸 `$fetch` 可以自己补。选哪种写法之前先回答"这一趟该不该带着浏览器那一份身份"。

793 讲 `lazy`、`pick`、`transform`、`default` 这些选项在两条口子上各叫什么、是不是等价；`useFetch` 是编译器保留的名字，自定义同名会怎样，那一站量。

## 本文依据

- 官方：讲 `useRequestFetch` 的那一节（服务端渲染时要手工转发请求头、那份不该转发的头清单、`useFetch` 底层用它、浏览器那条腿上它就是普通 `$fetch`）；讲 `useRequestHeaders` 的那一节（浏览器里返回空对象）；讲 data fetching 那一节的 "Pass Client Headers to the API" 小节；讲 `useFetch` 的那一节（那段不要把整包头转给外部接口的警告）
- 包内：`node_modules/nuxt/dist/app/composables/fetch.js:108`（那三位条件）、`ssr.js:37-40`（两条腿各返回什么）、`node_modules/h3/dist/index.mjs:1138-1147` 与 `1263-1274`（那份头集合，以及相对地址时把 `host` 放回来的那一句）
- 仓库：`app/pages/me.vue`（对照台那两行与那句"哪条腿"）、`test/e2e/cookie-forward-door.spec.ts`（新增，3 条）、`test/nuxt/request-leg-client.test.ts`（新增，1 条）、`test/unit/data-fetch-surface.test.ts`（加了一条认人地址的形状清单）、`server/api/session-state.get.ts` 与 `server/api/session-demo.post.ts`（本站现成的那个探针与发演示会话号的接口）
- 上一站（类型从哪来、地址写错只塌成 `unknown`）：[791 - 两写法分工（3）：类型从哪来，本站接口认地址、第三方接口要自己写](/posts/49efdf2b2e881d15daef241b8cc90152/)
- 下一篇：[793 - 两写法分工（5）：lazy、pick、transform 这些选项在两条路上的名字](/posts/02b5c6d7d18d09efba3de6f838725a82/)
