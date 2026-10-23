---
url: /posts/f6fe6459234353927cd4b13fcdc6d199/
title: 808 - 错误边界（2）：给这一块套上 NuxtErrorBoundary，抛错只吃掉这一块
date: 2026-10-31T10:00:00+08:00
updated: 2026-10-31T10:00:00+08:00
author: cmdragon

summary:
  摘要：上一站量到"客户端渲染期抛错不白屏、也没痕迹"，这一站把同一位探针套进 `<NuxtErrorBoundary>`，读它换来了什么：那块位置上的默认内容被 `#error` 槽顶掉（回退看得见）、`@error` 把"红过一次"记成界面上一个可读的数字、而边界外面一位没少——正文、旁边那块、导航全在，也没换成整屏错误页。还有一条反直觉的读数：加了边界之后控制台里反而一条错误都没有，因为错误被边界接住就不再往上冒。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 错误边界
  - 错误处理
  - 会员区
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 这一档页面上多了三行读数

访问 `http://localhost:3100/docs/getting-started?islandLab=boundary`，正文下方是这样：

```text
这块在边界外面 LAB_BOUNDARY_KEEP_808
边界里红过：0 次
边界收到的错：还没有
[在边界里挂上那一位会炸的组件]
边界里的默认槽 LAB_BOUNDARY_DEFAULT_808
```

点那颗按钮，最后一行换成：

```text
错误块已出现 LAB_ERROR_SLOT_808：LAB_BOOM_807：渲染期抛错（没有边界包住它）
[清掉它]
```

同一位组件、同一次抛错，与上一站唯一的区别是它外面多了一层 `<NuxtErrorBoundary>`。这一站就读这三件事：**那块位置有没有留下看得见的回退**、**"红过一次"有没有变成可读的数**、**代价是不是零**。

## 官方给的依据就三句

讲 `<NuxtErrorBoundary>` 那一节：「handles client-side errors happening in its default slot」；Events 段：「`@error`: Event emitted when the default slot of the component throws an error」；Slots 段：「`#error`: Specify a fallback content to display in case of error」，示例是 `<template #error="{ error, clearError }">`。

## 仓库里加了什么

页面 `app/pages/docs/[slug].vue` 新增一档，探针组件用的还是 807 那一位 `app/components/LabBoomPanel.vue`：

```vue
<script setup lang="ts">
const 挂上会炸的探针 = ref(false)
const 边界报错次数 = ref(0)
const 边界错误文本 = ref('')

function 记边界报错 (错误: unknown) {
  边界报错次数.value += 1
  边界错误文本.value = 错误 instanceof Error ? `${错误.name}: ${错误.message}` : String(错误)
}
</script>

<template v-if="实验档 === 'boundary'">
  <p data-testid="boundary-keep">这块在边界外面 LAB_BOUNDARY_KEEP_808</p>
  <p data-testid="boundary-count">边界里红过：{{ 边界报错次数 }} 次</p>
  <p data-testid="boundary-error">边界收到的错：{{ 边界错误文本 || '还没有' }}</p>
  <button data-testid="boundary-mount" type="button" @click="挂上会炸的探针 = true">在边界里挂上那一位会炸的组件</button>
  <NuxtErrorBoundary @error="记边界报错">
    <p v-if="!挂上会炸的探针" data-testid="boundary-default">边界里的默认槽 LAB_BOUNDARY_DEFAULT_808</p>
    <LabBoomPanel v-else where="render" />
    <template #error="{ error: 收到的错误, clearError: 清掉它 }">
      <p data-testid="boundary-slot">错误块已出现 LAB_ERROR_SLOT_808：{{ … }}</p>
      <button data-testid="boundary-clear" type="button" @click="清掉它">清掉它</button>
    </template>
  </NuxtErrorBoundary>
</template>
```

