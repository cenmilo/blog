---
url: /posts/bbde1d2dbc3164aed3db8c7d503aa74b/
title: 195 - head 标签的去重与清理：key、patch 与 dispose 三处原文
date: 2026-12-16T09:00:00+08:00
updated: 2026-12-16T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：同一个 `name="description"` 在页面里冒出两条，是我在改造成多组件各自 `useHead` 之后遇到的第一个怪事。这一篇只把 Nuxt 文档里跟"去重和清理"有关的三处原话并起来：第一处是模板组件那条建议——"It's suggested to wrap your components in either a `<Head>` or `<Html>` component as tags will be deduped more intuitively."，注意它给的理由是**去重更直觉**，不是"必须包"；第二处是紧跟着的 warning——"If you need to duplicate tags across client-server boundaries, apply a `key` attribute on the `<Head>` component."，反过来读就是**默认情况下跨客户端和服务端边界的重复标签会被合掉**，要保留两份才需要 `key`；第三处是 `useHead` 返回值里的两个方法，`patch` 的注释写着 "Will first clear any side effects for previous input."，`dispose` 写着 "Dispose the entry, removing it from the active head. Will queue side effects for removal."。另外还有一处必须点出来的矛盾：同一页的 Type 块签名是 `export function useHead (meta: MaybeComputedRef<MetaObject>): ActiveHeadEntry<UseHeadInput>`，而 Return Values 一节写 "This composable does not return any value."——两处都在官方页上，我照抄并给出一次实测办法。

categories:
  - 前端开发

tags:
  - nuxt4
  - useHead
  - 去重
  - head
  - DOM

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


我的第一反应是"head 里的标签是谁在管"。查下来 Nuxt 这边只是把活交给了 Unhead（SEO 页第一句就是 "Nuxt head tag management is powered by [Unhead]."），而 Nuxt 文档里明确写了去重与清理的只有下面三处。

## 第一处：包在 Head 或 Html 里，去重更直觉

SEO 页的 Components 一节，给了九个组件之后有这句建议：

> It's suggested to wrap your components in either a `<Head>` or `<Html>` component as tags will be deduped more intuitively.

两个词值得抠：一是 **suggested**，不是 required；二是 **more intuitively**，它比较的是"直觉程度"，没有承诺"包了就一定只有一条"。同一节还有一句容易被误读的：

> `<Head>` and `<Body>` can accept nested meta tags (for aesthetic reasons) but this does not affect *where* the nested meta tags are rendered in the final HTML.

我当初就是把它读成"`<Head>` 里的东西渲染进 head"。原文说的是反话：**嵌套只是为了好看，跟最终落点无关**。落点由 `tagPosition` 决定（188 讲过 `'head' | 'bodyClose' | 'bodyOpen'`）。

## 第二处：想要重复标签，就给 Head 加 key

同节末尾的 warning：

> If you need to duplicate tags across client-server boundaries, apply a `key` attribute on the `<Head>` component.

这句我的读法是**反向的**：既然"需要跨边界保留重复"时才要求你写 `key`，那么不写 `key` 时，跨边界的同类标签就会被合起来。文档没有正面写出去重算法（哪些键参与判重、`id` 之类的历史键还在不在），所以我把它记成一条**只能实测**的规则，而不是一条可以背的规则。

最省事的实测法：在两个不同组件里都注册同一个 `name="description"`，内容分别写成 `A` 和 `B`，然后

- 服务端：`nuxt preview` 起生产构建，`curl` 首页 HTML，数 `<head>` 里 `name="description"` 出现几次、留的是哪一条；
- 客户端：导航过去之后再数一次（首屏和水合后的 DOM 可能不一样，这也是为什么第 195 篇要区分"跨边界"）。

如果两处次数不同，说明判重发生在某一侧，那正好是文档没说清的地方。

## 第三处：useHead 的返回值里有 patch 和 dispose

`useHead` 页 Type 一节的完整签名（照抄）：

```ts [Signature]
export function useHead (meta: MaybeComputedRef<MetaObject>): ActiveHeadEntry<UseHeadInput>

interface ActiveHeadEntry<Input> {
  /**
   * Updates the entry with new input.
   *
   * Will first clear any side effects for previous input.
   */
  patch: (input: Input) => void
  /**
   * Dispose the entry, removing it from the active head.
   *
   * Will queue side effects for removal.
   */
  dispose: () => void
}
```

这两句注释是文档里唯一正面讲"清理"的地方，用词很实在：

