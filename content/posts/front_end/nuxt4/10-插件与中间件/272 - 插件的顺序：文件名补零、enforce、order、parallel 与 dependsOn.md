---
url: /posts/455aa0452e6b971fda60a367e499b52c/
title: 272 - 插件的顺序：文件名补零、enforce、order、parallel 与 dependsOn
date: 2027-01-10T11:00:00+08:00
updated: 2027-01-10T11:00:00+08:00
author: cmdragon

summary:
  摘要：多个插件之间是有先后关系的——B 要用 A 提供的东西，B 就得排在 A 后面。Nuxt 给了四样东西控制顺序：文件名的"字典序编号"、对象写法的 enforce 三档、order 数字、以及 parallel 和 dependsOn。这一篇把它们的规则逐个说清，重点标出三处最容易翻车的点：文件名按字符串排序所以 10 排在 2 前面、order 会覆盖 enforce、以及"默认是顺序加载"这个前提。最后用官方那两条 important 说明顺序错会在组合式上怎么表现。

categories:
  - 前端开发

tags:
  - nuxt4
  - 插件顺序
  - enforce
  - parallel
  - dependsOn
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


前提先钉死：**默认情况下，Nuxt 是顺序加载插件的**，官方原话——"By default, Nuxt loads plugins sequentially." 也就是说没有你做特别安排的话，插件是一个接一个跑的。这一篇讲的就是"你凭什么安排这个先后"，一共四个手段。

## 手段一：文件名的字典序编号

最朴素、也是最常用的办法是给文件名加数字前缀。官方给的例子：

```bash [Directory structure]
plugins/
 | - 01.myPlugin.ts
 | - 02.myOtherPlugin.ts
```

> In this example, `02.myOtherPlugin.ts` will be able to access anything that was injected by `01.myPlugin.ts`.

也就是 `02` 能用到 `01` 已经 provide 出去的东西。规则就是**按文件名字典序（string sort）排列**。

这里有个坑，官方专门用一条 note 提醒，因为它反直觉：

> In case you're new to 'alphabetical' numbering, remember that filenames are sorted as strings, not as numeric values. For example, `10.myPlugin.ts` would come before `2.myOtherPlugin.ts`. This is why the example prefixes single digit numbers with `0`.

因为按字符串比，`"1"` 这个字符小于 `"2"`，所以 `10.` 排在 `2.` 前面——和数字直觉相反。解法就是示例里的做法：**个位数前面补 `0`**（`01`、`02`…`09`，再 `10`）。补零到能覆盖你的最大插件数即可。

## 手段二：enforce 三档

对象写法有个 `enforce` 键，官方参数表描述是 "Controls when the plugin runs relative to other plugins."，取值三档：

```ts
export default defineNuxtPlugin({
  name: 'my-plugin',
  enforce: 'pre', // 或 'post'
})
```

- `'pre'`：尽量靠前跑
- `'default'`：默认档（不写就是这个）
- `'post'`：尽量靠后跑

它给的是**相对粗的分层**：`pre` 的都在前面、`post` 的都在后面、中间的按 default 归堆。适合"我这个必须最早、那个必须最晚"这种粗粒度需求。

注意 271 篇提过的那条约束在这里同样成立：`enforce` 得写成字面量，别 `import.meta.server ? 'pre' : 'post'`，否则静态分析打掉、还可能拿不到你想要的位置。

## 手段三：order 数字（会盖掉 enforce）

比 enforce 更细的是 `order`，一个数字。官方参数表对它的描述信息量很大，逐字看：

> `order`: This allows more granular control over plugin order and should only be used by advanced users. **It overrides the value of `enforce` and is used to sort plugins.**

三句话三个意思：

1. 它是**更精细**的顺序控制（granular）——用一个数字直接排。
2. 官方明说**只给高级用户用**（"should only be used by advanced users"）。日常用文件名补零 + enforce 基本够了。
3. 关键点：**`order` 覆盖 `enforce`**。当一个插件同时给了 `enforce` 和 `order` 时，`enforce` 不生效，Nuxt 按 `order` 排序。别指望两个叠加。

