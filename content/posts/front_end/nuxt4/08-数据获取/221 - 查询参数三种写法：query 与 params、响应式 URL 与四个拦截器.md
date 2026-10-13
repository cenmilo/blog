---
url: /posts/6ec3d01739eff967d00f918600217aa4/
title: 221 - 查询参数三种写法：query 与 params、响应式 URL 与四个拦截器
date: 2026-12-24T11:00:00+08:00
updated: 2026-12-24T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：把参数放进一次取数，文档给了三条形状不同的路，加上拦截器那一组四个钩子。① `query` 选项："Using the `query` option, you can add search parameters to your query. This option is extended from unjs/ofetch and is **using unjs/ufo to create the URL. Objects are automatically stringified.**"，示例是 `query: { param1, param2: 'value2' }`（`param1` 是个 ref），官方直接给出结果 URL："The above example results in `https://api.nuxt.com/modules?param1=value1&param2=value2`."；选项表里 `params` 行写 "Alias for `query`."。② **响应式 URL**：`url` 参数类型是 `string | Request | Ref<string | Request> | (() => string | Request)`，官方 "Reactive Keys" 示例写的是 `useFetch(() => \`/api/posts/${id.value}\`)`，注释 "When the route changes and id updates, the data will be automatically refetched"。③ **响应式选项自动重取**：`query: { q: searchQuery }` 之后改 `searchQuery.value`，官方注释是 "triggers a refetch: /api/search?q=new%20search"（注意值被编码过），不想重取就 `watch: false`。拦截器那组四个钩子是 `onRequest`、`onRequestError`、`onResponse`、`onResponseError`，官方在 `onRequest` 那行挂了一句依赖提示："note that this relies on **ofetch >= 1.4.0** - you may need to refresh your lockfile"，而它的 `onResponse` 示例里写了 `localStorage.setItem`——这一处我在文末单独提出风险，因为服务端那一趟也会跑拦截器。

categories:
  - 前端开发

tags:
  - nuxt4
  - query
  - 拦截器
  - ofetch
  - useFetch

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


先说一个判断：这三条路的差别不在"能不能带参数"，而在**参数变了以后谁负责重发请求**。选错的表现通常是两种：要么筛选条件改了数据不动，要么每改一个字符打一次接口。

## 路一：query 选项（以及它的别名 params）

官方那段说明（含实现来源）：

> Using the `query` option, you can add search parameters to your query. This option is extended from [unjs/ofetch] and is using [unjs/ufo] to create the URL. **Objects are automatically stringified.**

```ts [app/pages/index.vue]
const param1 = ref('value1')
const { data, status, error, refresh } = await useFetch('/api/modules', {
  query: { param1, param2: 'value2' },
})
```

> The above example results in `https://api.nuxt.com/modules?param1=value1&param2=value2`.

三件事一次说清了：**值是直接拼进 query 的**、**ref 可以原样放进对象**（不用 `.value`）、**对象会被自动序列化**。另外 `params` 行写的是 "Alias for `query`."，`query` 行反过来标了 "Alias: `params`."——两个名字等价。

那 `query` 里的 ref 变了会怎样？走的是"选项响应式"那条（见路三），官方在同一个例子里给了编码后的结果 URL。

## 路二：整段 URL 自己拼（可以是函数）

`url` 参数的类型支持四种形态：`string | Request | Ref<string | Request> | (() => string | Request)`。官方 "Reactive Keys and Shared State" 一节给的是函数形态：

```vue [app/pages/[id].vue]
<script setup lang="ts">
const route = useRoute()
const id = computed(() => route.params.id)

// When the route changes and id updates, the data will be automatically refetched
const { data: post } = await useFetch(() => `/api/posts/${id.value}`)
</script>
```

这条路适合**参数是路径的一部分**（`/api/posts/12`）而不是查询串。注意它和 `key` 的连带关系：URL 变了，自动生成的 key 也跟着变（`key` 行写明生成依据包含 URL），所以这是"换一格数据"；而路一/路三是在同一格里重取（215 那张表讲的正是这个分界）。

## 路三：响应式选项，改值就重发

官方 Reactive Fetch Options 一节整段：

> Fetch options can be provided as reactive, supporting `computed`, `ref` and computed getters. When a reactive fetch option is updated it will trigger a refetch using the updated resolved reactive value.

```ts [app/pages/index.vue]
const searchQuery = ref('initial')
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
})
// triggers a refetch: /api/search?q=new%20search
searchQuery.value = 'new search'
```

那行注释我特意看了两遍，因为它透露了两件事：**`new search` 里的空格被编码成 `%20`**（编码由 ufo 那层做，官方只给结果形态），以及**赋值本身就触发重发**，不需要额外 `watch`。