| 方法 | 注释原文 | 我的理解 |
| --- | --- | --- |
| `patch(input)` | "Updates the entry with new input. **Will first clear any side effects for previous input.**" | 先清旧的副作用，再上新的——不是原地改 |
| `dispose()` | "Dispose the entry, **removing it from the active head**. Will **queue** side effects for removal." | 从活动 head 里摘掉；清理是**排队**（queue）而不是立刻 |

`queue` 那个词我不敢展开：它意味着 dispose 之后不一定在同一帧看到效果，但文档只写了这一个词，没有写队列何时被消费。要确认就调用 `dispose()` 之后立刻读一次 DOM、下一个 tick 再读一次，比较两次结果。

## 顺带必须点出来的矛盾：到底有没有返回值

同一页里两句话对不上：

- Type 一节：签名末尾是 `: ActiveHeadEntry<UseHeadInput>`；
- Return Values 一节：**"This composable does not return any value. It registers the head metadata with Unhead, which manages the actual DOM updates."**

我在 187 里已经把这条记成文档内部矛盾，这里补一个具体影响：如果按签名去写 `const entry = useHead({...})` 然后调 `entry.dispose()`，能不能跑、类型对不对，**不能从文档推**，只能在自己的版本上试一次。对照 `useHeadSafe` 页的措辞就很干脆——"## Return Values / This composable does not return any value."——它连签名都是 `: void`：

```ts
export function useHeadSafe (input: MaybeComputedRef<HeadSafe>): void
```

所以我倾向于：需要 `patch`/`dispose` 的时候先验证 `useHead` 在你这个版本上是否真给对象；拿不到就退回"用反应式值把内容置空"的写法，那是文档明确支持的（"The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties."）。

## 顺序相关的另一把开关

去重之外还有"谁排在前面"。文档里唯一一句关于排序的是 `unhead.legacy` 的说明（196 细讲）：

> Enable the legacy compatibility mode for `unhead` module. This applies the following changes: - Disables Capo.js sorting - Adds the `DeprecationsPlugin`: supports `hid`, `vmid`, `children`, `body` - Adds the `PromisesPlugin`: supports promises as input

**默认 `false`**，也就是默认启用 Capo.js 排序。这句话同时告诉我另一件事：`hid`、`vmid`、`children`、`body` 这四个老键只在 legacy 模式下被支持——我在旧文章里抄来的 `hid` 写法，在 4.x 默认配置下是无效键。

## 自查

1. 官方为什么建议把组件包进 `<Head>` 或 `<Html>`？（答：因为"tags will be deduped more intuitively"，措辞是 suggested 而非必须。）
2. 想让同一个标签在客户端和服务端边界两侧都保留，文档给的办法是？（答：在 `<Head>` 组件上加 `key` 属性。）
3. `useHead` 页里关于返回值的两句话分别是什么？（答：签名是 `: ActiveHeadEntry<UseHeadInput>`；Return Values 一节写 "does not return any value"。）

## 下一篇

196 - unhead 配置节：legacy 与 renderSSRHeadOptions 两个键

## 本文依据

- `getting-started/seo-meta`：Components 一节 "Nuxt provides the following components for this purpose: `<Title>`, `<Base>`, `<NoScript>`, `<Style>`, `<Meta>`, `<Link>`, `<Body>`, `<Html>` and `<Head>`."、"`<Head>` and `<Body>` can accept nested meta tags (for aesthetic reasons) but this does not affect *where* the nested meta tags are rendered in the final HTML."、"It's suggested to wrap your components in either a `<Head>` or `<Html>` component as tags will be deduped more intuitively."、warning "If you need to duplicate tags across client-server boundaries, apply a `key` attribute on the `<Head>` component."；开篇句 "Nuxt head tag management is powered by Unhead."。
- `api/composables/use-head`：Type 一节的 `useHead (meta: MaybeComputedRef<MetaObject>): ActiveHeadEntry<UseHeadInput>` 与 `ActiveHeadEntry` 两条注释原文；Return Values "This composable does not return any value. It registers the head metadata with Unhead, which manages the actual DOM updates."；note "The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties."。
- `api/composables/use-head-safe`：签名 `export function useHeadSafe (input: MaybeComputedRef<HeadSafe>): void` 与 "This composable does not return any value."
- `api/nuxt-config`（unhead 一节）：`legacy` 的三条变更原文与 **Default:** `false`。
