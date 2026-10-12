---
url: /posts/7779812878e1c32f49812e44b66cc07a/
title: components：一节只给一份默认 dirs 表，没有子项清单
date: 2027-04-23T10:00:00+08:00
updated: 2027-04-23T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：`## components` 有两句定性（"Configure Nuxt component auto-registration." 与"这些目录里的组件可以在页面、布局和其他组件里直接使用，**不需要显式 import**"）、一行 `**Type**: object`、一份**两个元素**的默认 `dirs` 表（`{ path: "~/components/global", global: true }` 与裸字符串 `"~/components"`），加一条指向组件目录那一页的 See。**这一节没有任何 `###` 子项**——`pathPrefix`、`prefix`、`priority`、`global` 这些字段在配置参考里一个都没有条目。还有一处形状分歧：**配置页的类型行写 `object`，而组件目录那页的示例直接写 `components: [ { path, pathPrefix: false } ]`（数组）**。

categories:
  - 前端开发

tags:
  - nuxt4
  - 组件自动导入
  - 扫描目录
  - 全局组件
  - 配置
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

组件不用 import 就能用，这条自动行为就是 `components` 这一节管的。你要改扫描目录、想让名字不带路径前缀，都会来这一节找字段——然后发现**这一节一个字段都没列**。

## 这一节的记载

`## components` 全文是两句定性、一行类型、一份默认值、一条 See：

> Configure Nuxt component auto-registration.
>
> Any components in the directories configured here can be used throughout your pages, layouts (and other components) without needing to explicitly import them.

- **Type**: `object`
- **Default**

```json
{
  "dirs": [
    {
      "path": "~/components/global",
      "global": true
    },
    "~/components"
  ]
}
```

See 指向 `directory-structure/app/components`。

**这一节没有 `###` 子项。** 也就是说 `dirs` 这一项本身没有条目，`path`、`global`、`pathPrefix`、`prefix`、`priority` 这些在别处出现过的字段，在配置参考里**一个都没有对应小节**——你能从这一节确证的只有：默认值是上面那两元素，其中第一项是**对象形式**、第二项是**裸字符串**。

那份默认表里两元素的写法不同，这一条是有用的：`~/components/global` 被单独列成带 `global: true` 的对象，而 `~/components` 用最简形式。**同一个数组里混两种形状是官方自己的默认值写法**，所以这是合法的输入形状。

## 类型行与示例的形状不一致

配置页这一行的类型写的是 `object`。但组件目录那一页给的两个例子，**没有一个符合"对象"**：

```ts [nuxt.config.ts]
export default defineNuxtConfig({
  components: [
    {
      path: '~/components',
      pathPrefix: false, // [!code ++]
    },
  ],
})
```

这是数组形式。文档对它的说明是："If you want to auto-import components based only on its name, not path, then you need to set `pathPrefix` option to `false` using **extended form of the configuration object**." 效果也当场给了：`~/components/Some/MyComponent.vue` 用 `<MyComponent>` 而不是 `<SomeMyComponent>`，官方注明这与 Nuxt 2 用的策略相同。

另一种写法确实回到对象形式，但多了 `global`：

```diff
  export default defineNuxtConfig({
    components: {
+     global: true,
+     dirs: ['~/components']
    },
  })
```

也就是说文档里同时出现过**数组形式**和**带 `global` ＋ `dirs` 的对象形式**，而配置参考只声明 `object`。**这三种形状的确切类型定义不在这页**，要确认只能去读类型或实测。

## 全局注册：官方的态度写在措辞里

`global: true` 那种写法，目录页给的前缀是 **"Alternatively, though not recommended"**，理由也一并给了：它会给**所有**组件生成异步 chunk，让它们在整个应用里可用。

官方推荐的替代有两条：

- 把要全局用的组件放进 **`~/components/global` 目录**（这正是默认 `dirs` 表第一项在做的事）。
- 文件名用 **`.global.vue` 后缀**。

还有一句提醒：**每个全局组件都单独占一个 chunk，所以不要滥用**。另外一条 `::note` 补了粒度：**`global` 选项也能按目录设置**——这与默认表里那个对象形状（`{ path, global: true }`）是同一件事的两种说法。

## 自查

- 配置页 `## components`：两句定性 ＋ `Type: object` ＋ 两元素默认表 ＋ 一条 See，**零个子项小节**。
- 默认 `dirs` 的两项**写法不同**（对象项 `~/components/global` 带 `global: true`；字符串项 `~/components`），这确立了"数组里可混两种形状"。
- 想按名字而非路径自动导入：`pathPrefix: false`，要用**扩展的对象形式**；效果是让 `Some/MyComponent.vue` 变成 `<MyComponent>`（与 Nuxt 2 策略一致）。
- `components: { global: true, dirs: [...] }` 这种全量全局写法被官方写成 **not recommended**，代价是每个组件一个异步 chunk；推荐 `~/components/global` 或 `.global.vue`。
- 类型行 `object` 与目录页示例的数组形式**不一致**，形状的确切定义不在配置参考里。

## 下一篇

下一篇 581 看 `## imports`：**三个子项**（`dirs`、`global`、`scan`），其中一个（`global`）**只有类型行和默认值行、连一句说明都没有**；`dirs` 那句则明确说了它**不会覆盖默认目录**。

## 本文依据

- https://nuxt.com/docs/4.x/api/nuxt-config（`## components` 两句定性、`**Type**: object`、默认 `dirs` JSON 全文、指向组件目录页的 See）
- https://nuxt.com/docs/4.x/directory-structure/app/components（`pathPrefix: false` 的扩展对象形式与示例、"same strategy as used in Nuxt 2" 与 `<MyComponent>` 例子、"though not recommended" 的全局注册 diff 示例、`~/components/global` 与 `.global.vue` 两条替代、每个全局组件一个 chunk 的提醒、`global` 可按目录设的 `::note`）
