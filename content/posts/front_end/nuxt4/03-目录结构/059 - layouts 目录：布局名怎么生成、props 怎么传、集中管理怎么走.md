---
url: /posts/95efea14eed378ee2e3842155bb5f093/
title: 059 - layouts 目录：布局名怎么生成、props 怎么传、集中管理怎么走
date: 2026-10-31T11:00:00+08:00
updated: 2026-10-31T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：布局文件的名字由它的路径与文件名共同决定，官方给了两张对照表并明确"重复的路径段会被去掉"：~/layouts/desktop/default.vue 得到 desktop-default、~/layouts/desktop-base/base.vue 得到 desktop-base、~/layouts/desktop/index.vue 得到 desktop；官方紧接着建议让文件名与它生成的名字一致（desktop/DesktopDefault.vue → desktop-default）。启用布局的方式有三种：在页面用 definePageMeta 设 layout 属性、设 <NuxtLayout> 的 name prop、或在路由规则里设 appLayout（官方标注 v4.3，并说明它适合在配置里集中管理布局、或用于那些没有对应页面组件的路由如 catch-all）；没有指定时使用 app/layouts/default.vue。运行时可用 setPageLayout 切换，配合 definePageMeta({layout:false}) 还可以在页面内部自己放 <NuxtLayout>，此时官方要求它不能是页面的根元素（或禁用过渡）。v4.4 起可以给布局传 props：definePageMeta 里把 layout 写成 { name, props } 对象、或 setPageLayout('panel', { sidebar: true, title: 'Dashboard' })，并且官方说明 props 会依据布局自己的 defineProps 完整定型、能在编辑器里拿到自动补全与类型检查。另外官方在页首用 tip 指出：出于最佳性能，此目录里的组件会在使用时通过异步导入自动加载。

categories:
  - 前端开发

tags:
  - nuxt4
  - 布局
  - 命名规则
  - props
  - 目录结构

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

026 那篇我讲了要不要用布局这个判断。这一篇讲剩下的三件工程事，布局名是怎么从文件路径算出来的、props 怎么传进去、以及怎么在配置里集中管理而不是逐页写。

性能那条 tip，布局是异步导入的。官方在这一节第一句就给了这个性质，很多人读布局文档会跳过它，出于最佳性能，放置在此目录里的组件会在被使用时通过异步导入自动加载。

这条把 012 讲过的分包落到了布局上，布局不是被打包进主块、每个页面都带上它，而是用到时才取。所以把很重的公共 UI 放进布局这件事，代价比放在 `app.vue` 里小，054 那条 note 说 `app.vue` 里的 JS 与 CSS 会全局进每个页面，而布局这一层是按需加载的。

布局名生成规则，两张表。官方先给了那条规范化，布局名会被规范化为 kebab-case，所以 `someLayout` 变成 `some-layout`。

然后给了嵌套路径的生成规则，原文是，如果你的布局在嵌套目录里，布局的名字将基于它自己的路径目录与文件名，重复的段会被去掉。

这张表是这一节的核心，我完整抄下来。`~/layouts/desktop/default.vue` 得到 `desktop-default`，`~/layouts/desktop-base/base.vue` 得到 `desktop-base`，`~/layouts/desktop/index.vue` 得到 `desktop`。

对照着看，三条规则各出现在哪一行。

第一行，目录名加文件名，`default` 不消失，因为它在子目录里是具体那个布局。

第二行，`desktop-base/base.vue` 得到 `desktop-base`，文件名 `base` 与目录后缀 `base` 重复，重复段被去掉，这正是官方那句 duplicate segments being removed 的落点。

第三行，`index.vue` 被吃掉，只剩目录名 `desktop`。

然后官方给了一条与这套规则配套的建议，为了清晰，我们建议让布局的文件名与它自己的名字匹配。`~/layouts/desktop/DesktopDefault.vue` 得到 `desktop-default`，`~/layouts/desktop-base/DesktopBase.vue` 得到 `desktop-base`，`~/layouts/desktop/Desktop.vue` 得到 `desktop`。

两张表并排读，差别只有一个，文件名要不要长得像结果名。官方推荐后者，`DesktopDefault.vue` 生成的名字一眼就能预期，而 `default.vue` 在不同目录里生成 `desktop-default` 还是 `desktop`，取决于它在不在子目录。

三种指定方式，逐页、全局 prop、集中配置。官方给的三种，在页面里用 `definePageMeta` 设置 `layout` 属性，设置 `<NuxtLayout>` 的 `name` prop，以及在路由规则里设置 `appLayout` 属性。

外加默认那条，如果没有指定布局，会使用 `app/layouts/default.vue`。

