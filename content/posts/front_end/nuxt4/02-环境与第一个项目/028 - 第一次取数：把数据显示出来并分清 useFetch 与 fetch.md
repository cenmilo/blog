---
url: /posts/5cf7d5094c9eca53c204cf2f54382e24/
title: 第一次取数：把数据显示出来，并分清 useFetch 与 $fetch
date: 2026-10-21T10:00:00+08:00
updated: 2026-10-21T10:00:00+08:00
author: cmdragon

summary:
  摘要：Nuxt 给了三样取数工具：$fetch 是最简单的网络请求方式；useFetch 是包在 $fetch 外面的一层，作用是让数据在同构渲染里只取一次；useAsyncData 与 useFetch 类似但控制粒度更细。为什么要有后两个？官方说得很直白：如果在组件的 setup 里直接用 $fetch，数据可能被取两遍——服务端渲染 HTML 时一遍，浏览器水合时又一遍，这会造成水合问题、拖慢可交互时间并让行为难以预测。useFetch 和 useAsyncData 会把服务端取到的数据放进 payload 转发给浏览器，从而避免重复请求；而 $fetch 没有这个机制，官方因此建议它用于纯浏览器侧发起的请求（例如事件触发的提交）。两者的返回值也不同：useFetch 返回 data、refresh/execute、clear、error、status 五个成员，其中 data/error/status 是要用 .value 访问的 Vue ref。

categories:
  - 前端开发

tags:
  - nuxt4
  - 取数
  - usefetch
  - dollarfetch
  - 入门

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


页面能显示了，下一步就是让它显示**不是写死在文件里的数据**。这是全系列第一个"取数"动作，我先只做三件事：把数据显示出来、知道为什么不用 `fetch`、知道点击提交时该用哪个函数。

## 官方给的是三样工具，不是一样

> Nuxt 自带两个组合式函数和一个内置库，用于在浏览器或服务端环境中取数：`useFetch`、`useAsyncData` 和 `$fetch`。

官方紧接着给的"一句话版本"，我原样抄下来，这是后面 20 多篇取数文章的地图：

> - `$fetch` 是发起网络请求的最简单方式。
> - `useFetch` 是包在 `$fetch` 外面的一层，它让数据在同构渲染（universal rendering）中**只取一次**。
> - `useAsyncData` 与 `useFetch` 类似，但提供更细粒度的控制。

三行的差别全在"**取一次**"这三个字上。为什么这件事需要框架来保证？下一节。

## 为什么不直接在 setup 里用 `$fetch`

这一节是全篇的核心，官方的论证是这样的：

> Nuxt 是一个可以在服务端和客户端两个环境运行同构（isomorphic/universal）代码的框架。如果在 Vue 组件的 setup 函数里用 `$fetch` 取数，**这可能导致数据被取两遍**：一次在服务端（为了渲染 HTML），一次在客户端（当 HTML 被水合时）。这会造成水合问题、增加达到可交互状态的时间，并导致不可预测的行为。
>
> `useFetch` 和 `useAsyncData` 通过保证「如果服务端已经发起了这次 API 调用，数据会被转发到客户端的 payload 里」来解决这个问题。

对照 013 那张"同一份代码跑两个地方"的图，这段话就完全说得通了：setup 函数本来就要跑两遍，**普通的 `fetch` 式请求不认识这个事实，所以它就跟着跑两遍**。

那个 payload 是什么？官方给的位置是：

> payload 是一个 JavaScript 对象，可以通过 `useNuxtApp().payload` 访问。在客户端，它被用来避免重复取同样的数据（当代码在浏览器中于水合期间执行时）。

还有一个很实用的提示：

> 用 Nuxt DevTools 的 **Payload tab** 查看这些数据。

**这一条建议你现在就用一次**：起 `npm run dev`，打开 DevTools，看 Payload 标签页里躺着的正是你 `useFetch` 拿到的东西。看到它，"数据被转发到客户端"就不再是抽象说法了。

## 最小写法：显示一个数

官方给的最短可用形态：

```vue [app/app.vue]
<script setup lang="ts">
const { data: count } = await useFetch('/api/count')
</script>

<template>
  <p>Page visits: {{ count }}</p>
</template>
```

三点要说明：

