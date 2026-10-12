---
url: /posts/70dad9947cc299b632d60290c8208571/
title: extensions：解析器认哪些扩展名，三处清单并不相同
date: 2026-11-08T09:00:00+08:00
updated: 2026-11-08T09:00:00+08:00
author: cmdragon

summary: >-
  摘要：extensions 这一项官方只给了一句定义与一份默认值——它是"应当由 Nuxt 解析器解析的扩展名"，类型为 array，默认依次是 .js、.jsx、.mjs、.ts、.tsx 与 .vue。但它在本系列里牵出三张并不相同的清单：nuxt.config 这个文件本身官方明说扩展名只能是 .js、.ts 或 .mjs 三种；pages 目录那一页的措辞是"页面是 Vue 组件，可以有 Nuxt 支持的任何有效扩展名（默认 .vue、.js、.jsx、.mjs、.ts 或 .tsx）"，也就是六项且把 .vue 放在第一位写；components 目录则说"默认情况下任何扩展名在 nuxt.config.ts 的 extensions 键里被指定的文件都被当作组件"，并给了按目录收紧的办法——在扩展形式的目录声明里写它的 extensions 键，例如 { path: '~/components', extensions: ['.vue'] }。另有一处会覆盖它：官方在自定义扫描目录那一节明确，如果你在目录项里指定了 pattern，那么 extensions 选项就没有效果，需要确保 pattern 本身匹配到你要扫描的扩展名。而文档没有定义的一件事是：同一个文件名的两种扩展同时存在时解析器选哪一个，也没有说明这个数组的顺序是否就是尝试顺序——这两点不要依赖。

categories:
  - 前端开发

tags:
  - nuxt4
  - extensions
  - 模块解析
  - 组件
  - 配置

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


这一篇的标题看起来是个小配置项，实际是三个高频问题：**`.sfc`/`.md` 这种文件为什么 Nuxt 不认？"`index.vue` 和 `index.ts` 同时存在会怎样？""我只想让它把 `.vue` 当组件，怎么收？"**

## 官方给的定义，只有一句

> `extensions`——应当由 **Nuxt 解析器**解析的扩展名。
>
> - **类型**：`array`
> - **默认值**

```json
[
  ".js",
  ".jsx",
  ".mjs",
  ".ts",
  ".tsx",
  ".vue"
]
```

**六项，全是小写带点，官方按字母序列出。** 这句话的作用域要说准：**它管的是"解析器要不要认这个扩展名"**——051 那套别名（`~`、`#build`）解析一个不带扩展名的导入时，就是靠这份清单去补后缀。

## 第一处对比：`nuxt.config` 自己只认三种

072 抄过官方在配置文件那一页的原话：

> `nuxt.config` 文件的扩展名可以是 **`.js`、`.ts` 或 `.mjs`**。

**这是三张清单里最窄的一张**：`.jsx`、`.tsx`、`.vue` 都不行。**也就是说 `extensions` 那份六项清单不适用于"配置文件叫什么名"这个问题**——两份规则各管一件事，别互相推。

## 第二处对比：pages 与 components 的措辞不一样

页面那一页（057 抄过）：

> 页面是 Vue 组件，可以有 Nuxt 支持的**任何[有效扩展名]**（**默认 `.vue`、`.js`、`.jsx`、`.mjs`、`.ts` 或 `.tsx`**）。

组件那一页（062 抄过）：

> 默认情况下，**任何扩展名在 `nuxt.config.ts` 的 `extensions` 键里被指定的文件都被当作组件。**

**两句的集合相同，但语义角色不同**：

| 位置 | 官方怎么说 | 后果 |
| --- | --- | --- |
| `pages/` | "可以有 Nuxt 支持的任何有效扩展名" | **同一份清单决定哪些文件会变成路由** |
| `components/` | "扩展名在 `extensions` 键里被指定的都被当作组件" | **这份清单决定哪些文件会被注册成组件** |

**这就解释了一个常见现象：一个 `Button.ts`（渲染函数写法）放进 `app/components/` 也会被注册**——因为 `.ts` 在默认清单里。057 那页官方给的三种写法（`.vue`、`.ts` 用 `defineComponent`、`.tsx` 用 JSX）正是这件事的示例。

## 第三处：按目录收紧，官方给的是这个键