槽名保持官方那两个（`error`、`clearError`），但**本地要起别名**：这一页上面已经有一位叫 `error` 的（取数那一位），同名会被 `vue/no-template-shadow` 报出来，读代码的人也要猜一下这里说的是哪一个错。判据在 `test/e2e/error-boundary.spec.ts`，四条。

## 四条读数，原样打印

```text
[边界·没炸] {"默认槽":1,"错误槽":0,"边界外那块":1,"正文":1,"计数":" 边界里红过：0 次 ","错误文本":" 边界收到的错：还没有"}
[边界·炸了] {"点前默认槽":1,"默认槽":0,"错误槽":1,"边界外那块":1,"正文":1,"页眉":1,"计数":" 边界里红过：1 次 ","整页错误页":0,"错误槽文本":" 错误块已出现 LAB_ERROR_SLOT_808：LAB_BOOM_807：渲染期抛错（没有边界包住它）","控制台条数":0,"控制台":[]}
[边界·事件] {"计数":" 边界里红过：1 次 ","错误行":" 边界收到的错：Error: LAB_BOOM_807：渲染期抛错（没有边界包住它）","疑似堆栈":false}
[边界·对照] {"错误槽":0,"计数位":0,"整页错误页":0,"抛错提示":1}
```

`4 passed`；与 807 那六条一起跑是 `10 passed`；端到端全量连跑两趟各 `211 passed／57 skipped`。

## 换来了两件事，代价是零

**一件：那块位置留下了可见的回退。** 点之前 `默认槽:1`、`错误槽:0`；点之后 `默认槽:0`、`错误槽:1`，而且错误槽里的文本带着那次抛错的消息。**二件：红过一次变成了数。** `边界里红过：0 次 → 1 次`，`边界收到的错` 那一行把 `名字: 消息` 两位都落在了页面上。

**代价那一栏读出来是零**：`边界外那块:1`、`正文:1`、`页眉:1`、`整页错误页:0`。边界只吃它包住的这一位——这就是标题里那句"抛错只吃掉这一块"，而它成立的前提是官方那句范围限定：**只管它默认槽里的客户端错误**。

对照档（同一位组件、不套边界那次）读回来是 `[边界·对照] {"错误槽":0,"计数位":0,"整页错误页":0,"抛错提示":1}`：回退块和计数位**根本不存在**，页面照样活着。也就是说这两件可读性完全来自那层包装，不是来自"错误变小了"。

包的位置也有讲究，读回来的这几位顺带说明了为什么：边界要包在**那块数据外面**，不是包在整页外面。要是把它套在 `<main>` 外面，判据读回来同样会是"错误槽出现、正文与导航全在"吗——不会，那时正文与导航都在错误槽这一层之下，跟着一起被顶掉；这一站没跑那一档（形状从两条官方原文就能定：管的是"它默认槽里"的错误，默认槽有多大、被顶掉的范围就有多大）。也就是说，"只吃掉这一块"这句能不能成立，取决于你圈的那一块有多大，与边界本身无关。

## 反直觉的那条：加了边界，控制台反而一条都没有

`[边界·炸了]` 里 `控制台条数: 0`。而 807 那一站，同样一次渲染期抛错，控制台里明明有一条 `Error: LAB_BOOM_807…`。

差别就在这层包装本身：边界内部用的就是 Vue 的 `onErrorCaptured`，**它接住之后这个错误就不再往上冒**了——上一层没人再收到它，Nuxt 那条把错误记到控制台的路径自然走不到。所以：

- **别把控制台当"有没有红过"的依据**：加了边界之后控制台变干净，恰恰是因为错误被处理掉了；
- 要上报（送进监控、写进日志），只能挂在 `@error` 上，或者用下一节要讲的 `error` 槽那两个参数——这是 810 那一步的题。

顺手钉了一条安全口径：`疑似堆栈: false`。页面上只放 `名字: 消息`，`错误行` 里读不到 `at …` 或 `文件.js:行号` 这种堆栈帧（807 那轮控制台里是能看到完整调用位置的）。把堆栈渲染到用户界面上不是这件组件该做的事。