1. **`/api/count` 这个路径哪来的？** 文档这里的示例假使你已有一个接口。这个接口自己怎么写（放在 `server/api/` 里、用 `defineEventHandler` 导出）属于服务端目录那几篇的内容，这里先借用文档的路径写法，把注意力放在取数这一侧。
2. **`data: count` 是重命名。** `useFetch` 返回的成员叫 `data`，这里把它改名成 `count`，模板里就能写 `{{ count }}`。
3. **`await`。** 这一行先 `await` 再往下走。它的后果官方专门有一节讲（下面第 5 节），这里先照抄。

## `$fetch` 该用在哪：点击事件里

官方给的完整对照例子，恰好就是"一个显示 + 一个提交"：

```vue [app/app.vue]
<script setup lang="ts">
const { data } = await useFetch('/api/data')

async function handleFormSubmit () {
  const res = await $fetch('/api/submit', {
    method: 'POST',
    body: {
      // My form data
    },
  })
}
</script>

<template>
  <div v-if="data == undefined">
    No data
  </div>
  <div v-else>
    <form @submit="handleFormSubmit">
      <!-- form input tags -->
    </form>
  </div>
</template>
```

注意 `data == undefined` 那句——页面在数据到达前先显示"No data"，这是**首屏取数**要处理的加载态。而提交表单这一侧用的是 `$fetch`，带 `method: 'POST'` 和 `body`。

官方对这个搭配的结论：

> 在上面的例子里，`useFetch` 确保了请求发生在服务端并被正确地转发到浏览器。`$fetch` **没有这种机制**，当请求纯粹由浏览器发起时，它是更好的选择。

还有一条正式的警告：

> 小心：只使用 `$fetch` 不会提供**网络调用去重与导航阻止**。建议把 `$fetch` 用于客户端交互（基于事件的），或者与 `useAsyncData` 组合用于获取组件的初始数据。

把这两段合起来，就是你现在唯一需要记住的分工：

| 你要做的事 | 用哪个 |
| --- | --- |
| 首屏要显示的数据（进页面就得有） | `useFetch` |
| 用户点击/提交触发的动作（POST 之类） | `$fetch` |
| 第三方查询层之类的取数 | `useAsyncData` 包住它 |

## 返回值不止 `data`

官方列了全量（`useFetch` 与 `useAsyncData` 的返回值一致）：

> - `data`：传入的异步函数的结果。
> - `refresh`/`execute`：可以用来刷新 `handler` 函数返回的数据的函数。
> - `clear`：一个函数，把 `data` 置为 `undefined`（或 `options.default()` 提供的值）、把 `error` 置为 `undefined`、把 `status` 置为 `idle`，并把当前挂起的请求标记为取消。
> - `error`：取数失败时的错误对象。
> - `status`：一个字符串，表示数据请求的状态（`"idle"`、`"pending"`、`"success"`、`"error"`）。

外加一条容易踩的实现细节：

> `data`、`error` 和 `status` 是 Vue ref，在 `<script setup>` 里要通过 `.value` 访问。

**所以模板里直接写 `{{ count }}`（不用 `.value`），但在 `<script setup>` 里要用 `count.value`。** 这个差别是 Vue 的模板自动解包，不是你写错了。

第一次写取数，你只需要真正常用这三个：`data`、`status`、`error`。`refresh`/`execute`/`clear` 等表单提交后要重新拉数据时再回来查（那部分在 214 逐条讲）。

## 那个 `await` 到底改变了什么

官方专门写了一节"A note on `await`"，因为它推翻了很多人对 `await` 的直觉：

> 文档里的示例通常会对 `useFetch` 和 `useAsyncData` 的调用加 `await`，但这并不总是必需的。
>
> `await` **不会**改变服务端渲染出的 HTML。在服务端渲染期间，无论有没有 `await`，Nuxt 都会先等请求完成再序列化页面（底层用的是 `<Suspense>` 和 `onServerPrefetch`），所以发到浏览器的始终是填好的结果。

那 `await` 改的是什么？

