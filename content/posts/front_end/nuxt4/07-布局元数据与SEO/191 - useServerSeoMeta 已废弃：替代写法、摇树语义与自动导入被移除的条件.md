---
url: /posts/73d3b014d07c14f9e18cb52e09bf07f4/
title: 191 - useServerSeoMeta 已废弃：替代写法、摇树语义与自动导入被移除的条件
date: 2026-12-14T11:00:00+08:00
updated: 2026-12-14T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一节开头第一条就是废弃通知，措辞很短但含三个信息："> **`useServerSeoMeta` is deprecated. Wrap `useSeoMeta` in an `if (import.meta.server)` block instead. The auto-import is removed under `future.compatibilityVersion: 5`.**"。它的原语义是这一句："`useServerSeoMeta` lets you define your site's SEO meta tags as a flat object with full TypeScript support, **exactly like `useSeoMeta`, but it only runs server-side and is tree-shaken from the client bundle.**"，参数关系是"**Parameters are exactly the same as with `useSeoMeta`.**"，替代写法官方直接给了代码：`if (import.meta.server) { useSeoMeta({ robots: 'index, follow' }) }`。`useSeoMeta` 那一节也指认同一次替换："This previously used the `useServerSeoMeta` composable, **but it has been deprecated in favor of this approach**."。**三条信息里最容易读错的是第三条**：官方写的是"自动导入在 `future.compatibilityVersion: 5` 下被移除"，**字面对象是自动导入**，没有说这个函数本身是否在该版本被删除——所以老项目升级时最先炸的是"不写 import 的地方找不到这个名字"。我因此在文中把"要改什么"列成清单，并把"替代写法是否保住摇树收益"标成需要用构建产物核对的一项（原文只说旧 API "is tree-shaken from the client bundle"，新写法那句没有重复这个承诺）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useServerSeoMeta
  - 废弃
  - 自动导入
  - 升级

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


看到"已废弃"我通常只改调用点。这一条不一样：官方给的替代写法是一个 `if` 块，而它原本承诺的是"从客户端包里摇掉"。这两件事是不是等价的，值得单独想一次。这一篇把废弃通知里的三个信息拆开读。

## 废弃通知的三个分句

> `useServerSeoMeta` is deprecated. **Wrap `useSeoMeta` in an `if (import.meta.server)` block instead.** The auto-import is removed under `future.compatibilityVersion: 5`.

| 分句 | 说了什么 | 没说什么 |
| --- | --- | --- |
| ① deprecated | 这个 API 不再推荐 | 没说它在哪个版本被移除 |
| ② 替代写法 | 用 `useSeoMeta` + 服务端条件 | 没说替代写法是否同样"从客户端包摇掉" |
| ③ 自动导入条件 | `compatibilityVersion: 5` 下**自动导入**被移除 | 没说函数本体是否还在 |

**第③句要按字面读**：受影响的是"不写 import 就能用"这件事。也就是说升级那条兼容版本后，最先报错的是所有裸写 `useServerSeoMeta(...)` 的位置——名字找不到。

（`future.compatibilityVersion` 这个键本身在 138、157 出现过：它是"提前启用未来默认值"的开关，180 那条链也归它管。）

## 它原来承诺的是什么

官方对旧 API 的完整定位句：

> `useServerSeoMeta` lets you define your site's SEO meta tags as a flat object with full TypeScript support, **exactly like `useSeoMeta`, but it only runs server-side and is tree-shaken from the client bundle.**

两个承诺：

1. **只在服务端运行**；
2. **从客户端 bundle 里摇掉**。

参数关系另有一句：

> Parameters are exactly the same as `useSeoMeta`.

**这两句决定了迁移的难度**：参数一模一样，所以是机械替换；但"摇树"这件事是旧 API 的**内在性质**，替换写法能不能保住它，得看条件分支被不被当作可静态判定的死代码——官方这句替代说明没有重复该承诺。

## 官方给的替代写法

```vue [app/app.vue]
<script setup lang="ts">
if (import.meta.server) {
  useSeoMeta({
    robots: 'index, follow',
  })
}
</script>
```

