---
url: /posts/1807d2a20861d6386d8defaad26617e6/
title: 168 - preloadComponents 与 prefetchComponents：官方说两者实现目前一样
date: 2026-12-07T09:00:00+08:00
updated: 2026-12-07T09:00:00+08:00
author: cmdragon

summary:
  摘要：这对函数名看着是"提前加载"与"预热"两种策略，但官方给了一句会让人改变用法的说明："**Current implementation behaves exactly the same as `preloadComponents` by preloading components instead of just prefetching we are working to improve this behavior.**"。两节各自的说法确实是两件事。`preloadComponents`："**Preloading components loads components that your page will need very soon, which you want to start loading early in rendering lifecycle. This ensures they are available earlier and are less likely to block the page's render, improving performance.**"；`prefetchComponents`："**Prefetching component downloads the code in the background, this is based on the assumption that the component will likely be used for rendering, enabling the component to load instantly if and when the user requests it. The component is downloaded and cached for anticipated future use without the user making an explicit request for it.**"——一个绑定"本页渲染早期"，一个绑定"未来可能用到"。可用范围两节写的是同一句："**Use `preloadComponents` to manually preload individual components **that have been registered globally** in your Nuxt app. **By default Nuxt registers these as async components. You must use the Pascal-cased version of the component name.**"，`prefetchComponents` 那段除动词外完全同构。写法都是 `await preloadComponents('MyGlobalComponent')` / `await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])`（prefetch 同款两行），服务端都无效果（"**On server, `preloadComponents` will have no effect.**" / "**On server, `prefetchComponents` will have no effect.**"），源码都在 `packages/nuxt/src/app/composables/preload.ts`。它和 145 的 `preloadRouteComponents`（"**loads the components of a given route**"）一起构成三个入口，而 `addComponent` 上的 `prefetch`/`preload` 是第四个、完全不同层的东西（"used **in production** to configure how components with **Lazy prefix** are handled by **webpack via its magic comments**"）。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组件预热
  - 性能
  - preload
  - prefetch

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


按字面意思很容易这么用这两个：`preload` 当"马上要用"、`prefetch` 当"闲着预热"。读完第二节的 note 才会发现，**在当前的实现里这两种写法做的是同一件事**。这一篇把两节原文并排放，再把它和另外两个同名概念分干净。

## 两个定义确实写的是两件事

`preloadComponents` 页（定位句 "Nuxt provides utilities to give you control over preloading components."）：

> Preloading components loads components that **your page will need very soon**, which you want to start loading **early in rendering lifecycle**. This ensures they are available earlier and **are less likely to block the page's render**, improving performance.

`prefetchComponents` 页（定位句 "Nuxt provides utilities to give you control over prefetching components."）：

> Prefetching component downloads the code **in the background**, this is based on the assumption that **the component will likely be used for rendering**, enabling the component to load instantly **if and when the user requests it**. The component is **downloaded and cached for anticipated future use** without the user making an explicit request for it.

并排看差别全在时间参照上：

| | `preload` | `prefetch` |
| --- | --- | --- |
| 官方时间词 | "early in rendering lifecycle" | "in the background" / "anticipated future use" |
| 服务谁 | **这一页的渲染** | **用户接下来的操作** |
| 收益表述 | 别挡住本页渲染 | 用户要的时候已经下好了 |

**语义上这是两个不同的调度决策**（一个紧迫、一个乐观），官方文档也照这两个意思各写了一段。

## 但实现上官方说它们现在一样

第二页那条 note 是全篇最该先看的一行：

> Current implementation **behaves exactly the same as `preloadComponents`** by preloading components instead of just prefetching **we are working to improve this behavior.**

这句话里有两个信息，值得分开处理：

1. **"behaves exactly the same"** 是当下的事实陈述——所以现在在两者之间做选择，**得不到任何行为差别**。
2. **"we are working to improve this behavior"** 说明官方认为这是**待修正的**，两者最终应当不同。

我的处理原则因此定了：**今天按 `preloadComponents` 写**（它的名字与当前实际行为一致），需要"未来语义"时再换名；**不要把代码组织在"`prefetch` 更省带宽"这种预期上**——那还没实现。这类"文档承诺与实现有落差"的地方，按实现走、把承诺记下来更省事。

## 共同前提：只对"已全局注册的组件"有效

两节给的可用范围是同一句话（`preloadComponents` 版）：

> Use `preloadComponents` to manually preload individual components **that have been registered globally** in your Nuxt app. By default Nuxt registers these as async components. **You must use the Pascal-cased version of the component name.**

`prefetchComponents` 那段除了动词换成 "manually prefetch"，其余逐字相同。

两个约束由此推出来，都和前面的篇目直接相关：

