---
url: /posts/2bd98a990aa9032d79ef1bc9bb5fb206/
title: 038 - nuxt generate 的真实身份：prerender 为 true 的 build
date: 2026-10-24T11:00:00+08:00
updated: 2026-10-24T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方对 nuxt generate 的说明里有一句把它的身份直接说破了："该命令触发把 prerender 参数设为 true 的 nuxt build 命令"。所以 generate 不是一条独立的构建管线，它的选项表也和 build 高度重合（--cwd、--logLevel、--target、--dotenv、--envName、-e/--extends、--profile），唯一区别是它自己不再提供 --prerender——因为它本身就是 prerender: true。行为上有三个后果：它会预渲染每一个路由并把结果存成纯 HTML 文件，可部署到任何静态托管；它和 nuxt build --prerender 都会自动生成 /200.html 与 /404.html 两个单页兜底页；预渲染出来的路由还会发出 _payload.json 存放构建时捕获的数据，供客户端导航时复用。另一个分支是 ssr: false：此时 generate 产出 .output/public/index.html 与普通客户端 Vue 应用一样的包，你会失去预渲染带来的 SEO 好处，官方因此建议改用 <ClientOnly> 包住那些确实不能服务端渲染的部分。

categories:
  - 前端开发

tags:
  - nuxt4
  - 预渲染
  - generate
  - 静态托管
  - 命令行

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

很多人以为 `generate` 是另一套构建，因为它名字里有 generate。官方只用一句话就把这个误解消掉了。

官方给的身份，它就是 build。`generate` 命令预渲染你应用的每一个路由，并把结果存成纯 HTML 文件，你可以把它们部署到任何静态托管服务上。该命令触发把 `prerender` 参数设为 `true` 的 `nuxt build` 命令。

第二句是关键。它意味着两件事。

不存在 generate 专属的构建流程。你要理解 generate 的行为，看 037 那条 `build` 就够了。

`nuxt build --prerender` 与 `nuxt generate` 是同一件事的两种写法。官方在两处分别这么描述，build 页给了 `--prerender`，构建 Nuxt 并预渲染静态路由，generate 页说它触发的就是这个。

把两张选项表并排放，结论立刻出来。`generate` 的用法行是这样。

```bash [Terminal]
npx nuxt generate [ROOTDIR] [--cwd=<directory>] [--logLevel=<silent|info|verbose>] [--target=<target>] [--dotenv=<path>...] [--envName=<environment>] [-e, --extends=<layer-name>...] [--profile=<verbose>]
```

对着 037 的 `build` 选项表，差别只有一处。`--cwd` 两边都有，`--logLevel` 都有，`--target` 都有，`--dotenv` 都有且都可重复靠后优先，`--envName` 都有，`-e, --extends` 都有，`--profile` 都有，唯独 `--prerender`，build 有，generate 没有。

generate 没有 `--prerender`，因为它自己就是 `prerender: true`。这张对照表的意义是，以后你在 build 上学到的规则，比如 037 规则二，`--prerender` 始终把 target 设为 `static`，自动适用于 generate，因为走的正是那条路。

顺带一条对 037 的补充核对，build 页明确说本命令把 NODE_ENV 设为 production，而 generate 页没有给这条 note。我在这一篇里不替它补结论，你要在构建期区分环境，用官方在两节都给了的 `--envName`，构建时默认 `production`。

后果一，它会生成两个兜底页。官方在部署页的静态托管一节写得很直白，静态站点生成配合 `ssr: true` 在构建时预渲染你应用的路由，这是运行 `nuxt generate` 时的默认行为。它还会生成 `/200.html` 与 `/404.html` 两个单页应用兜底页，它们可以在客户端渲染动态路由或 404 错误，不过你可能需要在你的静态宿主上配置这一项。

同页后面 Static Fallback Pages 一节把两个页面的用途分开说了，`200.html` 是单页应用兜底，当你要客户端路由来处理这个 URL 时，配置你的宿主为未匹配的路由提供它。`404.html` 是未找到兜底，对于应当保持 404 状态的路由，配置你的宿主提供它。