也就是说四个手段里，"精调顺序"这一路是：文件名决定同层内的先后 → enforce 做粗分层 → order 直接按数字排、且会盖住 enforce。

## 手段四：parallel 与 dependsOn

顺序加载是默认，但你可以让某些插件**并行**，也可以让某个插件**等**另一个。

**parallel**：官方原话——

> By default, Nuxt loads plugins sequentially. You can define a plugin as `parallel` so Nuxt won't wait until the end of the plugin's execution before loading the next plugin.

```ts [plugins/my-plugin.ts]
export default defineNuxtPlugin({
  name: 'my-plugin',
  parallel: true,
  async setup (nuxtApp) {
    // 下一个插件会立刻开始执行，不等这个跑完
  },
})
```

参数表对 parallel 的措辞是 "Whether to execute the plugin in parallel with other parallel plugins."——注意是和其它标了 `parallel` 的插件并行，不是和所有插件并行。一个很实际的用处：某个插件 `setup` 里有 `await $fetch(...)`，你不希望后面的插件被这次网络请求阻塞，就标 `parallel`。

**dependsOn**：反过来，某个插件必须等特定插件跑完再动。官方——

> If a plugin needs to wait for another plugin before it runs, you can add the plugin's name to the `dependsOn` array.

```ts [plugins/depending-on-my-plugin.ts]
export default defineNuxtPlugin({
  name: 'depends-on-my-plugin',
  dependsOn: ['my-plugin'],
  async setup (nuxtApp) {
    // 会等 my-plugin 执行完才开始
  },
})
```

`dependsOn` 里填的是**插件的 `name`**（所以要用 dependsOn，被依赖的插件得有 `name`）。参数表对它的描述是 "Array of plugin names this plugin depends on. Ensures proper execution order."。

parallel 和 dependsOn 是能搭 together 用的：一组插件并行跑，其中某个再声明它依赖谁。它们都是对象写法的键，同样要静态可分析。

## 顺序错了会在哪冒出来

顺序问题最常表现为"组合式里拿不到别的插件提供的东西"。官方 Using Composables 一节有两条 important，正好是两种典型失败：

> **If a composable depends on another plugin registered later, it might not work.**
> Plugins are called in order sequentially and before everything else. You might use a composable that depends on another plugin which has not been called yet.

意思是：某个自动导入的组合式内部用了 `$something`（另一个插件 provide 的），而这个插件排在后面还没跑，那这个组合式在靠前的插件里调用时，`$something` 还不存在。这就是前面费心排顺序的直接动机——让"被依赖的"排在"依赖它的"前面。

> **If a composable depends on the Vue.js lifecycle, it won't work.**
> Normally, Vue.js composables are bound to the current component instance while plugins are only bound to `nuxtApp` instance.

这条和顺序无关但放一起记：插件绑的是 `nuxtApp` 实例，不是组件实例，所以那些依赖"当前组件生命周期"的组合式（`onMounted` 之类）在插件里不工作。这是插件的固有边界，不是排序能救的。

## 自查

- 文件名用了数字前缀？个位数没补零的话 `10.` 会跑到 `2.` 前面。
- 同时写了 `enforce` 和 `order`？`order` 赢，`enforce` 被覆盖——别两个都指望生效。
- 用 `dependsOn`？被依赖的那个插件必须有 `name`，数组里填的是名字不是文件名。
- 给慢的插件标 `parallel` 前想清楚：它是不是后面有插件要等它的结果？要等就别并行，改用它下游的 `dependsOn`。
- 有组合式在靠前的插件里读靠后插件 provide 的 `$x`？把顺序调过来，或者干脆把这个能力改成自动导入组合式（270 那条推荐）。

## 下一篇

270-299 从这里继续。下一篇 273 讲 `app:created`、`app:beforeMount`、`app:mounted`、`app:rendered` 这几个应用生命周期钩子在插件里的订阅时机。
