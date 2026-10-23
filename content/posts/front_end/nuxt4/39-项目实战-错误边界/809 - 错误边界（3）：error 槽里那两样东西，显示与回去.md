---
url: /posts/de847f25f637a01d9f4f6ce7449e27d8/
title: 809 - 错误边界（3）：error 槽里那两样东西，显示与回去
date: 2026-10-31T11:00:00+08:00
updated: 2026-10-31T11:00:00+08:00
author: cmdragon

summary:
  摘要：上一站给那块位置套上错误边界，这一站读 `#error` 槽交出来的两样东西。一样是 `error`：抛的是 Error 时它不可枚举那几位挡住了细节，落到页面上只有「名字加消息」五十来个字符；抛的是自己拼的普通对象时同一行插值会走另一条分支，把 `status`、`code`、`detail.field` 一个个缩进排好显示出来。另一样是 `clearError`：它只把边界自己那一位状态置回 null，默认槽于是重新渲一遍，原因没改掉就再红一次，连点三次就红三次。

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

## 你刚点下"清掉它"，然后页面又红了

上一站那个实验位（`http://localhost:3100/docs/getting-started?islandLab=boundary`）现在长这样，多出两行读数和一颗新按钮：

```text
这块在边界外面 LAB_BOUNDARY_KEEP_808
边界里红过：0 次
边界收到的错：还没有
错误形状：还没有
边界里那位探针现在：还是会抛
[在边界里挂上那一位会炸的组件] [把抛错的原因改掉（这位组件留在原地，只是不再抛）]
边界里的默认槽 LAB_BOUNDARY_DEFAULT_808
第二个边界里红过：0 次
[在第二个边界里抛一个普通对象]
第二个边界的默认槽 LAB_PLAIN_DEFAULT_809
```

挂上探针之后它换成错误块，错误块里除了上一站那行取消息的文本，这次还照官方示例多写了一行原样插值：

```text
错误块已出现 LAB_ERROR_SLOT_808：LAB_BOOM_807：渲染期抛错（没有边界包住它）
原文插值 LAB_ERROR_RAW_809：Error: LAB_BOOM_807：渲染期抛错（没有边界包住它）
[清掉它]
```

这一站就盯着两件事：**那行原样插值到底把什么东西放到了用户屏幕上**，以及**`清掉它` 之后页面上到底发生了什么**。第二个问题的答案决定这颗按钮能不能放进真产品里。

## 官方给的两句话

讲错误处理那一节的 Render Error in Component 段，先说范围，再给这一站的判据原文：