> - **有 `await`**：执行会暂停直到数据就绪，所以这行之后的代码可以直接依赖 `data` 已经有值。在客户端导航时，这会阻止导航直到数据返回：用户留在当前页（可选地配合 `<NuxtLoadingIndicator>`），然后落到一个数据完整的页面。这是默认行为。
> - **没有 `await`**：执行立即继续，请求在后台跑，所以 `data` 一开始是它的默认值，请求完成后才填上。在客户端导航时，这意味着导航立即发生，而加载态和错误态要你自己处理（通常通过返回的 `status` 和 `error`）。

官方的收尾判断我觉得值得每个初学者看一遍：

> 两种做法都不是普遍更好；正确的选择取决于你想要的那个路由上的体验。

顺便一个背景机制，官方也是在这里给的：

> Nuxt 底层使用 Vue 的 `<Suspense>` 组件，来阻止在每份异步数据可用之前的导航。

**这就是"默认会挡住跳转"的来源。** 你想让它不挡，用 `lazy`；你想等，就 `await`。

有一条官方警告现在不用记住，但**撞上时会省你半天**：

> `await` 和 `lazy` 是相互独立的，在客户端 `await` 一个 `lazy` 调用不会有你期待的效果。如果你 `await` 一个 `lazy` 调用（例如 `await useLazyFetch(...)` 或 `await useFetch(..., { lazy: true })`），它照常阻塞服务端渲染，但在**客户端导航**时那个 `await` 会立即返回、不等请求。`data` 在 `await` 之后仍是默认值，你必须通过 `status` 处理加载态。如果你真的想让导航等数据，请**去掉 `lazy`**，而不是指望 `await`。

## 自查

- [ ] 你能说出"在 setup 里用 `$fetch` 取首屏数据"为什么会取两遍吗？（提示：013）
- [ ] 服务端取到的数据通过什么机制到达浏览器？你在 DevTools 的哪个标签里能看到它？
- [ ] 点击提交按钮发 POST，该用哪个函数？为什么？
- [ ] `status` 有哪四个取值？`data` 在 `<script setup>` 里怎么访问？
- [ ] `await` 改变的是服务端 HTML，还是客户端导航行为？

## 下一篇

029 把项目推向"能上线"的那一步：`npm run build` 产出什么、`nuxt preview` 在本地跑的是哪一份东西，以及为什么 preview 和真正的生产启动命令不是一回事。

## 本文依据

- `getting-started/data-fetching`（开头三样工具与 in a nutshell 三条；The Need for useFetch and useAsyncData：同构框架、setup 里用 $fetch 可能取两遍（服务端渲染 HTML 一次、客户端水合一次）造成水合问题/增加可交互时间/行为不可预测，useFetch 与 useAsyncData 保证服务端调用产生的数据被转发进 payload，payload 是 JavaScript 对象可通过 useNuxtApp().payload 访问并在客户端水合期间用于避免重复取数，tip 用 Nuxt DevTools 的 Payload tab 查看；app.vue 示例 const { data } = await useFetch('/api/data') 与 handleFormSubmit 中 $fetch('/api/submit',{method:'POST',body})、模板 v-if data == undefined 显示 No data、"useFetch 确保请求在服务端发生并正确转发到浏览器；$fetch 无此机制，当请求仅由浏览器发起时是更好的选项"；warning 只用 $fetch 不提供网络调用去重与导航阻止，建议 $fetch 用于客户端（事件式）交互或与 useAsyncData 组合获取初始组件数据；Suspense：Nuxt 底层用 Vue 的 <Suspense> 阻止每份异步数据可用前的导航，tip 可加 <NuxtLoadingIndicator>；A note on await：await 不改变服务端渲染 HTML（服务端渲染期间无论哪种都会等请求完成再序列化，底层是 <Suspense> 与 onServerPrefetch），有 await 时客户端导航被阻塞直到数据返回并落到完整页面（默认行为），无 await 时执行立即继续、data 先为默认值、导航立即发生且需自行用 status 与 error 处理；两种做法并非普遍更好，取决于该路由想要的体验；warning await 与 lazy 独立，await 一个 lazy 调用在客户端导航时立即返回、data 仍为默认值、需用 status 处理加载态，想让导航等数据应去掉 lazy；Return Values 五项 data/refresh-execute/clear/error/status 与 status 的 idle、pending、success、error 四值，note data/error/status 是 Vue ref 需 .value；useFetch 示例 const { data: count } = await useFetch('/api/count') 配 <p>Page visits: {{ count }}</p>）
