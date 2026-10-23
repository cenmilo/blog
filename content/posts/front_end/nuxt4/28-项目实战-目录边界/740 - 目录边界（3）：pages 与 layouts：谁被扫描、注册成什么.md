---
url: /posts/c32e3025857237f0fc287b8d67de1040/
title: 740 - 目录边界（3）：pages 与 layouts：谁被扫描、注册成什么
date: 2026-10-20T11:00:00+08:00
updated: 2026-10-20T11:00:00+08:00
author: cmdragon

summary:
  摘要：`app/pages/` 里放一个文件，站点就多一个地址；`app/layouts/` 里放一个文件，多的是一个能用名字呼叫的壳。这两个目录都不靠配置文件注册，但它们被扫描的方式完全不同：页面的文件名变成路径，布局的文件名变成一张类型表里的键。这一站用十个探针文件把两边的规矩一次量清，顺带看四道类型护栏各自挡在哪。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 文件路由
  - 布局
  - definePageMeta
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 屏幕上看得见的事

这个仓库的 `app/pages/` 下六个文件、`app/layouts/` 下一个 `default.vue`，全项目没有一个路由配置文件——地址是文件名生成的：

```bash [app/pages 与 app/layouts 的真实清单]
app/pages/index.vue
app/pages/me.vue
app/pages/docs/index.vue
app/pages/docs/[slug].vue
app/pages/console/index.vue
app/layouts/default.vue
```

`app/pages/` 是可选目录：只用 `app.vue` 而没有 pages 目录时，`vue-router` 根本不会进包；要强制启用这套文件路由，在 `nuxt.config` 里写 `pages: true`，或者按自定义路由那一节放一份 `router.options.ts`。

`app/layouts/` 则相反：只有一层壳、只有一个布局时，官方那条 important 直接建议你别用布局，改在 `app.vue` 里写。真有多个布局时，布局的名字不写在文件里，而是被汇成一张表。这张表就是这一站的落点。

## 三个词先说清

**路由记录**：`vue-router` 内部的一条注册，含 `path`（地址形状）、`name`（可以按名字跳）、`children`（子路由）、`meta`（这一条路由的元信息）。

**布局**：包在页面外面的一层壳，页面内容通过它里面的 `<slot />` 露出来。它的根元素**不能就是 `<slot />`**——官方那条 important 说明 reasons 是布局切换要有过渡。

**`definePageMeta`**：写在页面文件里的编译期宏。它会被搬出组件，挂到那条路由记录的 `meta` 上，所以它里面**不能引用组件内的响应式数据**，只能引用导入进来的东西或本地定义且无副作用的纯函数。

## pages 怎么变成地址

文件名到地址的对应，文档写了几条，本轮一次量出来（后四行是探针文件，量完就删）：

| 文件 | 路由记录里读到的 |
| --- | --- |
| `app/pages/index.vue` | `/` 名字 `index` |
| `app/pages/me.vue` | `/me` 名字 `me` |
| `app/pages/docs/index.vue` | `/docs` 名字 `docs` |
| `app/pages/docs/[slug].vue` | `/docs/:slug()` 名字 `docs-slug` |
| `app/pages/console/index.vue` | `/console` 名字 `console` |
| `app/pages/(probegroup)/probe-about.vue` | `/probe-about` 且 `groups=["probegroup"]` |
| `app/pages/probe-catch/[...slug].vue` | `/probe-catch/:slug(.*)*` 名字 `probe-catch-slug` |
| `app/pages/probe-parent.vue` ＋ `app/pages/probe-parent/probe-child.vue` | `/probe-parent` 带 `children=probe-child`，子记录名字 `probe-parent-probe-child` |

这几行背后是六条规则，逐条对得上：

