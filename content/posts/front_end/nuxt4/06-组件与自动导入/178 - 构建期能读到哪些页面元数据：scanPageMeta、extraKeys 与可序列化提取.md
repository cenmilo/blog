---
url: /posts/9c588ffeee60b348234ea689af915e6d/
title: 178 - 构建期能读到哪些页面元数据：scanPageMeta、extraKeys 与可序列化提取
date: 2026-12-10T10:00:00+08:00
updated: 2026-12-10T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：143 说过"构建期只提取六个 key"，这一篇补齐**控制这件事的三个开关**。官方的总体说明是："The `definePageMeta()` macro is a useful way to collect build-time meta about pages. **Nuxt itself provides a set list of supported keys which is used to power some of the internal features such as redirects, page aliases and custom paths.**"**开关一 `scanPageMeta`**：官方点名那份清单是 `alias`、`name`、`path`、`redirect`、`props`、`middleware`——"Nuxt exposing some route metadata defined in `definePageMeta` at build-time to modules (specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`)."，并限定写法："**This only works with static or strings/arrays rather than variables or conditional assignment.**"；时机句是"**By default page metadata is only scanned after all routes have been registered in `pages:extend`. Then another hook, `pages:resolved` will be called.**"；关档 `experimental.scanPageMeta: false`。**开关二 `extraPageMetaExtractionKeys`**："This option allows passing additional keys to extract from the page metadata **when using `scanPageMeta`**."，官方示例是页面里 `definePageMeta({ foo: 'bar' })` + `extraPageMetaExtractionKeys: ['foo']` + `hooks: { 'pages:resolved' (ctx) { // ✅ foo is available } }`，收尾一句给模块作者："This allows modules to access additional metadata from the page metadata in the build context. **If you are using this within a module, it's recommended also to augment the `NuxtPage` types with your keys.**"**开关三 `extractSerializablePageMeta`**（默认随 `future.compatibilityVersion` 到 5 才开）："**By default, Nuxt only reads a fixed list of `definePageMeta()` keys at build time (plus anything in `extraPageMetaExtractionKeys`); everything else is resolved at runtime from a separate module generated per page. With this option, Nuxt writes every JSON-serializable property straight into the generated route record instead.**"，收益句是"When every property of a page's `definePageMeta()` can be resolved statically, the route no longer imports that page's meta module at all, **which removes one module per page from the dev module graph**."，两条边界是"Properties whose values cannot be serialized (functions, variable references, spreads, computed keys) are unaffected and still resolved at runtime." 与 "**This has no effect when `scanPageMeta` is `false`**, as the route record does not override the meta module in that case."

categories:
  - 前端开发

tags:
  - nuxt4
  - definePageMeta
  - 实验特性
  - 构建期
  - 模块

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


143 里我记下"构建期只看得见六个 key"。后来想在模块里读一个自定义元数据（比如页面级的权限标记），就撞上第二个开关；再往后想把整份元数据静态化以减少 dev 时的模块数，又撞上第三个。这一篇把三个开关一次讲完。

## 三个开关管的是同一件事

官方对这件事的总述只有一句，但它定义了全部前提：

> The `definePageMeta()` macro is a useful way to collect build-time meta about pages. Nuxt itself provides a set list of supported keys which is used to power some of the internal features such as redirects, page aliases and custom paths.

| 开关 | 管什么 | 默认 |
| --- | --- | --- |
| `experimental.scanPageMeta` | **要不要在构建期扫页面元数据**，以及能看到哪六个 key | 默认在扫（官方句："By default page metadata is only scanned after all routes have been registered in `pages:extend`."） |
| `experimental.extraPageMetaExtractionKeys` | **在固定清单之外再多读哪几个自定义 key** | 无（空） |
| `experimental.extractSerializablePageMeta` | **把所有可序列化的属性直接写进路由记录**，省掉每页一个 meta 模块 | `future.compatibilityVersion >= 5` 时才默认开 |

**三者的关系是有方向的**：第三个明确"对 `scanPageMeta: false` 无效"，第二个明确"配合 `scanPageMeta` 使用"。所以它们不是三个独立开关，而是一层依赖两层。

## 开关一：那份固定清单

官方给的范围与写法限制（143 也引过，这里补齐语境）：

> Nuxt exposing some route metadata defined in `definePageMeta` at build-time to modules (specifically `alias`, `name`, `path`, `redirect`, `props` and `middleware`).

> This only works with static or strings/arrays rather than variables or conditional assignment. See original issue for more information and context.

> By default page metadata is only scanned after all routes have been registered in `pages:extend`. Then another hook, `pages:resolved` will be called.

> You can disable this feature if it causes issues in your project.

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    scanPageMeta: false,
  },
})
```

**这六个 key 为什么是这六个**，官方那句给了答案的一半：它们服务于框架内部功能——"redirects, page aliases and custom paths"。也就是说这份清单**不是为了让你读元数据而存在的**，它是框架自身构建路由表需要的最小集。

这解释了 143 那条顺序后果：`layout`、`keepalive`、`pageTransition` 这些**渲染期行为**不在清单里，构建期拿不到，也不需要拿到。

## 开关二：多加几个自定义 key

官方定位句：

> This option allows passing additional keys to extract from the page metadata **when using `scanPageMeta`**.

官方那一节把三样东西并成一个示例，很值得原样抄：

```vue
<script lang="ts" setup>
definePageMeta({
  foo: 'bar',
})
</script>
```

```ts
export default defineNuxtConfig({
  experimental: {
    extraPageMetaExtractionKeys: ['foo'],
  },
  hooks: {
    'pages:resolved' (ctx) {
      // ✅ foo is available
    },
  },
})
```

三个细节：

| 细节 | 依据 |
| --- | --- |
| 值要在**页面里也声明**，配置里只是把它加入提取清单 | 示例两段合起来才成立 |
| 拿到它的位置是 **`pages:resolved`**，不是 `pages:extend` | 示例的钩子名；原因见开关一的顺序句 |
| 官方注释直接写 `// ✅ foo is available` | 也就是"在 `pages:extend` 里不可用" |

