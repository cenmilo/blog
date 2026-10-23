---
url: /posts/915737a3a8c388133bf56e7c3ebbfe1d/
title: 731 - 工程基线（6）：环境变量进 runtimeConfig：用 zod 把 .env 变成有类型的东西
date: 2026-10-18T14:00:00+08:00
updated: 2026-10-18T14:00:00+08:00
author: cmdragon

summary:
  摘要：环境变量要能在部署前就测出问题，而不是等页面白屏。这一站先把框架那条链量清楚：键必须先在 `nuxt.config` 声明、只有 `NUXT_` 开头的全大写变量能覆盖、客户端只拿得到 `public` 与 `app`、值还会被按 JavaScript 类型解析（`4848e0` 会变成数字）。再给项目补一个能单独跑的校验入口，把 zod 那份 schema 真正接上。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - runtimeConfig
  - 环境变量
  - zod
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 三行"已设"，一行结论

先在终端里跑一条命令，看它的输出长什么样：

```text
$ npm run check:env
已设  DATABASE_URL
已设  NUXT_PUBLIC_SITE_URL
已设  NUXT_SESSION_SECRET
环境变量合法
退出码 0
```

把站点地址那一位改成不像 URL 的东西、密钥改成六个字符，同一条命令变成：

```text
已设  DATABASE_URL
已设  NUXT_PUBLIC_SITE_URL
已设  NUXT_SESSION_SECRET
环境变量不合法 → NUXT_PUBLIC_SITE_URL: Invalid URL; NUXT_SESSION_SECRET: Too small: expected string to have >=32 characters
退出码 1
```

上半部分三行一模一样还是"已设"，变的只有最后一行和退出码。这就是这一站要立住的那件事：**"有没有值"和"值能不能用"是两个判断**，只看前者会一路绿灯到线上。

## 框架那条链：谁声明、谁能盖、谁能读