- 页面可以是任何 Nuxt 认的扩展名，默认 `.vue`、`.js`、`.jsx`、`.mjs`、`.ts`、`.tsx`；渲染函数写法也算一个页面。
- 方括号里的东西变成动态参数；要让参数可省，得写成双层方括号 `[[slug]].vue`，它同时匹配 `/` 和 `/test`。
- `[...slug].vue` 是 catch-all，匹配该路径下的一切，参数是一个数组。
- 目录名不影响地址，除非同层有一个同名文件：`probe-parent.vue` 与 `probe-parent/probe-child.vue` 就生成了父子两条记录，子级要显示出来得在父级里放一个 `<NuxtPage />`。
- 用圆括号包起来的目录只是分组，不进 URL；从 v4.3 起它会自动出现在 `route.meta.groups` 里（本轮那行 `groups=["probegroup"]` 就是它）。
- 具名父路由优先于嵌套动态路由：`/foo/hello` 会被 `foo.vue` 抓住而不是 `foo/[slug].vue`；要分开接，就用 `foo/index.vue` 加 `foo/[slug].vue`。

另外三条本轮没跑（是文档的规矩，不是这一轮的读数）：页面**必须只有一个根元素**，而且 HTML 注释也算一个元素——服务端渲出来正常、客户端跳转过去那一个页面不显示，就是踩了这条；`.client.vue` 与 `.server.vue` 后缀能让整张页面只在某一侧渲；v4.5 起还能用 `name@view.vue` 这种同层兄弟文件，给父级里多个具名 `<NuxtPage>` 出口各喂一个组件，但 `definePageMeta` 只从默认那份文件读。

## layouts 的名字是算出来的

布局 nested 目录也扫，名字按路径拼、重复的段去掉。文档给的两张表，本轮用探针一一对上：

| 文件 | 注册出来的名字 |
| --- | --- |
| `app/layouts/probe-dir/index.vue` | `probe-dir` |
| `app/layouts/probe-dir/inner.vue` | `probe-dir-inner` |
| `app/layouts/probe-solo/solo.vue` | `probe-solo` |
| `app/layouts/probeCustom.vue` | `probe-custom` |
| `app/layouts/default.vue` | `default` |

`probe-solo/solo.vue` 那一行是"重复段去掉"的样子（目录尾段和文件名相同，只留一次）；`probeCustom.vue` 那行是**布局名会归一成 kebab-case**，文档同样的例子是 `errorLayout.vue` 要用 `error-layout` 呼叫。所以推荐的做法就是文档那句：文件名跟名字长一样，省得心里换算。

`nuxt prepare` 之后，这些名字落在两份生成物里：

```ts [来自 .nuxt/types/layouts.d.ts]
declare module 'nuxt/app' {
  interface NuxtLayouts {
    default: ComponentProps<typeof import(".../app/layouts/default.vue").default>
    "probe-dir": ComponentProps<typeof import(".../app/layouts/probe-dir/index.vue").default>
    "probe-custom": ComponentProps<typeof import(".../app/layouts/probeCustom.vue").default>
  }
  export type LayoutKey = keyof NuxtLayouts extends never ? string : keyof NuxtLayouts
  interface PageMeta {
    layout?: MaybeRef<LayoutKey | false> | ComputedRef<LayoutKey | false> | { ... }
  }
}
```

```ts [来自 .nuxt/types/nitro-layouts.d.ts]
export type LayoutKey = "default" | "probe-dir" | "probe-dir-inner" | "probe-solo" | "probe-custom"
declare module 'nitropack' {
  interface NitroRouteConfig {
    appLayout?: LayoutKey | false
  }
}
```

第二份是服务端一侧的：`LayoutKey` 是一个字面量联合，而 `appLayout` 这个路由规则位的类型就是它。也就是说布局名不只活在页面里，配置里那一位也一起被钉住。对照一下上一站：`app/layouts` 的名字进的是这种类型联合，`app/types` 与 `server/types` 里的东西连注册表都进不去。

## 一趟红趟量出四道护栏

在探针页面里故意写四个错，`npx nuxt typecheck` → `rc=2`，四行各出现一次：

