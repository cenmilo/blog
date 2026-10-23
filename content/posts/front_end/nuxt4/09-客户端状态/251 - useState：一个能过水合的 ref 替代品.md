---
url: /posts/295218c3985a6eefb7b5acf29c4f984d/
title: 251 - useState：一个能过水合的 ref 替代品
date: 2027-01-03T11:00:00+08:00
updated: 2027-01-03T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇只看 `useState` 这一节写了什么。**签名两条重载**：`useState<T> (init?: () => T | Ref<T>): Ref<T>` 与 `useState<T> (key: string, init?: () => T | Ref<T>): Ref<T>`——**返回值永远是一个 `Ref`**，所以 `counter.value` 才是值。两个参数的官方说明我逐字抄：`key` 是 "A unique key ensuring that data fetching is properly de-duplicated across requests. If you do not provide a key, then a key that is unique to the file and line number of the instance of `useState` will be generated for you."（注意这句用的是**取数那套"去重"措辞**）；`init` 是 "A function that provides initial value for the state when not initiated. **This function can also return a `Ref`.**"；`T` 标了 "(typescript only) Specify the type of state"。最短用法是 `const count = useState('counter', () => Math.round(Math.random() * 100))`。三条限制：important——"Because the data inside `useState` will be serialized to JSON, it is important that it does not contain anything that cannot be serialized, such as classes, functions or symbols."；warning——"`useState` is a reserved function name transformed by the compiler, so you should not name your own function `useState`."；以及一个可选优化：**"If you don't need your state to be deeply reactive, you can combine `useState` with `shallowRef`. This can improve performance when your state contains large objects and arrays."**，官方还给了判据注释 `// isShallow(state) === true`。

categories:
  - 前端开发

tags:
  - nuxt4
  - useState
  - 响应式
  - shallowRef
  - 状态

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


250 讲了"为什么需要它"，这一篇讲"它到底是什么形状"。先记一句定性：官方在状态管理那一节写的是 **"`useState` is an SSR-friendly `ref` replacement"**——它是 `ref` 的替换品，不是 store。

## 两条重载与它给的两样东西

```ts [Signature]
export function useState<T> (init?: () => T | Ref<T>): Ref<T>
export function useState<T> (key: string, init?: () => T | Ref<T>): Ref<T>
```

**两条的返回都是 `Ref<T>`**。这是最容易忘的事：`useState` 给你的不是值，是一个 ref，所以模板里能直接用、`<script setup>` 里要 `.value`（和 214 讲的取数返回值同一套规则）。

最短示例（官方 Usage）：

```ts
// Create a reactive state and set default value
const count = useState('counter', () => Math.round(Math.random() * 100))
```

参数三条原文：

| 参数 | 官方说明 |
| --- | --- |
| `key` | "A unique key ensuring that data fetching is properly de-duplicated across requests. If you do not provide a key, then a key that is unique to the file and line number of the instance of `useState` will be generated for you." |
| `init` | "A function that provides initial value for the state when not initiated. This function can also return a `Ref`." |
| `T` | "(typescript only) Specify the type of state" |

两条我要单独说：

1. **`key` 的说明写的是"data fetching is properly de-duplicated"**——这行措辞看起来是从 `useAsyncData` 那边复制过来的（215 抄过几乎同一句）。我读到的有效信息仍然只有一句：**不给 key 就按文件名+行号生成**。至于为什么状态页要说"取数去重"，文档没解释，我不替它圆。
2. **`init` 可以返回 `Ref`**。这不只是类型细节——它正是下面 `shallowRef` 那一档能写的原因。

## init 是工厂，而且只在"还没初始化时"求值

`init` 那句里的限定词值得抠："provides initial value **when not initiated**"。**已经存在的那一格不会被你的 `init` 覆盖**——这和 218 讲的 `default`（"before async resolves"）不同层：`default` 是"数据还没回来时显示什么"，`init` 是"这一格第一次被创建时是什么"。

官方 Advanced Usage 里有一个把 init 与别的 composable 组合的真实形状：

```ts
export const useLocale = () => {
  return useState<string>('locale', () => useDefaultLocale().value)
}

export const useDefaultLocale = (fallback = 'en-US') => {
  const locale = ref(fallback)
  if (import.meta.server) {
    const reqLocale = useRequestHeaders()['accept-language']?.split(',')[0]
    if (reqLocale) {
      locale.value = reqLocale
    }
  } else if (import.meta.client) {
    const navLang = navigator.language
    if (navLang) {
      locale.value = navLang
    }
  }
  return locale
}
```