`useSeoMeta` 那一页给的是更完整的两段形态（静态进条件、动态留外面），并配了官方注释：

```vue
if (import.meta.server) {
  // These meta tags will only be added during server-side rendering
  useSeoMeta({
    robots: 'index, follow',
    description: 'Static description that does not need reactivity',
    ogImage: 'https://example.com/image.png',
    // other static meta tags...
  })
}

const dynamicTitle = ref('My title')
// Only use reactive meta tags outside the condition when necessary
useSeoMeta({
  title: () => dynamicTitle.value,
  ogTitle: () => dynamicTitle.value,
})
```

**这里还有个顺序细节**：`import.meta.server` 分支内的 `useSeoMeta` 调用是**顶层同步调用**，所以它不违反 151 那条"必须同步调用 composable"的规则（`<script setup>` 本身就是允许跨 `await` 的四个场合之一，何况这里没有 `await`）。

## 替换会不会丢掉摇树收益：一项要自己核对

我的判断是把这件事列成**必须验证**，不当已知结论用：

| 已知（文档写了） | 未写 |
| --- | --- |
| 旧 API "is tree-shaken from the client bundle" | 新写法（`if (import.meta.server)`）是否同样从客户端包中移除 |
| 新写法的语义："These meta tags will only be added during server-side rendering" | —— |

**核对动作**：`nuxt build` 之后在客户端产物里搜那段独有的字符串（例如那句 `description` 的值）。搜不到就是被移除了；搜得到就说明只剩"运行时不执行"，那与旧 API 不等价，需要重新权衡。这个检查一次几分钟，比引用一句我没验证的承诺可靠。

（顺带一提，`nuxt analyze` 也能看这块，175 讲过它的用法。）

## 迁移清单

按这个顺序做，一次过：

1. **找出所有调用点**：`useServerSeoMeta(` 全文检索。
2. **逐个替换**为 `if (import.meta.server) { useSeoMeta({...}) }`，参数原样搬（官方那句"Parameters are exactly the same"允许你不动内容）。
3. **检查这些调用是否都在 `<script setup>` 顶层**（不在的话，`import.meta.server` 分支里的 composable 调用位置要复核，见 151 的上下文规则）。
4. **核对产物**（上一节那条）：确认静态那段没有留在客户端包里；如果留着，考虑把整段逻辑上移到服务端专属位置（例如 nitro 路由或插件）——这一步是设计变更，要单独评估。
5. **顺手处理兼容版本**：如果你正打算升 `future.compatibilityVersion: 5`，第 1 步的检索范围要包括**没写 import 的裸调用**（第③分句的直接影响面）。

## 与相邻篇章的分工

| 篇章 | 关系 |
| --- | --- |
| 190 | 替代写法所在：`useSeoMeta` 的字段与反应式形态 |
| 186 | 三处 head 入口的选择：静态默认值该进 `app.head` 而不是包在条件里 |
| 189 | `useServerSeoMeta` 没有"值不可信"这条职责；用户输入仍要走 `useHeadSafe` |
| 149 | "服务端跑一次、客户端水合一次"这条时间线，是理解 `import.meta.server` 分支为什么有效的前提 |

## 自查

1. 废弃通知的三个分句分别说了什么？各自"没说什么"？
2. 第③句的字面对象是"自动导入"，这对升级时的老代码意味着什么具体故障？
3. 旧 API 的两个承诺是什么？替代写法官方怎么写的？
4. 参数关系那句原文是什么？它让迁移变成机械替换的原因？
5. "替代写法是否保住摇树"这件事文档给了答案吗？你的核对动作是什么？
6. `if (import.meta.server)` 分支里调用 composable 会不会违反同步规则？为什么？
7. 迁移清单的第 3 步在防什么问题？

## 下一篇

192 摊平 OG 与 Twitter 的字段映射：官方示例里出现过的扁平字段名各对应哪种 `<meta>` 元素、`name` 与 `property` 的分工，以及为什么官方把"用 `name` 写 `og:title`"列为常见错误。