## 那条基线判据，这一站按预告改了

`console-baseline.spec.ts` 里数 `app/` 下有几个 `.vue` 真把边界用起来的那条，807 时读到 `boundaryHits: 0`，判据写的是"一处都没有"。这一站加了一处，它必须红——**红的正是现实变了，不是判据写错**。改法是把"没有"换成钉住现在这个形状，而不是把断言删掉：

```text
[baseline error boundary] {"vueFiles":22,"boundaryHits":1,"命中边界":["pages/docs/[slug].vue"],"原文命中边界":2,"mainHits":2,"hasErrorPage":true}
```

两位都有出处：`boundaryHits: 1` 且命中的就是文档详情页（注释里提到名字的两位仍然算在 `原文命中边界: 2` 里，这就是上一站先剥注释的原因）；`mainHits: 2` 那位非零对照组还在，说明扫描真读了文件。判据里另外钉了一条方向：**会员区那两个页面这时还不该有边界**（`pages/console/index.vue`、`pages/me.vue`），把边界接进产品页面是这条线收尾之前要补的改动，别提前放宽。

## 排错表

| 想知道的 | 读哪一位 | 只读一处会错在哪 |
| --- | --- | --- |
| 边界有没有生效 | 默认槽与错误槽的此消彼长 | 只看错误槽出现，可能只是它本来就渲了 |
| 有没有只吃这一块 | 边界外那块、正文、导航各数一次 | 不看会误以为"没白屏"就等于"没影响" |
| 这次抛错去哪了 | `@error` 计数 ＋ 控制台条数 | 只看控制台会得出"没红过"，恰恰反了 |
| 页面上有没有泄漏细节 | 那一行里有没有堆栈帧的形状 | 只截文本看不出差 |
| 现实与判据谁变了 | `boundaryHits` 与命中文件名 | 只有数字会说不出是哪一页改的 |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/error-boundary.spec.ts
```

预期 `4 passed`。手点版：进 `?islandLab=boundary`，开 Console 面板盯着，点`在边界里挂上那一位会炸的组件`——页面下方那一段换成错误块，上面三行读数同时变成"红过：1 次"和那条错误文本，而 **Console 里一条错误都没有**。再点`清掉它`会发生什么（默认槽回来、还是再红一次）就是下一站 809 要量的那句「you'll need to ensure that the error is fully resolved first or the error slot will just be rendered a second time」。

## 本文依据

- 官方：讲 `<NuxtErrorBoundary>` 的那一节（范围那句、Events 段的 `@error`、Slots 段的 `#error` 与 `error`／`clearError`、`useTemplateRef` 那段）；讲错误处理那一节（Render Error in Component 段那句「On client-side, it will prevent the error from bubbling up to the top level, and will render the `#error` slot instead」，以及"`error = null` 会重渲默认槽、没真解决就会再渲一次错误槽"那句）
- 仓库：`app/pages/docs/[slug].vue`（`boundary` 那一档：三行读数、一颗挂载按钮、`记边界报错` 处理器）、`app/components/LabBoomPanel.vue`（807 立的探针，这一站复用）、`app/pages/docs/index.vue`（`lab-boundary` 入口）、`test/e2e/error-boundary.spec.ts`（新增 4 条）、`test/e2e/console-baseline.spec.ts`（基线③从"一处都没有"改成钉住"只在实验位这一页有、会员区还没有"）
- 上一批：[807 - 错误边界（1）：没有边界时，一次抛错会掀掉多大一片](/posts/16746246ced418e27431e5af1797fbb6/)
- 下一篇：错误边界（3）——`#error` 槽里那两样东西：`error` 的形状与 `clearError` 之后到底重渲哪一遍（本篇不预先给链接，地址要等那一篇的文件名定下来才能算）