这段我读到的三件事：

1. **两侧读法不同**：服务端从 `accept-language` 头读（223 讲过这个头只在服务端有值，浏览器里 `useRequestHeaders` 返回空对象），客户端读 `navigator.language`。
2. 官方自己用 `import.meta.server` / `import.meta.client` 分流，说明**这就是它推荐的写法**（185/232 也引过同一招）。
3. `useState` 的 init 里调别的 composable 是合法的，因为 init 在 setup 的同步流里被调用（对照 233 的上下文规则）。

## 大对象状态：配 shallowRef

整节原文（含判据注释）：

> If you don't need your state to be deeply reactive, you can combine `useState` with [`shallowRef`]. **This can improve performance when your state contains large objects and arrays.**

```ts
const state = useState('my-shallow-state', () => shallowRef({ deep: 'not reactive' }))
// isShallow(state) === true
```

这条与 213 里 `deep` 那个选项是同一个动机的两个入口：`useAsyncData` 的 `deep` 默认 `false`（"for improved performance (shallow ref object)"），`useState` 侧则要靠你主动包 `shallowRef`。

**但要注意那句注释的含义**：`isShallow(state)` 为 `true` 意味着内部对象的属性变化不会触发更新——**只有整体替换 `state.value = {...}` 才有效**。官方在这里没写这个后果（"深响应关闭后要怎么改数据"它一个字没提），所以我把它列成用前必须自己验证的一项：改一层嵌套属性，看界面有没有动。

## 三条限制与一条排错

**序列化边界**（important，与 250 同句）：

> Because the data inside `useState` will be serialized to JSON, it is important that it does not contain anything that cannot be serialized, such as classes, functions or symbols.

**保留名**（warning）："`useState` is a reserved function name transformed by the compiler, so you should not name your own function `useState`." —— 和 `useAsyncData`/`useFetch`/`useLazyFetch` 是同一族规则（212/219/225），**自己封的包装函数要起别名**（250 的 `useColor` 就是示范）。

**排错**：Troubleshooting 小节标题就是一个真实报错——

> ### `Cannot stringify arbitrary non-POJOs`
>
> This error occurs when you try to store a non-serializable payload with `useState`, such as class instances.
>
> If you want to store class instances with `useState` that are not supported by Nuxt, you can use `definePayloadPlugin` to add a custom serializer and deserializer for your classes.

**POJO 指的是"普通 JS 对象"**（Plain Old JavaScript Object），这句报错的意思就是：它认不出这个对象的类型。解法官方只给了一条路：**给 payload 注册自定义 reducer/reviver**（`definePayloadPlugin`，指向 `use-nuxt-app` 的 payload 一节），而不是"换个容器放"。

## 与其它 key 寻址工具的关系

我按各页覆盖范围原文对齐，**别把两组当成同一套**：

| 工具 | 官方点名的作用对象 |
| --- | --- |
| `useState` | 自己创建的那一格状态 |
| `clearNuxtState` | "Delete the cached state of `useState`." |
| `useNuxtData` | `useAsyncData`、`useLazyAsyncData`、`useFetch`、`useLazyFetch`（**四个取数 composable**） |
| `clearNuxtData` | "cached data, error status and pending promises of useAsyncData and useFetch" |

也就是说：**状态用 `useState` + `clearNuxtState` 这一对，取数用另一对**。常见的一种误用是拿 `clearNuxtData()` 想清 `useState` 的值——按这两节的文字，它管的不是同一个东西（252 细说清理）。

## 自查

1. `useState` 的返回是什么？怎么读它的值？（答：`Ref<T>`；`<script setup>` 里要 `.value`，模板里自动解包。）
2. `init` 什么时候不生效？（答：那一格已经被初始化过；官方说明是"provides initial value for the state when not initiated"。）
3. 存了类实例导致 `Cannot stringify arbitrary non-POJOs`，官方给的解法是什么？（答：用 `definePayloadPlugin` 给 payload 加自定义序列化/反序列化。）

## 下一篇

252 - useState 的 key 规则与序列化约束
