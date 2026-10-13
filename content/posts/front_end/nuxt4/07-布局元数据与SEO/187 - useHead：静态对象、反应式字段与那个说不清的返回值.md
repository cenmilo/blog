---
url: /posts/08c68f0046570fb7bc352a578ec50f52/
title: 187 - useHead：静态对象、反应式字段与那个说不清的返回值
date: 2026-12-13T10:00:00+08:00
updated: 2026-12-13T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`useHead` 我用了很久都没注意一件事：**官方同一页里对返回值给了两种说法**。这一篇先把入参形状定清，再处理这个矛盾。定性句是："The `useHead` composable allows you to manage your head tags in a programmatic and reactive way, powered by Unhead. It lets you customize the meta tags, links, scripts, and other elements in the `<head>` section of your HTML document."，参数与反应式的范围由一条 note 说全："**The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties. The `meta` parameter can also accept a function returning an object to make the entire object reactive.**"，Parameters 一节再补一句："All properties support reactive values (`ref`, `computed`, `reactive`) **or can be a function returning the metadata object**."；SEO 那一节的措辞是三种："Reactivity is supported on all properties, by providing **a computed value, a getter, or a reactive object**."。**矛盾在返回值**：类型签名写的是 `export function useHead (meta: MaybeComputedRef<MetaObject>): ActiveHeadEntry<UseHeadInput>`，那个 `ActiveHeadEntry` 带两个方法——`patch: (input: Input) => void`（注释 "Updates the entry with new input. **Will first clear any side effects for previous input.**"）与 `dispose: () => void`（注释 "Dispose the entry, removing it from the active head. **Will queue side effects for removal.**"）；可下面的 "Return Values" 一节写的是"**This composable does not return any value. It registers the head metadata with Unhead, which manages the actual DOM updates.**"。我的处理是**按"不返回东西"来写代码**，需要 dispose/patch 时先在项目里确认能拿到实例。**还有一条别忘**："> If the data comes from a user or other untrusted source, we recommend you check out `useHeadSafe`."

categories:
  - 前端开发

tags:
  - nuxt4
  - useHead
  - 反应式
  - unhead
  - 返回值

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


很容易把 `useHead` 当成"把 meta 数组传进去"。直到要在一个组件卸载时手动清掉它写进去的标签，才去翻返回值那一节——然后发现**同一页里两处说法不一致**。这一篇把入参形状、反应式的三种写法和这个矛盾一次处理掉。

## 定性：反应式是它的第一属性

官方描述句：

> The `useHead` composable allows you to manage your head tags in a **programmatic and reactive** way, powered by Unhead. It lets you customize the meta tags, links, scripts, and other elements in the `<head>` section of your HTML document.

最小形态是静态对象（`app/app.vue` 官方示例）：

```vue [app/app.vue]
<script setup lang="ts">
useHead({
  title: 'My App',
  meta: [
    { name: 'description', content: 'My amazing site.' },
  ],
  bodyAttrs: {
    class: 'test',
  },
  script: [{ innerHTML: 'console.log(\'Hello world\')' }],
})
</script>
```

这一例里有 `bodyAttrs` 与 `script.innerHTML`，也就是 head 管理的不止 `<meta>`（字段全表见 188）。

## 入参可以反应式的四种写法

官方的范围句写在 note 里，两句都要读：

> The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties. **The `meta` parameter can also accept a function returning an object to make the entire object reactive.**

> All properties support reactive values (`ref`, `computed`, `reactive`) or can be a function returning the metadata object.

SEO 那一节给的三分法措辞略有不同，一并记着："Reactivity is supported on all properties, by providing **a computed value, a getter, or a reactive object**."

对到代码上是四种可写形状：

**一：字段值是 ref/computed**

```vue [app/pages/profile.vue]
<script setup lang="ts">
const profile = ref({ name: 'John Doe' })

useHead({
  title: computed(() => profile.value.name),
  meta: [
    {
      name: 'description',
      content: computed(() => `Profile page for ${profile.value.name}`),
    },
  ],
})
</script>
```

**二：整个入参是函数**

```vue [app/pages/dynamic.vue]
<script setup lang="ts">
const count = ref(0)

useHead(() => ({
  title: `Count: ${count.value}`,
  meta: [
    { name: 'description', content: `Current count is ${count.value}` },
  ],
}))
</script>
```

**三：直接传 ref（SEO 页的写法）**

```vue
const description = ref('My amazing site.')

useHead({
  meta: [
    { name: 'description', content: description },
  ],
})
```

**四：`reactive` 对象**（官方列在支持列表里，`ref`/`computed`/`reactive` 三个词并排出现）

**选一还是选二有实际后果**：形状二把 `title` 与 `meta` 放在同一个函数里，两者必然同步更新；形状一里 `title` 与 `content` 各自是独立 computed，靠同一个 `profile.value.name` 才一致。**当我需要"这一组标签一起变"，我写成函数形态**。

## 那个矛盾：类型说返回 entry，正文说没有返回值

同一页里两处并排：

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

而下面的 "## Return Values" 是一句否定：

> **This composable does not return any value.** It registers the head metadata with Unhead, which manages the actual DOM updates.

**两处不能同时为真**。我的处理规则：