> 如果你需要限制哪些扩展名应当被注册为组件，**你可以使用组件目录声明的扩展形式及其 `extensions` 键**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  components: [
    {
      path: '~/components',
      extensions: ['.vue'], // [!code ++]
    },
  ],
})
```

**注意这个收窄是"每个目录一条"的**——它挂在 062 讲过的那个数组式声明里，与 `path`、`prefix`、`pathPrefix` 同级。**所以你可以让一个目录只认 `.vue`、另一个目录全认。**

## 第四处：`pattern` 会让 `extensions` 失效

062 抄过那条 note，它正是为这一项准备的例外：

> 如果指定了 `pattern`，**`extensions` 选项就没有效果**，所以请确保你的 pattern 匹配上你想扫描的扩展名。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  components: [
    // ~/domains/blog/components/PostCard.vue => <PostCard />
    {
      path: '~/domains',
      pattern: '*/components/**',
      pathPrefix: false,
    },
  ],
})
```

**这条的排查价值极高：**"我在目录项里写了 `extensions: ['.vue']`，结果 `.ts` 还是被注册了"——**先看这个目录项有没有配 `pattern`**。**一旦配了 `pattern`，过滤职责整个交给 pattern，`extensions` 不再参与。**（前半句是官方原文，后半句是我对"没有效果"四个字的读法。）

## 文档没有定义的两件事

我把这一项能确证的都写完了。**还有两个问题，这一页与组件、页面那两页都没有回答，我不会替它编：**

1. **数组的顺序是不是解析时的尝试顺序？** 官方只写"应当被解析的扩展名"，没有说顺序语义。
2. **同一个文件名的两种扩展同时存在时谁赢？** 比如同目录下有 `Button.vue` 与 `Button.ts`——**文档没有任何一条规则提到这种冲突**（对照 156、159 那两节会看到的"扫描顺序"与"冲突时谁赢"的配置项 `priority`，**它们讲的是不同目录之间的注册冲突，不是同一文件名的扩展名竞争**）。

**所以这两件事的可靠做法只有一个：别制造这种歧义。** 一个组件只留一个文件、一个名字；如果你确实需要验证，跑一次 `nuxt dev` 看它注册出来的是哪个（033 的 `p` 键能浏览页面与服务端路由，组件用 DevTools 看生成物——052 讲的 Virtual Files 标签）。

## 自查

- [ ] `extensions` 的默认六项分别是什么？它管的是哪一层的事？
- [ ] 为什么 `nuxt.config.tsx` 这个名字不行？哪一份清单说了算？
- [ ] 把 `Button.ts` 放进 `app/components/`，它会被注册成组件吗？为什么？
- [ ] 想让某个目录只认 `.vue`，正确写法是什么？它是全局的还是按目录的？
- [ ] 什么情况下你写的 `extensions` 会完全没效果？
- [ ] "同名不同扩展两个文件都在"这件事，官方有没有给规则？你该怎么处理？

## 下一篇

082 讲 `ignore`、`ignorePrefix` 与 `ignoreOptions`：默认那九条 glob 里其实藏着官方对"哪些目录不该被扫"的答案（包括 `.output`、`.cache`、`.data`、`git` 与 `pnpm-store`），而 `ignorePrefix` 只管四个目录。

## 本文依据

- `api/nuxt-config`（extensions：应当由 Nuxt 解析器解析的扩展名、类型 array、默认 ["​.js",".jsx",".mjs",".ts",".tsx",".vue"]）
- `directory-structure/nuxt-config`（对照用：nuxt.config 文件的扩展名可以是 .js、.ts 或 .mjs，见 072）
- `directory-structure/app/pages`（对照用：页面是 Vue 组件可有 Nuxt 支持的任何有效扩展名（默认 .vue、.js、.jsx、.mjs、.ts 或 .tsx），并给出 index.vue/index.ts/index.tsx 三种写法示例，见 057）
- `directory-structure/app/components`（对照用：默认情况下任何扩展名在 nuxt.config.ts 的 extensions 键中的文件都被当作组件、可用目录项扩展形式的 extensions 键限制（示例 {path:'~/components',extensions:['.vue']}）、以及 note 指定 pattern 时 extensions 选项无效需确保 pattern 匹配要扫描的扩展名（示例 {path:'~/domains',pattern:'*/components/**',pathPrefix:false}），见 062）
- `api/commands/dev`（对照用：交互式面板的 p 键浏览页面与服务端路由；组件生成物可在 DevTools 的 Virtual Files 标签查看，见 033、052）
