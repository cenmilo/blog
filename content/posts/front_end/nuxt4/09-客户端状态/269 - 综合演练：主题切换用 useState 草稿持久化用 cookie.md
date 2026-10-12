---
url: /posts/5bb0d5b76741f7d5b39c7fad6e6b313e/
title: 综合演练：主题切换用 useState 草稿持久化用 cookie
date: 2027-01-09T11:00:00+08:00
updated: 2027-01-09T11:00:00+08:00
author: cmdragon

summary:
  摘要：把上一张决策表跑一遍。两个最常写的功能：一是主题/配色，只在当前这一次访问里跨组件共享，用 useState 封一个 useColor；二是写了一半的草稿，刷新页面还得在，用 useCookie，并且要理解它的 watch 与"push 不会自动写回"那个坑。两个例子都贴着官方 state-management 与 use-cookie 两页的示例来，把可复用组合式的写法、序列化和读写时机一并落地。

categories:
  - 前端开发

tags:
  - nuxt4
  - useState
  - useCookie
  - 组合式
  - 演练
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

决策表看着抽象，落到两个具体功能上立刻清楚。挑这两个是因为它们刚好踩在表的两条不同分支上：主题切换是"活这一次访问"，草稿是"活过刷新"。

## 演练一：主题色，用 useState 封一个组合式

需求是"当前颜色"这种状态，多个组件都要读同一个值、改一处到处跟着变，但不需要跨刷新。官方 state-management 的 "Shared State" 给的正是这个形状——把 `useState` 包进一个自动导入的组合式：

```ts [app/composables/states.ts]
export const useColor = () => useState<string>('color', () => 'pink')
```

任何组件里 `useColor()` 拿到的都是同一个 key（`'color'`）下的那份状态，注释也点明了 `const color = useColor() // Same as useState('color')`。

为什么这里用 `useState` 而不是模块级 `ref`：`useState` 的值服务端渲染时会进 payload、水合时接得上，而模块级 `ref` 会在服务端把不同请求串到一起（268 那条 warning）。主题这种"首屏就要有正确颜色、点了要立刻变"的，正是 `useState` 的主场。

进阶一点，官方给的多语言例子演示了"两端各自取初值、但共用一个 `useState`"的写法骨架：

```ts
export const useLocale = () => {
  return useState<string>('locale', () => useDefaultLocale().value)
}
```

`useDefaultLocale()` 在服务端读 `accept-language` 请求头、在客户端读 `navigator.language`，最后结果都汇进 `useState('locale', ...)`。这套"init 函数里按 `import.meta.server`/`client` 分流、但容器统一用 useState"是官方推荐的组合式写法，主题、深浅色模式都可以照它套。

一个必须记得的约束（264）：`useState` 里的东西要能被 JSON 序列化——主题名、颜色字符串、布尔开关都行，别往里塞函数或类实例。

## 演练二：草稿，用 useCookie 但要看清 watch

需求换了：用户在文本框写了半篇东西，手滑刷新了，内容还得在。"活过刷新" = 跨请求 = cookie（268）。最直觉的写法：

```vue
<script setup lang="ts">
const draft = useCookie('draft', { default: () => '' })
</script>

<template>
  <textarea v-model="draft" placeholder="草稿"></textarea>
</template>
```

`v-model="draft"` 每次输入都会给 `draft.value` 赋新字符串——这是**对 ref 本身赋值**，配合 `useCookie` 默认的 `watch: true`（深监听），每次赋值都会写回 cookie。刷新后 `useCookie('draft')` 从 cookie 读回，内容还在。这条路对"值是基本类型、每次整体赋值"是成立的。

坑在**值是对象/数组、你想往里 push 的时候**。官方 use-cookie 页专门给了这个反例，注释直接写着不会写回：

```vue
<script setup lang="ts">
const list = useCookie('list', {
  default: () => [],
  watch: 'shallow',
})

function add() {
  list.value?.push(Math.round(Math.random() * 1000))
  // list cookie won't be updated with this change
}

function save() {
  // the actual `list` cookie will be updated
  list.value &&= [...list.value]
}
</script>
```

原因在选项表那句：`watch: 'shallow'` 只监听**顶层属性**变化，`push` 改的是数组内部、没换 `list.value` 这个引用，所以浅监听不触发写回。`save()` 那句 `list.value &&= [...list.value]` 是**用一个新数组替换整个 value**——顶层引用变了，监听命中，cookie 才写。

所以草稿要是个对象（比如 `{ title, body, tags: [] }`），你有两种确定写法：

- 改嵌套字段时，改完**整体重新赋值**一次（`draft.value = { ...draft.value }`），别指望 `draft.value.title = 'x'` 在浅监听下自动写回。
- 或者干脆别关深监听：默认 `watch: true` 会深监听对象内部变化、自动写回；代价是大对象上监听开销。官方对深监听还有一句 `::note` 级别提醒——大对象建议配 `shallowRef` 之类降开销（251 也提过 shallow）。

## 两者放一起的分界

| | 主题/配色 | 草稿 |
| --- | --- | --- |
| 要活多久 | 这一次页面加载 | 跨刷新、跨请求 |
| 容器 | `useState('color')` | `useCookie('draft')` |
| 写回时机 | 赋值即改响应式、进 payload | 赋值触发 watch 才写 cookie |
| 典型坑 | 塞了不可序列化对象 | 浅监听下 push 不写回，要整体重赋 |

一句话对照：`useState` 你几乎不用操心"什么时候落盘"，它本来就只为这一次加载服务；`useCookie` 多一层"值→cookie"的写入时机，`watch` 档位（`true`/`'shallow'`/`false`）决定什么改动算"写回信号"。

## 自查

- 主题/配色：封成 `useColor = () => useState('color', () => 'pink')`，别用模块级 ref；存进去的要是可序列化值。
- 草稿：基本类型直接 `v-model` 就写回；对象/数组要么整体重赋值、要么保持深监听。
- `watch: 'shallow'` 时记住那句官方注释——`push` 不更新 cookie。
- 别把密钥/敏感草稿明文放 cookie：cookie 在浏览器可读，敏感内容要么走加密封装（259），要么压根别存前端。

## 下一篇

P09 客户端状态到此收尾。下一篇 270 开 P10 插件与中间件：先从"插件到底解决什么问题、和组件里的组合式有什么分工"讲起。

## 本文依据

- State Management（`getting-started/state-management`）："Shared State" 的 `export const useColor = () => useState<string>('color', () => 'pink')` 与 `// Same as useState('color')`；"Advanced Usage" 里 `useLocale = () => useState<string>('locale', () => useDefaultLocale().value)`、`useDefaultLocale` 内按 `import.meta.server`（`accept-language`）/`import.meta.client`（`navigator.language`）分流。
- useCookie（`api/composables/use-cookie`）：Writable Cookies 示例 `list.value?.push(...)` 配注释 "list cookie won't be updated with this change"，以及 `save()` 里 `list.value &&= [...list.value]` 配注释 "the actual `list` cookie will be updated"；选项表 `watch` 的 `true`/`'shallow'`/`false` 与 shallow="data changes for only top level properties"；`default` 选项。
- useState（`api/composables/use-state`）：序列化为 JSON 的 important；`shallowRef` 组合段（大对象性能）。