以及谁负责生成它们，`nuxt generate` 与 `nuxt build --prerender` 会自动生成这些文件。如果你用 `nuxt build` 配合路由规则来预渲染选定的路由，请显式加上兜底页。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    '/200.html': { prerender: true },
  },
})
```

这里有两个容易漏的动作。

一，生成不等于生效。官方两处都在提醒宿主侧要配置，一些服务商使用 `200.html`，一些使用 `404.html`，一些允许你两个都配。部署后请检查你的托管服务商的静态兜底或重写设置。

二，默认的兜底页是空壳。官方给的这句要读完，默认情况下，两个兜底页都是空壳。把 `experimental.prerenderErrorPages` 设为真，可在构建时把你的 `error.vue` 服务端渲染进 `404.html`。

症状是我 404 页在本地正常、上线变成一片白，那不是路由坏了，是 `404.html` 从来就没渲染过内容。

后果二，预渲染会留下 `_payload.json`。官方在部署页紧接着一句，预渲染的路由还会发出 `_payload.json` 文件，其中存放构建时捕获的数据，Nuxt 在客户端导航期间复用它们。

这和 028 讲的 payload 是同一个思路的两种落地，`useFetch` 在服务端取到的数据，预渲染时被存成一份可以直接取用的文件，于是客户端换页不必重新请求。延伸阅读在预渲染页的 payload extraction 一节，038 不展开。

这也解释了预渲染站点最常见的一句疑问，为什么我改了后台数据、页面没变。因为那份数据是构建时捕获的，它要等到下一次构建或预渲染才会更新，这条推论的依据就是上面那句构建时捕获。

另一条分支，`ssr: false` 时的 generate。官方明确给了第二个选项，而且它的态度很值得照搬。或者，你可以用 `ssr: false` 预渲染你的站点，静态单页应用。这会产生 HTML 页面，其中 `<div id="__nuxt"></div>` 是空的，那里本该渲染你的 Vue 应用。你会失去预渲染站点带来的大部分 SEO 好处，因此建议改为使用 `<ClientOnly>` 包住站点里确实不能被服务端渲染的部分，如果有。

同页的 Client-side Only Rendering 一节补了产物形态，如果你不想预渲染路由，另一种使用静态托管的方式是在 `nuxt.config` 文件里把 `ssr` 属性设为 `false`。`nuxt generate` 命令届时会输出一个 `.output/public/index.html` 入口，以及 JavaScript 包，就像一个经典客户端 Vue.js 应用。

把这两段合起来读，`generate` 在不同 `ssr` 下的产物差异是结构性的。`ssr: true`，默认，产物是每个路由一个预渲染好的 HTML 加两个兜底页加 `_payload.json`，首屏 HTML 里有内容。`ssr: false`，产物是 `.output/public/index.html` 加 JS 包，首屏 HTML 里没有内容，空 `__nuxt` div。

我明明生成了 HTML，为什么抓取到的是空白页，`ssr: false` 就是这个答案。而官方的建议不是那就别用静态托管，是保留 SSR、只把不能服务端渲染的那一小块包起来。

## 自查

- [ ] 一句话说明 `nuxt generate` 与 `nuxt build --prerender` 的关系。
- [ ] `generate` 的选项表比 `build` 少了哪一个？为什么？
- [ ] `200.html` 与 `404.html` 分别该在什么情况下由宿主提供？默认它们是渲染好的错误页吗？
- [ ] `_payload.json` 里放的是什么？它为什么会解释「改了数据页面没变」？
- [ ] 什么配置会让 `generate` 产出空的 `<div id="__nuxt"></div>`？官方对这种情况的建议是什么？

## 下一篇

039 把 preview 与 start 的开关摊全，并和 034 那张 dev 选项表对照——你会看到 dev 有而 preview 没有的那一整排（`--open`、`--https`、`--inspect`）意味着什么。