```text
app/pages/probe-meta.vue(7,7): error TS2353: Object literal may only specify known properties,
  and 'nope' does not exist in type 'NonNullable<{ readonly sidebar?: boolean | undefined; }
  & VNodeProps & AllowedComponentProps & ComponentCustomProps>'.
app/pages/probe-meta.vue(16,17): error TS2322: Type '"probe-ghost"' is not assignable to type
  'MaybeRef<false | keyof NuxtLayouts> | ComputedRef<...> | { name?: ... } | ... 4 more ...'.
app/pages/probe-meta.vue(21,35): error TS2322: Type '"probe-ghost"' is not assignable to type ...
nuxt.config.ts(65,23): error TS2322: Type '"probe-ghost"' is not assignable to type
  'false | LayoutKey | undefined'.
```

四道护栏分别挡的是：布局对象写法里的 `props` **只认那个布局自己 `defineProps` 声明过的位**；`<NuxtLayout>` 的 `name`；`<NuxtLayout>` 的 `fallback`（这一条容易以为不管写什么都行）；`nuxt.config.ts` 里 `routeRules` 的 `appLayout`。第四个尤其值得留意——配置文件也在护栏内，写错布局名不用等运行时。

运行时那一侧的规矩是文档写明的：`name` 传了无效布局名时**什么都不渲染**，这时 `fallback` 才有机会顶上；`fallback` 得指一个真实存在的布局，否则等于没写。

## 自定义 meta：写得进去，读不出来

`definePageMeta` 里加一位自己的东西，类型检查不管：

```vue [app/pages/probe-meta.vue]
<script setup lang="ts">
definePageMeta({
  layout: { name: 'probe-custom', props: { sidebar: true } },
  pageType: 'probe',
})
</script>
```

这一趟 `rc=0`。但另一处想读它，就红了：

```vue
const probeRoute = useRoute()
const probeKind: string = probeRoute.meta.pageType ?? '-'
```

```text
app/pages/probe-meta.vue(13,7): error TS2322: Type '{}' is not assignable to type 'string'.
```

按文档那段配方补一个扩展，`rc=0`：

```ts [app/types/probe-page-meta.ts]
declare module '#app' {
  interface PageMeta {
    pageType?: string
  }
}

export {}
```

两件事要一起记住。第一，那句 `export {}` 不是装饰——文档专门写了"扩展类型时一定要 import 或 export 点什么"。第二，这个文件放在 `app/types/` 里，而上一站刚量过这个目录**不进自动导入注册表**：这里照样生效，因为类型扩展不需要被谁 import，只要它在那个类型上下文的项目里被扫到（`app` 那一份配置把 `../app/**/*` 全收了）。文档给的对应关系就三句：要扩 app 上下文的类型就放 `app/`，要扩服务端的就放 `server/`，两边都要的放 `shared/`。

## definePageMeta 里那些有特殊用途的位

除自定义位之外，这十个键有明确语义，全量列一遍：

| 键 | 作用 |
| --- | --- |
| `alias` | 同一张页面挂多个地址，字符串或数组 |
| `keepalive` | 为真时页面被包进 `<KeepAlive>`；父路由要保状态用 `<NuxtPage keepalive />` |
| `key` | 控制子路由何时重渲，可传函数；也可由父级 `<NuxtPage :page-key>` 决定 |
| `layout` | `false`、布局名字符串、ref/computed，或 `{ name, props }` 对象写法 |
| `layoutTransition` | 包在布局外的 `<transition>` 的属性，`false` 关掉 |
| `pageTransition` | 同上，管页面 |
| `middleware` | 这条路由要跑的中间件，字符串、函数或数组，与父子级的合并 |
| `name` | 给这条路由起名，覆盖按文件名算出来的那个 |
| `path` | 文件名表达不出来的匹配模式，直接写 `vue-router` 的路径语法 |
| `props` | 把路由参数当 props 交给页面组件 |

嵌套路由时父子各处的 page meta 会合并成一个对象。布局还有第三个开关：`routeRules` 里的 `appLayout`（v4.3），适合集中管一批地址，或者给那些没有对应页面组件的地址（catch-all 匹配到的一堆路径）指定布局。

## 出问题对照