要往应用里放配置，得先在 `nuxt.config.ts` 的 `runtimeConfig` 里把键声明出来，官方给的最小形状就是这样：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  runtimeConfig: {
    // 私有键：只在服务端可用
    apiSecret: '123',
    // public 里的键也会发给客户端
    public: {
      apiBase: '/api',
    },
  },
})
```

文档把规矩写成两条：**自定义的键必须在 `nuxt.config` 里定义过**（这一条保证任意环境变量不会被暴露给应用代码）；**只有特定名字的环境变量能覆盖它**——以 `NUXT_` 开头、全大写、用 `_` 分隔键与大小写变化的那一种。

这两条都能当场验。在最小工程里加两个变量跑一次静态生成：

```text
NUXT_PUBLIC_API_BASE=https://api.example.test  → 页面读到 https://api.example.test
NUXT_NOT_DECLARED=hello                       → 生成出来的 HTML 里 0 处命中
```

`NUXT_NOT_DECLARED` 是**没声明过的键**：值进不去，页面里搜不到——第一条要求的真实效果。`NUXT_PUBLIC_API_BASE` 这个名字则是按规则从 `public.apiBase` 换算来的：`apiBase` → `API_BASE`，再带上 `NUXT_PUBLIC_` 前缀。

谁能读到也分两侧，这一点很容易踩。把整个 config 的键名渲到页面上，服务端预渲染时读到的是这四组：

```text
Object.keys(useRuntimeConfig())      → apiSecret,app,nitro,public
客户端收到的那一份                    → window.__NUXT__.config={public:{apiBase:"…"},app:{baseURL:"/",buildId:"…",buildAssetsDir:"/_nuxt/",cdnURL:""}}
```

**客户端只拿到 `public` 和 `app` 两块**，私有键根本不在那个对象里；服务端反过来能读到全部，但它是只读的（官方那句理由是"避免上下文共享"）。模板里想直接用公共那一份，可以写 `$config.public`。

上面那两行读数本身是故意违规的产物：为了量清楚，这一趟把私有键也渲进了 HTML，于是 `apiSecret` 这个词确实出现在生成的文件里。官方那条安全提醒说的就是这个动作——**别把 runtimeConfig 的键渲染出来或塞进 `useState`**。正常写法是只在服务端读它。

还有一条反模式被文档用 warning 标出来：把默认值写成"另一个名字"的变量，例如 `apiSecret: process.env.OTHER_VARIABLE`。这种写法**只在构建期有效，运行期会坏**——因为运行期覆盖只认那条命名规则。要按环境换值，就让变量名和 `runtimeConfig` 的结构对上。

## `.env` 只在三条命令里被读

官方那句提醒很短：Nuxt 命令行在 dev、build、generate 时会读 `.env`，**但当你跑构建好的服务器时，`.env` 不会被读**。

前半句可以验。在最小工程根放一份 `.env`：

```ini [.env]
NUXT_API_SECRET=from-dotenv-file
NUXT_PUBLIC_API_BASE=https://from-dotenv.example
```

```text
npx nuxt generate    rc=0    页面读到 from-dotenv-file 与 https://from-dotenv.example
```

后半句（产物服务不读 `.env`）这一趟没验——那要把构建好的服务起起来，本系列不在这一篇做。它的实际意思是：部署时得自己把变量交进去，讲 `preview` 命令那节给的写法是 `NODE_ENV=production node --env-file .env .output/server/index.mjs`。

再说一条 Node 侧的行为，CI 里常用：`process.loadEnvFile('.env')` **不会覆盖进程里已经存在的同名变量**。同一份 `.env`（里面写 `PROBE_KEY=from-dotenv`）两种跑法都量了一遍：

```text
PROBE_KEY=from-shell node …   → from-shell
node …（shell 没传）          → from-dotenv
```

所以命令行传进去的值赢过文件里的旧值。`.env` 与 `.env.*` 都该在 `.gitignore` 里，仓库里只留一份 `.env.example`——上一篇那份名单最后三条就是为这个。

## 值会被按 JavaScript 类型解析：`4848e0` 变成 `4848`

这一条最容易吃亏。环境变量进 runtimeConfig 时会被 `destr` 解析成对应的 JS 类型，官方用 warning 举的就是这个例子。三条读数：

| 你设的值 | 页面里读到的 | `typeof` |
| --- | --- | --- |
| 不设（用 `nuxt.config` 里的默认值 `from-config`） | `from-config` | `string` |
| `NUXT_API_SECRET=4848e0` | `4848` | `number` |
| `NUXT_API_SECRET='"4848e0"'` | `4848e0` | `string` |

第一行是基线；第二行是事故现场——`4848e0` 被当成科学计数法，值变成数字 `4848`，长度和内容都变了，而且没有任何提示。第三行是官方给的解法：**值本身要带字面双引号**。在 `.env` 里写 `NUXT_API_SECRET='"4848e0"'`；在 shell、Dockerfile 或托管平台的面板里设时，要确认那对引号是值的一部分、没被 shell 吞掉。

密钥这一类还有一句相关的：`app` 前缀的键和 `appSecret` 是 Nuxt 自己留的位，不要拿来放自己的值。官方说 `NUXT_APP_SECRET` 可以**不用在 `nuxt.config` 声明就设置**，并且开发期如果没配、Nuxt 会生成一枚随机密钥存进 `.nuxt/app-secret`，而**构建时不会生成**——每个要部署的环境都得自己设。这一趟在预渲染读到的 `appSecret` 是 `undefined`（顶层键只有 `apiSecret,app,nitro,public`），产物运行期那一半没取证，所以上面这几句按文档讲，别当已验。生成密钥的官方命令是 `openssl rand -base64 32`，并要求每个环境用不同值、但在同一环境的每次部署和每个实例之间保持稳定——换了值，依赖它的会话与签名就全部失效。

## 装不进去的东西，和类型从哪来

`runtimeConfig` 在交给 Nitro 之前会被序列化一次。**函数、Set、Map 这类放不进去**——需要这些就得写到 Nuxt 或 Nitro 的插件、中间件里去。

类型上，Nuxt 会用 `unjs/untyped` 从你写的 `runtimeConfig` 自动生成一个 interface；也可以自己声明覆盖掉：

```ts [index.d.ts]
declare module 'nuxt/schema' {
  interface RuntimeConfig {
    apiSecret: string
  }
  interface PublicRuntimeConfig {
    apiBase: string
  }
}
// 增补一个类型时，一定要 import 或 export 点什么，否则这段不生效
export {}
```

`nuxt/schema` 是给项目使用者访问"本项目这份 schema"的方便入口；如果你是写模块的人，要增补的是 `@nuxt/schema`。

## 项目里怎么落地：一份 schema 加一个能被跑的入口

`Substrate` 里那份 schema 是这样（`shared/schemas/env.ts`，zod 版本 `^4.6.7`）：

```ts [shared/schemas/env.ts]
const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  NUXT_PUBLIC_SITE_URL: z.string().url(),
  NUXT_SESSION_SECRET: z.string().min(32),
})

export function loadEnv (source: NodeJS.ProcessEnv = process.env): Env {
  const parsed = envSchema.safeParse(source)
  if (!parsed.success) {
    const detail = parsed.error.issues.map(i => `${i.path.join('.')}: ${i.message}`).join('; ')
    throw new Error(`环境变量不合法 → ${detail}`)
  }
  return parsed.data
}
```

三个键、三条判据（两个必须是合法 URL、密钥至少 32 字符），失败时**把所有不合法的键拼成一行**报出来——这就开头那行"已设但仍不合法"的来源。它配了三条单测（`test/unit/env.test.ts`，全用合成值）：齐全时返回类型化对象、缺字段时报错并点名缺哪个、密钥短于 32 字符报错。跑出来的读数是 `test/unit/env.test.ts (3 tests) 4ms`，整套单元测试 `Test Files 25 passed (25)`、`Tests 184 passed (184)`。

这里有个真问题：这份 schema 写完、单测也绿了，但全仓库搜 `loadEnv`，除了它自己的定义和那三条单测之外一个调用都没有。也就是说，谁也不会去跑它，线上照旧拿着一个不合法的密钥启动。这一站补的就是这个入口——`scripts/check-env.mjs`：

```js [scripts/check-env.mjs（节选）]
try {
  process.loadEnvFile('.env')
} catch {
  console.log('（当前目录没有 .env，只按进程环境变量判）')
}