前两种 026 讲过。这里补第三种，它的适用理由官方写得很具体。

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  routeRules: {
    // Set layout for specific route
    '/admin': { appLayout: 'admin' },
    // Set layout for multiple routes
    '/dashboard/**': { appLayout: 'dashboard' },
    // Disable layout for a route
    '/landing': { appLayout: false },
  },
})
```

官方的 tip 是，当你想在配置里集中管理布局而不是在每个页面文件里做，或者你需要把布局应用到那些没有对应页面组件的路由，例如可能匹配许多路径的 catch-all 页面，时，这很有用。

后面那半句是第三种方式独有的能力，`/dashboard/**` 这种通配、以及 catch-all 匹配到的众多路径，逐页写 `definePageMeta` 覆盖不到，catch-all 只有一个文件、而它匹配很多路径。

另外注意示例里 `appLayout: false` 这一行，它能关掉布局，这与下一节的 `layout: false` 是同一语义的两种入口。

运行时切换与页面自己套布局。

`setPageLayout`，官方原话，你也可以用 `setPageLayout` 助手来动态切换布局。

```vue [app/pages/index.vue]
<script setup lang="ts">
function enableCustomLayout () {
  setPageLayout('custom')
}

definePageMeta({
  layout: false,
})
</script>

<template>
  <div>
    <button @click="enableCustomLayout">
      Update layout
    </button>
  </div>
</template>
```

注意这两行的搭配，`definePageMeta({ layout: false })` 加事件里 `setPageLayout('custom')`。官方这个示例本身就说明，先声明这个页面默认不套布局，再在运行时决定套哪个。

页面内部自己用 `<NuxtLayout>`，官方原话，如果你在使用页面，你可以设 `layout: false` 然后在页面内部使用 `<NuxtLayout>` 组件来取得完全控制。

官方给的这一对文件里，父页面把 `<NuxtLayout name="custom">` 放在一个 `<div>` 里面。

```vue [app/pages/index.vue]
<script setup lang="ts">
definePageMeta({
  layout: false,
})
</script>

<template>
  <div>
    <NuxtLayout name="custom">
      <template #header>
        Some header template content.
      </template>

      The rest of the page
    </NuxtLayout>
  </div>
</template>
```

```vue [app/layouts/custom.vue]
<template>
  <div>
    <header>
      <slot name="header">
        Default header content
      </slot>
    </header>
    <main>
      <slot />
    </main>
  </div>
</template>
```

而这条要求官方用 important 写着，如果你在你的页面里使用 `<NuxtLayout>`，请确保它不是根元素，或者禁用布局与页面过渡。

这和 026 那条布局必须有单一根元素、且根不能是 `<slot />` 是两条不同层面的规则，一条管布局文件本身，一条管布局组件在页面里的位置。上面这对示例正好同时满足两条，`custom.vue` 的根是 `<div>`，而 `index.vue` 里的 `<NuxtLayout>` 被包在 `<div>` 里。

v4.4 起，给布局传 props。官方给的第一种写法是对象形式的 `layout`。

```vue [app/pages/dashboard.vue]
<script setup lang="ts">
definePageMeta({
  layout: {
    name: 'panel',
    props: {
      sidebar: true,
      title: 'Dashboard',
    },
  },
})
</script>
```

```vue [app/layouts/panel.vue]
<script setup lang="ts">
const props = defineProps<{
  sidebar?: boolean
  title?: string
}>()
</script>

<template>
  <div>
    <aside v-if="sidebar">
      Sidebar
    </aside>
    <main>
      <h1>{{ title }}</h1>
      <slot />
    </main>
  </div>
</template>
```

官方的 tip 是，props 会依据你布局的 `defineProps` 完整定型。你会在你的编辑器里拿到自动补全与类型检查。

这一条把类型闭环做上了，布局文件里 `defineProps` 写什么类型，页面里那个 `props` 对象就只能填什么，填错键名会在编辑器里被标出来。

第二种写法是运行时那条，你也可以在用 `setPageLayout` 动态切换布局时传 props。

```ts
setPageLayout('panel', { sidebar: true, title: 'Dashboard' })
```

## 自查

- [ ] `~/layouts/desktop/index.vue` 与 `~/layouts/desktop/default.vue` 生成的布局名分别是什么？
- [ ] `desktop-base/base.vue` 为什么得到 `desktop-base`？官方哪句话解释了这个？
- [ ] `appLayout` 与 `definePageMeta({layout})` 相比，官方指出它多能覆盖什么场景？
- [ ] 在页面内部使用 `<NuxtLayout>` 时，官方的那条 important 要求是什么？
- [ ] 布局的 props 类型是从哪里来的？两种传法分别怎么写？
- [ ] 为什么「很重的公共 UI 放布局」比放 `app.vue` 代价小？

## 下一篇

060 讲 `app/middleware/`：官方在这一节列了三种中间件、「只扫描顶层文件与子目录的 index」这条目录规则，以及那条最容易被撞的禁令——中间件里不能用 `useRoute()`，要用函数参数或 `useRouter()` 的 `currentRoute` 代替。