想按住在输入框里连续打字的场景，官方给的是关掉自动监听：

```ts
const { data } = await useFetch('/api/search', {
  query: { q: searchQuery },
  watch: false,
})
// does not trigger a refetch
searchQuery.value = 'new search'
```

但**"输入防抖"这件事文档里没有写**（只有 `watch: false` 这一刀切的关法）。我的做法是关掉自动重取、再用一个防抖后的 ref 手动调 `refresh()`——**这半句是我自己的方案，官方没有这种配方**，读到时请当成待验证的写法。

## 四个拦截器：官方的完整示例

`useFetch` 页直接给了四个钩子一起用的形状（原文照抄，含那句依赖提示）：

```ts [app/pages/index.vue]
const { data, status, error, refresh, clear } = await useFetch('/api/auth/login', {
  onRequest ({ request, options }) {
    // Set the request headers
    // note that this relies on ofetch >= 1.4.0 - you may need to refresh your lockfile
    options.headers.set('Authorization', '...')
  },
  onRequestError ({ request, options, error }) {
    // Handle the request errors
  },
  onResponse ({ request, response, options }) {
    // Process the response data
    localStorage.setItem('token', response._data.token)
  },
  onResponseError ({ request, response, options }) {
    // Handle the response errors
  },
})
```

从这张代码里能直接读出的四点：

| 钩子 | 入参 | 官方注释给的用途 |
| --- | --- | --- |
| `onRequest` | `{ request, options }` | 改请求头，写法是 `options.headers.set(...)`（**Headers 对象，不是普通对象**） |
| `onRequestError` | `{ request, options, error }` | 处理请求阶段的错误 |
| `onResponse` | `{ request, response, options }` | 处理响应数据，示例里取的是 `response._data` |
| `onResponseError` | `{ request, response, options }` | 处理响应错误 |

那句依赖提示值得单独记住：**"this relies on ofetch >= 1.4.0 - you may need to refresh your lockfile"**——`options.headers` 是 Headers 实例这件事取决于装到的 ofetch 版本，锁文件里钉着老版本时行为就不一样。

## 这个示例里有一处我不敢照抄

`onResponse` 那行写的是 `localStorage.setItem('token', response._data.token)`。问题在于**服务端渲染那一趟同样会跑这次请求**，而浏览器存储在服务端不存在。官方文档在别处对这件事的态度很统一：

- 取 DOM 或浏览器 API 的时机，`client-only` 那一节的说法是组件"rendered only after being mounted"（198 抄过）。
- hydration 那一节把"服务端使用浏览器 API"列为第一类成因，示例正是 `localStorage.getItem('theme')` 直接写在 `<script setup>` 里会错配（199 抄过），并点名 `localStorage` 不存在于服务端。

**所以我不会把这个 `localStorage` 调用原样放进拦截器**——文档在这里给的是拦截器形状，不是"在服务端也能写 localStorage"的保证。要落这个例子，我会把它挪到只在客户端跑的位置（例如点击处理函数里、或者 `import.meta.client` 分支中），并用一次服务端渲染实测确认没有报错。这一步属于我按文档其他页的规则做的修正，不是官方这段代码本身的意思。

## 自查

1. `query: { param1 }` 里放的是 ref，要不要写 `.value`？（答：不要，官方示例直接把 ref 放进对象，并说明"Objects are automatically stringified."）
2. 输入框里每改一个字符都会打接口，文档给的开关是什么？（答：`watch: false`；防抖方案文档没有写。）
3. 官方拦截器示例里那行 `localStorage.setItem` 为什么不能照抄？（答：服务端那一趟也会执行拦截器，而 `localStorage` 在服务器不存在——hydration 那一节正是把它列为第一类成因。）

## 下一篇

222 - 服务端那次请求要不要带头：useRequestFetch 与 baseURL 选项

## 本文依据

- `api/composables/use-fetch`：query 段落两句与 `{ param1, param2: 'value2' }` 示例及其结果 URL；选项表 `query` 行（"Alias: `params`."）与 `params` 行（"Alias for `query`."）、`body` 行（"Objects are automatically stringified."）；Reactive Keys and Shared State 一节（函数 URL 示例与注释）；Reactive Fetch Options 一节（两句 + `searchQuery` 示例 + `%20` 编码结果 + `watch: false` 示例）；拦截器示例全文（四个钩子、`options.headers.set`、`response._data.token`、ofetch >= 1.4.0 那句注释）；`URL` 参数类型四种形态。
- `api/components/client-only`：默认插槽"tree-shaken out of the server build"与"rendered only after being mounted"（198 已逐句引）。
- `guide/best-practices/hydration`：Browser-only APIs 成因（`localStorage` 不存在于服务端）与 `useCookie` 解法（199 已逐句引）。