for (const name of REQUIRED) {
  console.log(`${process.env[name] ? '已设' : '未设'}  ${name}`)
}

try {
  loadEnv()
  console.log('环境变量合法')
} catch (error) {
  console.error(error instanceof Error ? error.message : String(error))
  process.exitCode = 1
}
```

脚本里**一个值都不打印**，只报键名与结论——这份输出会被贴进工单、CI 日志，值不能跟着走。跑法与结果就是开头那两段，另外还有一趟是在没有 `.env` 的目录下用三个合成值跑的，结论同样是合法、退出码 0。

为什么不把这层校验塞进应用启动路径：这个项目要能"什么都没配也起得来"（端到端测试就靠这条降级——没配库时服务端把请求按未登录处理，而不是炸在连接上）。把强校验放进启动路径，降级就没了，测试会一片红却不代表代码坏。所以分工是这样：

| 入口 | 判什么 | 什么时候跑 |
| --- | --- | --- |
| `npm run check:env` | schema 认识的三个键：缺哪一位、哪一位不合法 | 部署前、换环境时、新人配机器后 |
| `runtimeConfig` 声明 | 键在不在、`NUXT_` 变量能不能盖 | 每次构建与每次请求 |
| 服务端那条降级 | 没配库时按未登录处理，配了库连不上就照原样抛 | 运行期 |

## 这一步的自查清单

1. 每个要用的键都在 `nuxt.config.ts` 的 `runtimeConfig` 里声明过一遍；没声明的 `NUXT_*` 变量不会进应用。
2. 变量名按那条命名规则换算（`NUXT_PUBLIC_SITE_URL` ↔ `public.siteUrl`），不要把默认值写成 `process.env.另一个名字`。
3. 密钥、token 之类如果可能长成数字或 JSON 的样子，就把它写成带字面双引号的值。
4. 每个环境设自己的 `NUXT_APP_SECRET`（`openssl rand -base64 32`），别指望构建帮你生成。
5. `runtimeConfig` 里不放函数、Set、Map；需要就写进插件或中间件。
6. 部署前跑一次 `npm run check:env`，看退出码而不是只看"已设"那三行。

| 现象 | 先查什么 |
| --- | --- |
| 页面读到 `undefined` 或默认值 | 键有没有在 `nuxt.config` 声明；变量名大小写与前缀对不对 |
| 本地好的、线上坏 | 线上是不是在跑构建好的服务——那种起法不读 `.env` |
| 密钥像被截断、变成数字 | `destr` 把值解析成了别的类型，加内层双引号 |
| 客户端读到了私有键 | 有人把它渲进了模板或塞进 `useState`，那是泄露不是特性 |
| 校验全绿但启动就崩 | 校验入口没人调用；`grep` 一次你的函数名看引用在哪 |

下一步 [732《工程把关（1）：装上 lint 的 flat config，当场制造一次真实报错再修掉》](/posts/7a6a1d8324230dc204ddb61e476f271d/)：工程基线这条线到此收尾，下一条线开始把关——`npm run lint` 到底管哪些文件、按哪套规则，以及它和类型检查之间那条没人守的缝。

## 本文依据

- `runtimeConfig` 的声明形状、两条硬要求（必须先声明、只有 `NUXT_` 全大写变量能覆盖）、反模式 warning 与 `destr` 类型解析 warning：`https://nuxt.com/docs/4.x/guide/going-further/runtime-config`
- 客户端只读 `public` 与 `app`、服务端全部可用且只读、安全提醒、`$config.public`、序列化限制、插件与服务端路由里的用法：`https://nuxt.com/docs/4.x/guide/going-further/runtime-config`
- `appSecret` 与 `NUXT_APP_SECRET`（不用声明即可设）、`openssl rand -base64 32`、开发期生成到 `.nuxt/app-secret` 而构建不生成、每环境稳定：`https://nuxt.com/docs/4.x/guide/going-further/runtime-config#application-secret`
- 手动增补 `RuntimeConfig` / `PublicRuntimeConfig` 的写法与 `nuxt/schema`、`@nuxt/schema` 的分工：`https://nuxt.com/docs/4.x/guide/going-further/runtime-config#typing-runtime-config`
- `.env` 在 dev/build/generate 被读、跑构建好的服务器时不读，以及 `node --env-file .env` 那条起法：`https://nuxt.com/docs/4.x/guide/going-further/runtime-config`、`https://nuxt.com/docs/4.x/directory-structure/env`、`https://nuxt.com/docs/4.x/api/commands/preview`
- `.env` 与 `.env.*` 要进 `.gitignore`、只留 `.env.example`：`https://nuxt.com/docs/4.x/directory-structure/gitignore`
- 本篇读数来自最小工程（Nuxt 4.5.2、Node v24.11.1）与正式项目 `Substrate/` 的同一轮运行；`.env` 用的是合成值，真实密钥与连接串没有出现在任何输出里
