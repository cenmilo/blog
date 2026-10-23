---
url: /posts/d28b274ae90f03d9be3616f0c71fd7f0/
title: 505 - 内置 mock 只有两个：intersectionObserver 与 indexedDB
date: 2027-03-29T10:00:00+08:00
updated: 2027-03-29T10:00:00+08:00
author: cmdragon

summary:
  摘要：`@nuxt/test-utils` 为 DOM 环境内置的 mock，官方 Built-In Mocks 一节**只列了两项**：`intersectionObserver`（默认 `true`，但造的是"没有任何功能的假类"）与 `indexedDB`（默认 `false`，打开后用 fake-indexeddb 提供**可用**的模拟）。两者的配置位置在 `vitest.config.ts` 的 `environmentOptions` 下，而配置键的名字与标题不一致——**标题写 `indexedDB`，键要写成 `indexedDb`**。这一节没有第三项，也没给出其他浏览器 API 的处理办法。

categories:
  - 前端开发

tags:
  - nuxt4
  - 测试
  - mock
  - dom
  - vitest
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

组件里用 `IntersectionObserver` 做图片懒加载，测试跑在 happy-dom 或 jsdom 里。这种浏览器才有的 API 谁来补？

官方在测试页给了一节专门管这件事，标题是 Built-In Mocks。看完你会发现它的范围比名字小得多。

## 定性句与两项清单

整节开头只有一句：

> `@nuxt/test-utils` provides some built-in mocks for the DOM environment.

下面列的，就两项。定义原句都在这：

| 名称 | 默认 | 官方描述 | 实际能做什么 |
| --- | --- | --- | --- |
| `intersectionObserver` | `true` | creates a dummy class without any functionality for the IntersectionObserver API | 有类可 `new`，但**无任何功能** |
| `indexedDB` | `false` | uses fake-indexeddb to create a functional mock of the IndexedDB API | 打开后是**可用**的 IndexedDB 模拟 |

这一表里最值得停一下的是两行默认的**方向相反**：`intersectionObserver` 默认开着，但开的是空壳；`indexedDB` 默认关着，开了才是能用的实现。

所以"默认值是 `true`"不等于"能测IntersectionObserver 的行为"。官方对它的用词是 **dummy class without any functionality**——如果你的组件靠 `IntersectionObserver` 的回调来决定要不要加载图片，这个空壳不会把回调叫起来，测试看到的仍然是"没加载"。**文档没有说这个空壳具体提供哪些方法、`observe()` 调用后发生什么。**

`indexedDB` 那行给的实现来源是 `fake-indexeddb` 这个库，描述词是 **functional mock**——与上一行的 dummy 是两个级别的东西。

## 配置写在哪、键名有个坑

官方给的配置位置一句话：**"These can be configured in the `environmentOptions` section of your `vitest.config.ts` file"**，示例整份如下：

```ts
import { defineVitestConfig } from '@nuxt/test-utils/config'

export default defineVitestConfig({
  test: {
    environmentOptions: {
      nuxt: {
        mock: {
          intersectionObserver: true,
          indexedDb: true,
        },
      },
    },
  },
})
```

路径是 `test.environmentOptions.nuxt.mock.<开关>`，三层。两个键名照抄这里，而不是照抄小节标题：

- 标题写 `intersectionObserver`，键也写 `intersectionObserver`——一致。
- 标题写 `indexedDB`（大写 DB），**键写作 `indexedDb`**。

这个大小写差异只在配置这一处出现，很容易按标题敲成 `indexedDB`。文档没有说明写错键会怎样（报错还是被忽略），所以别把"配了没生效"当成框架问题——先回去核对键名拼法。

另一处值得注意：这份配置挂在 `environmentOptions` 的 **`nuxt`** 之下，而 `nuxt` 是环境名。也就是说这套 mock 是**给 Nuxt 环境的测试用的**。文档没说在 `environment: 'node'` 的 project 里配它会不会被读——那里本来也没有 DOM。

## 文档没写的部分

为了不臆造，把缺口列出来：

- **只有这两项。** 这一节没列 `ResizeObserver`、`matchMedia`、`canvas`、`clipboard`、`IntersectionObserver` 之外的任何其他 API，也没说这些该怎么办。
- 没说这两个 mock 在哪个环境层级生效（jsdom 与 happy-dom 是否一致）。
- 没给"关掉 `intersectionObserver`"之后会是什么表现——示例里只演示了把它设成 `true`（与默认相同）。
- 没有说明 `fake-indexeddb` 需要你自己装还是随 test-utils 带。这一条要确认，最直接的办法是把 `indexedDb: true` 打开、跑一次用 IndexedDB 的测试，看是否报缺模块。

想验证空壳这件事，做法很短：在测试里断言你观察到的行为，而不是断言"官方 mock 了它"。

```ts
import { expect, it } from 'vitest'

it('probe what the DOM environment actually gives us', () => {
  expect(typeof IntersectionObserver).toBe('function')
  const observer = new IntersectionObserver(() => {})
  observer.observe(document.createElement('div'))
  // 到这里回调不会被触发：这是空壳的行为，不是文档的承诺
})
```

**上面这段是补的取证写法，官方在这一节没有任何示例代码。** 它解决的问题是：把"以为 mock 会怎样"换成"实测它怎样"，然后再决定这个测试要不要依赖可见性回调——真要测懒加载触发，得自己提供实现（在测试里覆盖 `IntersectionObserver`，或者走端到端在真浏览器里滚动）。

## 自查

- Built-In Mocks 一节只列两项，方向相反：`intersectionObserver` 默认 `true` 但是**无功能空壳**，`indexedDB` 默认 `false`、开了才是 fake-indexeddb 的可用模拟。
- 空壳不会触发回调；文档没列它的方法，也没说 `observe()` 之后发生什么。
- 配置路径是 `test.environmentOptions.nuxt.mock.*`，**IndexedDB 的键要写成 `indexedDb`**，与小节标题拼法不同。
- 文档没说写错键名的后果，也没说这两项在非 Nuxt 环境 project 里的行为。
- 其他浏览器 API 官方没给内置 mock，也没给替代方案；需要行为触发就自己提供实现或走端到端。

## 下一篇

下一篇 506 给 Helpers 一节的完整清单：五个入口各自的参数、返回项、以及它们与 e2e 那套混用时的硬限制。