> The `#error` slot will receive `error` as a prop. (If you set `error = null` it will trigger re-rendering the default slot; you'll need to ensure that the error is fully resolved first or the error slot will just be rendered a second time.)

括号里那句是分号的：**清错误会让默认槽重渲，所以原因没真解决就会再渲一次错误槽**。同段还有一句范围话：「On client-side, it will prevent the error from bubbling up to the top level, and will render the `#error` slot instead」。示例给的是 `<template #error="{ error, clearError }">`，里面直接写 `{{ error }}` 加一颗 `clearError` 按钮——也就是这两样东西，一个是显示，一个是回去。

## 仓库里加了什么

`app/pages/docs/[slug].vue` 那一档里加了三处。第一处是把错误对象的形状记下来，供界面读：

```ts
const 边界错误形状 = ref('')

function 记边界报错 (错误: unknown) {
  边界报错次数.value += 1
  边界错误文本.value = 错误 instanceof Error ? `${错误.name}: ${错误.message}` : String(错误)
  边界错误形状.value = JSON.stringify({
    是Error实例: 错误 instanceof Error,
    名字: 错误 instanceof Error ? 错误.name : `不是对象（${typeof 错误}）`,
    消息: 错误 instanceof Error ? 错误.message : String(错误),
    有stack: 错误 instanceof Error && typeof 错误.stack === 'string',
    可枚举键: typeof 错误 === 'object' && 错误 !== null ? Object.keys(错误) : [],
  })
}
```

第二处是一颗只改原因的按钮，加一位布尔：

```ts
const 探针还会炸 = ref(true)
```

```vue
<button data-testid="boundary-fix" type="button" @click="探针还会炸 = false">把抛错的原因改掉</button>
<NuxtErrorBoundary @error="记边界报错">
  <LabBoomPanel v-else :where="探针还会炸 ? 'render' : 'value'" />
  <template #error="{ error: 收到的错误, clearError: 清掉它 }">
    <p data-testid="boundary-raw">原文插值 LAB_ERROR_RAW_809：{{ 收到的错误 }}</p>
    <button data-testid="boundary-clear" type="button" @click="清掉它">清掉它</button>
  </template>
</NuxtErrorBoundary>
```

关键在于这颗按钮换的是**递进探针的那个 prop**，不是把探针拆掉：改成假之后这位组件还在原来的位置，只是它这一趟不再抛。没有这一位，"清掉了"和"清掉之后又抛了一次"在 DOM 上会读成同一个样子。判据在 `test/e2e/error-boundary.spec.ts`，这一站加了五条（档五到档九），整档九条一起跑 `9 passed`。

## 三条读数，原样打印（后面两节还各有一条）

```text
[边界·显示] {"形状":" 错误形状：{\"是Error实例\":true,\"名字\":\"Error\",\"消息\":\"LAB_BOOM_807：渲染期抛错（没有边界包住它）\",\"有stack\":true,\"可枚举键\":[]}","原文":" 原文插值 LAB_ERROR_RAW_809：Error: LAB_BOOM_807：渲染期抛错（没有边界包住它）","原文长度":58,"消息在里面":true,"含堆栈字样":false}

[边界·清了但没修] {"清前":{"默认槽":0,"探针":0,"错误槽":1,"计数":" 边界里红过：1 次 ","探针状态":" 边界里那位探针现在：还是会抛","正文":1,"边界外那块":1},
                    "清后":{"默认槽":0,"探针":0,"错误槽":1,"计数":" 边界里红过：2 次 ","探针状态":" 边界里那位探针现在：还是会抛","正文":1,"边界外那块":1},
                    "控制台条数":0}

[边界·先修再清] {"改完没清":{"默认槽":0,"探针":0,"错误槽":1,"计数":" 边界里红过：1 次 ","探针状态":" 边界里那位探针现在：已经不抛了","正文":1,"边界外那块":1},
                 "清后":{"默认槽":0,"探针":1,"错误槽":0,"计数":" 边界里红过：1 次 ","探针状态":" 边界里那位探针现在：已经不抛了","正文":1,"边界外那块":1}}
```

## 显示那一半：页面上只有五十来个字符

`error` 交出来的不是字符串、也不是被拆过的对象，就是 `onErrorCaptured` 收到的那个原始错误：`是Error实例: true`、`名字: "Error"`、`有stack: true`。但 `可枚举键` 读回来是空数组——`name`、`message`、`stack` 这几位在 Error 上都不是可枚举属性，所以 `Object.keys(错误)` 一个都不给。这决定了它能不能直接往界面上放。

官方示例那种 `{{ error }}` 之所以只留下 `Error: 消息`，原因在插值那一句判断里（`node_modules/@vue/shared/dist/shared.esm-bundler.js:533`）：值是对象时，**只有它自己没有可用的 `toString` 才走 JSON 那条路**，有 `toString` 的走 `String(val)`。Error 有自己的 `toString`，输出就是「名字: 消息」。这次读到的 `原文长度: 58`、`含堆栈字样: false` 就是这个形状。

这一条值得反过来钉一次，不然断言是空的：把模板里那行改成读 `.stack`，同一位置变成

```text
"原文长度":656,"含堆栈字样":true
```

整段调用帧（`at setup (http://localhost:3387/_nuxt/CVuD3bVy.js:1:179)` 之类）全铺到用户能选中的文本里，判据立刻红。改回去重新跑就绿了。

所以 `{{ error }}` 这一句的安全，不是"框架帮你过滤了"，而是它撞上了 Error 的 `toString`。**换一种抛法就没有这层保护**：这一档右边还挂着第二个边界，里面那位探针抛的不是 Error，而是自己拼的普通对象：

```ts
if (props.where === 'plain') {
  throw {
    status: 500,
    code: 'LAB_PLAIN_809',
    detail: { field: 'form.title', hint: '这一位在库里，不该出现在页面上' },
  }
}
```

```text
[边界·普通对象] {"原文":" {\n  \"status\": 500,\n  \"code\": \"LAB_PLAIN_809\",\n  \"detail\": {\n    \"field\": \"form.title\",\n    \"hint\": \"这一位在库里，不该出现在页面上\"\n  }\n}","原文长度":123,"是字符串形状":false,"字段名在里面":true,"红过":" 第二个边界里红过：1 次 ","整页错误页":0,"控制台条数":0}
```

同一行插值，这次把 `status`、`code`、`detail.field` 一个个缩进排好写在页面上（123 个字符），因为这种对象走的是 JSON 那条分支。它也不是变成没意义的 `[object Object]`（`是字符串形状: false`）。这两条读数放在一起才是这一半的结论：**往上抛的东西最好是 Error**，往界面上放的东西最好是**你自己挑出来的那一两位**（上一站那句 `名字: 消息`），别把整个 `error` 交出去。

## 回去那一半：清的是状态，不是原因

`清了但没修` 那一次读数是这一站的主角。点之前 `错误槽: 1`、`计数: 1 次`；点 `清掉它` 之后，最终 `错误槽` 还是 1，但 **`计数` 变成了 2 次**。中间那一帧太快，留不住也不要紧——计数翻到 2 就是它确实重新渲过默认槽的证据：默认槽里挂的就是那位探针，它不被重新挂上一次就抛不出第二次。

为什么是"重渲"而不是"消失"？把 `<NuxtErrorBoundary>` 那层包装摊开只有几行（`node_modules/nuxt/dist/app/components/nuxt-error-boundary.vue`）：模板里两个槽是互斥的（`:2-14`，`v-if="error"` 走 `#error`，`v-else` 走默认槽），而 `clearError` 就是 `error.value = null`（`:28-30`）。**它全部的工作就是把边界自己那一位状态清空，碰不到"为什么抛"**。清空之后 Vue 按 `v-else` 渲默认槽，原来那段代码该怎么抛还怎么抛。

连点三次是同一个道理，一格一格往上走，不会自己停：

```text
[边界·连环清] {"每一格":["边界里红过：2 次","边界里红过：3 次","边界里红过：4 次"],"最终错误槽":1,"最终探针":0,"正文":1}
```

第一次红是挂上探针那一次抛的（计数 1），后面每一格都对应"清掉 → 默认槽重渲 → 再抛一次"。用户对着这颗按钮连点三下，页面上什么都没有，只有那个数字在走——这就是为什么不能只放一颗"重试"按钮就完事。

`先修再清` 那一次给出了另一半答案，而且它顺手拆出一个更容易搞混的点：

- **只改原因、不点清掉**：读回来 `错误槽: 1`、`探针: 0`、`探针状态: 已经不抛了`。原因改了，界面**还停在错误块上**——已经红过一次这件事不会自己复原。
- **改完原因再点清掉**：`探针: 1`、`错误槽: 0`、计数停在 `1 次`。这时默认槽重新渲出来的是那位不再抛的组件，它就在原地正常工作了。

两步都要做，顺序无所谓，缺一步界面就不是你要的那个状态。这就是官方那句"先确保错误真的被解决了"在页面上的意思：那颗 `清掉它` 的按钮**只该出现在你真的能解决的地方**——重新载入数据、退回上一层、把那位组件换成不需要它的界面。如果只是把状态清了，用户看到的是同一块地方再红一次。

另外把上一站那条陷阱再钉一遍：这两次清与红，`控制台条数` 都是 0。界面上那个"红过 2 次"是唯一能读到的痕迹。

## 排错表

| 想知道的 | 读哪一位 | 只读一处会错在哪 |
| --- | --- | --- |
| `{{ error }}` 会不会漏细节 | 那一行的长度与有没有 `at …` 字样 | 只看"是 Error"就以为安全，普通对象走的是另一条 |
| 抛上去的是哪种东西 | 可枚举键那一位（Error 是空数组，自己拼的对象会列出字段名） | 不看键，就不知道插值会走哪条分支 |
| 清错误清了什么 | 计数有没有再翻一次 | 只看错误槽还在，会以为按钮没生效 |
| 改原因够不够 | 只改原因那一次的错误槽 | 不看会以为代码修好界面就自动复原 |
| 修好了清完是什么样 | 探针在不在场、计数停在哪 | 只看页面没红，可能压根没挂上东西 |
| 这次红过有没有别处记着 | 计数 ＋ 控制台条数 | 只查控制台会得出"没红过" |

## 自己动手跑一遍

```bash
npm run build
npx playwright test test/e2e/error-boundary.spec.ts
```

预期 `9 passed`。手点版：进 `?islandLab=boundary`，先点`在边界里挂上那一位会炸的组件`，看那行原样插值只有五十来个字符；再点`清掉它`——页面闪回默认槽一下又回到错误块，`边界里红过` 变成 2 次。刷新重来，这次先点`把抛错的原因改掉`（界面不动），再点`清掉它`（这次停在"探针面板：这一位正常工作"上，计数不再涨）。最后点`在第二个边界里抛一个普通对象`，比较那两行插值的长度差。

## 下一篇

错误边界（4）：`@error` 事件，把边界里红过一次变成可读的数——同一次抛错，`@error`、`useTemplateRef` 拿到的 `error`、顶层 `vue:error` 钩子三条口子各被叫到哪一次。

## 本文依据

- 官方：讲错误处理那一节（Render Error in Component 段那句范围话与括号里那句「set `error = null` it will trigger re-rendering the default slot … or the error slot will just be rendered a second time」、`#error="{ error, clearError }"` 示例）；讲 `<NuxtErrorBoundary>` 那一节（Slots 段 `#error`、Events 段 `@error`）
- 包内源码：`node_modules/nuxt/dist/app/components/nuxt-error-boundary.vue`（`:2-14` 两个槽互斥、`:27-30` `error` 是一位 `shallowRef` 而 `clearError` 只把它置回 null、`:39-46` `onErrorCaptured` 之后 `return false`）、`node_modules/@vue/shared/dist/shared.esm-bundler.js:533`（`toDisplayString` 那一句分支）
- 仓库：`app/pages/docs/[slug].vue`（`boundary` 档：`边界错误形状`、`探针还会炸`、那颗改原因的按钮、`{{ 收到的错误 }}` 那一行、第二个边界与它的普通对象探针）、`app/components/LabBoomPanel.vue`（`where` 多了 `plain` 这一档）、`test/e2e/error-boundary.spec.ts`（新增 5 条，整档 9 条）
- 上一批：[808 - 错误边界（2）：给这一块套上 NuxtErrorBoundary，抛错只吃掉这一块](/posts/f6fe6459234353927cd4b13fcdc6d199/)