| 我要做的 | 怎么办 | 理由 |
| --- | --- | --- |
| 常规声明标签、随响应式更新 | 直接 `useHead({...})`，不接返回值 | 两处说法在这一致：注册即生效，DOM 由 Unhead 管 |
| 组件卸载后自动清理 | 什么都不做 | "It registers the head metadata with Unhead" —— 生命周期由管理器接管（这也是 `useHead` 必须在 setup 同步调用的原因，151） |
| **手动改一份已注册的 head 输入 / 提前销毁它** | 先测：`const entry = useHead({...)}; console.log(entry)` | 签名说能拿到 `patch`/`dispose`，正文说拿不到；我不能替文档保证哪个是现状 |

稳妥的默认姿势是**按"不返回东西"写代码**：真需要 `patch`/`dispose` 时，先验证再依赖，并把这段依赖当作可能随版本变的东西。（对照：`useHeadSafe` 那一节的 "Return Values" 只有一句 "This composable does not return any value."，**没有**对应的 entry 签名——两节不同步，说明这一处需要以实测为准。）

## 别忘那条安全警告

官方在用法之后立刻挂了这条：

> If the data comes from a user or other untrusted source, we recommend you check out `useHeadSafe`.

它指的不是"加密"，而是**白名单过滤**（189 有整张表）。这一条与"能不能反应式"无关，所以官方把它放在了同一页最显眼的位置。

## 自查

1. 官方对 `useHead` 的两个形容词是什么？它管的东西除了 `<meta>` 还包括哪些（举示例里的两个字段）？
2. 入参支持哪几种反应式形态？三处文档（note、Parameters、SEO 页 Features）的措辞各是什么？
3. "整个入参写成函数"与"每个字段各自 computed"的差别在哪种场景下会显形？
4. 返回值这一处，同一页里哪两句冲突？各自的原文是什么？
5. 你按哪条规则处理这个冲突？如果确实需要 `dispose`，动作是什么？
6. 为什么"组件卸载后自动清理"这件事不需要你写代码？它与 151 的同步调用规则有什么关系？
7. `useHeadSafe` 那条警告针对的是什么风险？它和反应式是同一维度吗？

## 下一篇

188 摊平 `useHead` 的十个字段：`title` 与 `titleTemplate`、`templateParams` 的 `%s` 之外的占位、九类标签各自的映射规则，以及 `tagPosition` 那个能把标签放进 `<body>` 的选项。

## 本文依据

- `https://nuxt.com/docs/4.x/api/composables/use-head`
  - 原文（描述与示例）："useHead customizes the head properties of individual pages of your Nuxt app."；"The `useHead` composable allows you to manage your head tags in a programmatic and reactive way, powered by Unhead. It lets you customize the meta tags, links, scripts, and other elements in the `<head>` section of your HTML document."；`app/app.vue` 示例（`title`、`meta`、`bodyAttrs: { class: 'test' }`、`script: [{ innerHTML: 'console.log(\'Hello world\')' }]`）
  - 原文（warning 与 note）："If the data comes from a user or other untrusted source, we recommend you check out `useHeadSafe`."；"The properties of `useHead` can be dynamic, accepting `ref`, `computed` and `reactive` properties. The `meta` parameter can also accept a function returning an object to make the entire object reactive."；Parameters："An object accepting head metadata properties to customize the page's `<head>` section. All properties support reactive values (`ref`, `computed`, `reactive`) or can be a function returning the metadata object."
  - 原文（Type 与返回值矛盾两处）：`export function useHead (meta: MaybeComputedRef<MetaObject>): ActiveHeadEntry<UseHeadInput>`；`interface MetaObject { title?, titleTemplate?, base?, link?, meta?, style?, script?, noscript?, htmlAttrs?, bodyAttrs? }`；`interface ActiveHeadEntry<Input> { patch: (input: Input) => void; dispose: () => void }` 及其注释 "Updates the entry with new input. Will first clear any side effects for previous input." / "Dispose the entry, removing it from the active head. Will queue side effects for removal."；"## Return Values — This composable does not return any value. It registers the head metadata with Unhead, which manages the actual DOM updates."；"See [@unhead/schema](...) for more detailed types."
  - 原文（示例族）：Basic Meta Tags（`app/pages/about.vue` 的 `title: 'About Us'` 与 `og:title`/`og:description`）、Reactive Meta Tags（`title: computed(() => profile.value.name)` 与 `content: computed(...)`）、Using a Function for Full Reactivity（`useHead(() => ({ title: \`Count: ${count.value}\`, ... }))`）
- `https://nuxt.com/docs/4.x/getting-started/seo-meta`
  - 原文（三种反应式来源）："Reactivity is supported on all properties, by providing a computed value, a getter, or a reactive object." + `useHead` / `useSeoMeta` / 组件三栏 code-group（`content: description` 的 ref 直传写法）
- `https://nuxt.com/docs/4.x/api/composables/use-head-safe`
  - 对照用（见 189）："## Return Values — This composable does not return any value."，且其签名 `export function useHeadSafe (input: MaybeComputedRef<HeadSafe>): void` 没有 entry 返回类型
- `https://nuxt.com/docs/4.x/guide/concepts/auto-imports`
  - 对照用（见 151）：composable 必须在插件/路由中间件/setup 中**同步**调用——`useHead` 属于这一族