| 症状 | 原因 | 怎么办 |
| --- | --- | --- |
| 客户端跳过去白屏，直接访问那地址又正常 | 页面有多个根元素，或根前面留了 HTML 注释 | 收成一个根元素；注释挪进根里面 |
| `layout: 'custom'` 报 TS2322 | 那个名字不在 `NuxtLayouts` 里 | 确认 `app/layouts/custom.vue` 存在；写对象形式要连 `props` 一起对上 |
| 布局文件叫 `errorLayout.vue`，`name="errorLayout"` 不生效 | 名字归一成 kebab-case | 用 `error-layout` |
| 布局没渲染出来，也没报错 | `name` 传了无效值 | 给 `<NuxtLayout>` 配 `fallback`，且 `fallback` 要指真存在的布局 |
| 布局的过渡不生效 | `<NuxtLayout>` 成了页面的根元素 | 外面再套一层，或者关掉过渡 |
| 子页面显示不出来 | 父级里没有 `<NuxtPage />` 出口 | 在父页面里放出口 |
| 分组目录里的页面 `groups` 读不到 | v4.3 才自动注入这一位 | 升级或者自己写进 `definePageMeta` |
| 只想用 `app.vue` 却不出现路由 | `pages` 目录是可选的 | 建 `app/pages/`，或 `pages: true` 强制启用 |

## 自己跑一遍

1. 建一个 `app/pages/probe-x.vue`，跑 `npx nuxt dev`，访问 `/probe-x`——地址是文件名给的。
2. 把它挪进 `app/pages/(group)/probe-x.vue`，重启 dev，地址不变；在页面里 `console.log(useRoute().meta.groups)`，能看到 `['group']`。
3. 建 `app/layouts/probeLayout.vue`，在任何页面写 `definePageMeta({ layout: 'probelay' })`——红；改成 `'probe-layout'` 就通，这一红一绿就是 kebab 归一。
4. 在这个布局里 `defineProps<{ sidebar?: boolean }>()`，页面用对象写法给 `props: { sidebar: true }` 通、给一个没声明的位红。
5. 把布局文件删掉，跑 `npx nuxt prepare`，再打开 `.nuxt/types/nitro-layouts.d.ts`：第一行那个 `LayoutKey` 联合里它已经没了。

本轮十个探针文件（六个页面、四个布局）与一个扩展文件、一个测试文件都已删净，两个空目录回收；复跑 `nuxt prepare` 后 `.nuxt` 里搜 `probe` 命中 0、`LayoutKey` 回到只有 `"default"`，`npx nuxt typecheck` **rc=0、输出 0 字节**，`npm run lint` **rc=0**（还挂着那条 `vue/no-v-html` 的 warning），Nuxt 那一层 5 个测试文件 14 条用例全过。

下一步 [741《目录边界（4）：server／api 的地址前缀与方法后缀》](/posts/5b09d6d3d7584335e43e5a21bc471e6e/)：`server/api/` 里的文件为什么自动带 `/api` 前缀、后缀 `.get` 与 `.delete` 怎么被认成方法、写不认的后缀会发生什么——还有文档承诺的 405 在产物上到底出没出现。

## 本文依据

- 文件路由的全部规则（可选目录与 `pages: true`、扩展名、单根元素、动态与可选参数、catch-all、嵌套路由与子路由 key、具名视图 v4.5、路由分组与 `meta.groups` v4.3、`definePageMeta` 的十个特殊位与自定义位扩展、客户端与服务端专用页面、多 pages 目录走 layers）：`https://nuxt.com/docs/4.x/directory-structure/app/pages`
- 布局的启用方式、kebab-case 归一、没指定就用 `default`、单布局建议改用 `app.vue`、根元素不能是 `<slot />`、嵌套目录的命名两张表、`setPageLayout`、`appLayout` 路由规则 v4.3、布局 props 的类型化：`https://nuxt.com/docs/4.x/directory-structure/app/layouts`
- `<NuxtLayout>` 的 `name` 与 `fallback`、额外属性走 `$attrs`、`layoutRef`、过渡建议不要当根元素：`https://nuxt.com/docs/4.x/api/components/nuxt-layout`
- 四份类型上下文与"扩展要放在对应上下文目录"：`https://nuxt.com/docs/4.x/guide/concepts/typescript`
- 页面扩展名与 `pages` 相关配置项：`https://nuxt.com/docs/4.x/api/nuxt-config`