| 约束 | 后果 | 关联 |
| --- | --- | --- |
| 必须是**全局注册**的组件 | 普通自动导入的组件（用到才注册）**不在这两个函数的操作范围里** | 159：全局注册的两条路径（`components/global/` 目录、`.global.vue` 后缀）与代价"each global component is rendered in a separate chunk" |
| 参数是**组件名**、且要 **PascalCase** | 传路径、传 kebab 名都不会是有效入参 | 157：自动导入名是路径拼出来的 Pascal 形态 |

**这两条合起来其实给出了一种搭配读法**：159 里我把"全局注册"当成有代价的选项，而这两个函数是**唯一需要这份代价才成立的能力**——你能按名字点一个全局组件的 chunk 去预热。反过来，如果没全局注册，就没有可点名的对象，也就用不上它们。

## 写法：await，可以传数组

```ts
await preloadComponents('MyGlobalComponent')

await preloadComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])
```

```ts
await prefetchComponents('MyGlobalComponent')

await prefetchComponents(['MyGlobalComponent1', 'MyGlobalComponent2'])
```

两节示例形状完全对称，且都带 `await`——**这一点和 145 的 `preloadRouteComponents` 是一组对照**：那个官方示例**故意不 await**（"we don't await this async function, to avoid blocking rendering this component's setup function"），而这两页的示例是 await 的。差别在意图上：`preloadComponents` 服务的正是"本页马上要用"，等它反而合理。

服务端行为两节也各写了一句，内容一致：

> On server, `preloadComponents` will have no effect.
> On server, `prefetchComponents` will have no effect.

源码位置同一个文件：`packages/nuxt/src/app/composables/preload.ts`（145 那个也在这里）。

## 四个都叫 prefetch/preload 的东西，层次不同

这是我一开始最容易混的地方，一次列清：

| 名字 | 层 | 作用对象 | 官方原话关键词 |
| --- | --- | --- | --- |
| `preloadRouteComponents('/path')` | 运行时 API | **一条路由**的组件 | "loads the components of a given route that the user might navigate to in future"（145） |
| `preloadComponents('Name')` | 运行时 API | **已全局注册的组件** | "loads components that your page will need very soon" |
| `prefetchComponents('Name')` | 运行时 API | 同上 | 实现同上（那条 note） |
| `<NuxtLink>` 的 `prefetch` prop | 自动机制 | 视口内链接的 **middleware、layouts、payloads** | "When enabled will prefetch middleware, layouts and payloads (when using payloadExtraction) of links in the viewport."（147） |
| `addComponent` / `addComponentsDir` 的 `prefetch`、`preload` | **构建期字段** | 带 `Lazy` 前缀的组件如何被 webpack 处理 | "used **in production** to configure how components with Lazy prefix are handled by **webpack via its magic comments**"（161） |

**最后一行是"同名不同层"的典型**：它是打进构建配置的提示，与运行时调用毫无关系；而第四行是导航预取，与组件名无关。四行里只有中间三行是本篇的两个函数加 145 那一个。

## 什么时候真会用到这两个

| 场景 | 该用 |
| --- | --- |
| 下一跳是另一条路由（登录成功进 dashboard） | `preloadRouteComponents`（145），**不 await** |
| 本页某块内容稍后会出现（点开折叠面板才用的全局组件） | `preloadComponents`，await 它 |
| 想让链接指向的页面提前就绪 | 什么都不用写，`<NuxtLink>` 的自动预取已在做（147） |
| 想让 `Lazy` 组件在构建里带 magic comment | `addComponentsDir({ prefetch, preload })`，注意只对 webpack 那句话负责（161） |
| 组件没全局注册 | 前两个函数都点不到名，先回到 159 看是否真的值得全局化 |

我的取向：**这两个函数的使用频率远低于 `preloadRouteComponents`**——因为"按组件名预热"要求你先接受全局注册的代价，而大部分项目更想要的是"按路由切代码"。

## 自查

1. 两节定义句里各自的时间参照词是什么？用一句话说清语义差别。
2. 官方关于两者实现的那条 note 全文是什么？它包含哪两个信息，你分别怎么落地？
3. 这两个函数只能操作哪一类组件？为什么这跟 159 的代价是配套出现的？
4. 参数形态有什么硬要求？两处一致的地方还有哪些（服务端、示例形状、源文件）？
5. 为什么 `preloadComponents` 的示例 await、而 `preloadRouteComponents` 的示例故意不 await？
6. 表里五个 prefetch/preload 概念分别属于哪一层？哪两个与组件名无关？
7. `addComponent` 的 `prefetch` 字段与 `prefetchComponents` 的区别是什么？官方给它原话里的三个限定词是什么？

## 下一篇

169 转入岛屿：什么时候真的该用服务端组件、客户端导航时每个岛屿要发一次请求这件事的代价，以及官方对"不适合用岛屿"的判断句。