收尾那句是给模块作者的硬要求：

> This allows modules to access additional metadata from the page metadata in the build context. If you are using this within a module, **it's recommended also to augment the `NuxtPage` types with your keys**.

链接指向官方的 "Typing Custom Metadata" 一节。**增强写在哪取决于你在哪读**（143 那张表）：钩子里读 `page.meta.foo` 要增强 `@nuxt/schema` 的 `NuxtPageMeta`；页面里 `definePageMeta({ foo: 'bar' })` 要增强 `#app` 的 `PageMeta`。**两边都写才不会一处报错一处沉默**。

## 开关三：把可序列化的部分写进路由记录

这一条最容易被忽略，因为它解决的问题你平时看不见：

> By default, Nuxt only reads a fixed list of `definePageMeta()` keys at build time (plus anything in `extraPageMetaExtractionKeys`); **everything else is resolved at runtime from a separate module generated per page**. With this option, **Nuxt writes every JSON-serializable property straight into the generated route record instead**.

官方给的收益句是把代价说清楚：

> When every property of a page's `definePageMeta()` can be resolved statically, the route no longer imports that page's meta module at all, **which removes one module per page from the dev module graph**.

官方示例给的是一个"整页都能静态化"的形状：

```vue
<script lang="ts" setup>
definePageMeta({
  layout: { name: 'admin', props: { collapsed: true } },
  title: 'Dashboard',
})
</script>
```

两条边界：

> Properties whose values cannot be serialized (functions, variable references, spreads, computed keys) are unaffected and still resolved at runtime.

> **This has no effect when `scanPageMeta` is `false`**, as the route record does not override the meta module in that case.

默认状态与另两个开关不同，官方写清了条件：

> Enabled by default when `future.compatibilityVersion` is `5` or higher.

也就是说在当前的兼容版本下**它是关着的**，想立刻享受这份收益要显式开：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  experimental: {
    extractSerializablePageMeta: false,
  },
})
```

（官方给的就是这个关档示例——它同时说明这一项**可以被显式设成布尔值**，"enabled by default when compatibilityVersion is 5" 指的是默认取值，不是只能靠版本升。）

## 一张决策树：我的自定义 key 能不能在构建期读到

| 问自己 | 结论 |
| --- | --- |
| 这个 key 在不在那六个里？ | 在 → 直接能在 `pages:resolved`/`pages:extend` 拿到（前提 `scanPageMeta` 没关） |
| 不在，但我是模块作者、需要构建期分支逻辑？ | 加进 `extraPageMetaExtractionKeys`，并在 `pages:resolved` 读；同时增强 `NuxtPageMeta` |
| 不在，我只是运行时要看它？ | 什么都不用配，`useRoute().meta` 里那份是运行时解析的（这正是默认那份"每页一个 meta 模块"提供的） |
| 我想让静态值少一个模块、顺便省 dev 开销？ | 开 `extractSerializablePageMeta`；注意函数/变量引用/展开/计算键仍走运行时 |
| 我关了 `scanPageMeta`？ | 后两个开关都别指望生效——第三个官方明说无效，第二个的机制本身依赖扫描 |

**最后这行是本篇最需要记住的一条**：三个开关里只有 `scanPageMeta` 是根，另两个都是它的附加。

## 顺带一个"变量写法"的老问题

开关一那句"only works with static or strings/arrays rather than variables or conditional assignment"和 117 讲的编译器宏约束是同一件事的两面：

| 写法 | 运行时 | 构建期提取 |
| --- | --- | --- |
| `middleware: ['auth']` | ✅ | ✅ |
| `title: 'Dashboard'` | ✅ | 需开关三（不在六个 key 里） |
| `layout: { name: 'admin', props: {...} }` | ✅ | 需开关三 |
| `middleware: someVar` | ✅ | ❌ 拿不到 |
| `validate: cond ? fnA : fnB` | ✅ | ❌ 条件赋值 |

所以同一个值"能不能被模块看到"取决于它的**写法**，而不只是它的**名字**——这一层区分的是两种情况"明明写了却没生效"。

## 自查

1. 三个开关分别管什么？官方那句总述里"set list of supported keys"是为了支撑哪三类内部功能？
2. 那六个 key 是什么？为什么 `layout` 不在里面？
3. 顺序句决定了自定义 key 只能在哪个钩子里读到？官方注释怎么标的？
4. `extraPageMetaExtractionKeys` 的生效前提是哪一句里的哪个词？模块作者还要做什么、为什么两处增强都要写？
5. `extractSerializablePageMeta` 省掉的是什么？收益句里的数量关系是什么？
6. 哪四种值不受它影响？它对哪个开关关闭时完全无效？
7. 它的默认开启条件是什么？`experimental: { extractSerializablePageMeta: false }` 这段示例说明了什么？
8. 只用运行时元数据的场景，为什么要避免为它开这三个开关？
9. `title: 'Dashboard'` 与 `middleware: ['auth']` 在构建期可见性上有什么区别？

## 下一篇

179 给这一段的收尾：一张"自动导入或组件解析失败"的排错清单，按第 ① 到第 ⑤ 步逐级检查，并列出官方给的三件能看见真相的工具。
