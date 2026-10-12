---
url: /posts/3f393e7f1ed8fc69efd7a593a597c775/
title: useCookie 选项表逐条：14 个键与两处要小心
date: 2027-01-05T10:00:00+08:00
updated: 2027-01-05T10:00:00+08:00
author: cmdragon

summary: >-
  摘要：这一篇把 `useCookie` 的选项表整张抄完：**14 个键**，官方先给一句 "Most of the options will be directly passed to the cookie package."，所以属性类键（`maxAge`、`expires`、`httpOnly`、`secure`、`partitioned`、`domain`、`path`、`sameSite`）对应的是 `Set-Cookie` 的属性，行为在浏览器；另一半（`decode`、`encode`、`default`、`watch`、`refresh`、`readonly`）是 `useCookie` 自己的行为。**默认值我全部照表写**：`decode` 是 "decodeURIComponent + destr"、`encode` 是 "JSON.stringify + encodeURIComponent"、`default` 是 `undefined`、`watch` 是 `true`、`refresh` 是 `false`（v4.4 徽标）、`readonly` 是 `false`、`maxAge` 与 `expires` 与 `domain` 与 `sameSite` 是 `undefined`、`path` 是 `'/'`、`httpOnly`/`secure`/`partitioned` 是 `false`。两处我必须提醒：一，`secure: true` 的 note 直接点名水合——"This can lead to hydration errors."（没 HTTPS 时浏览器不再回传 cookie，服务端与客户端读到不同值）；二，`expires` 与 `maxAge` 同时设时官方引规范说 "if both `expires` and `maxAge` is set, then `maxAge` takes precedence, **but not all clients may obey this**, so if both are set, they should point to the same date and time!"。另外 `watch` 的三档与官方两段示例配套：`false` 时"改了值但 cookie 不更新"（Readonly Cookies 一节），`'shallow'` 时"push 不触发、整体替换才触发"（`list.value &&= [...list.value]`）。

categories:
  - 前端开发

tags:
  - nuxt4
  - useCookie
  - 选项
  - 隐私属性
  - 水合

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


上一节是概念，这一节全是选项。我把 14 行按"谁在管"分成两组读——**这个分组是我为了记住的整理，文档本身是一张表**。

## 行为组：useCookie 自己实现的六项

| 键 | 类型 | 默认 | 官方说明（关键部分） |
| --- | --- | --- | --- |
| `decode` | `(value: string) => T` | `decodeURIComponent` + destr | "Custom function to decode the cookie value." 后接一句容错：**"If an error is thrown from this function, the original, non-decoded cookie value will be returned as the cookie's value."** |
| `encode` | `(value: T) => string` | `JSON.stringify` + `encodeURIComponent` | "Custom function to encode the cookie value." 两句都提到 "the value of a cookie has a limited character set (and must be a simple string)" |
| `default` | `() => T \| Ref<T>` | `undefined` | "Function returning the default value if the cookie does not exist. The function can also return a `Ref`." |
| `watch` | `boolean \| 'shallow'` | `true` | "Whether to watch for changes and update the cookie. `true` for deep watch, `'shallow'` for shallow watch, i.e. data changes for only top level properties, `false` to disable." |
| `refresh` :v4.4 | `boolean` | `false` | 见下文"过期时间不会自动续" |
| `readonly` | `boolean` | `false` | "If `true`, disables writing to the cookie, on both server and client. A `default` value is still returned by the composable but is never persisted to the browser." |

两项我觉得最实用：

- **`decode` 抛错不会崩**：它会回落到"未解码的原始字符串"。这意味着**你自定义解码写坏了也不会白屏，只会读到怪值**——排错时别只盯 `useCookie` 调用点。
- **`readonly` 是"两侧都不写"**，但 `default` 仍然返回。这正是 199 说的"服务端算得出、客户端不该写回"那一类配置的正确形状。

## 属性组：透传给 cookie 的八项

| 键 | 类型 | 默认 | 说明要点 |
| --- | --- | --- | --- |
| `maxAge` | `number` | `undefined` | "Max age in seconds"，对应 `Max-Age`；"By default, no maximum age is set." |
| `expires` | `Date \| (() => Date \| undefined)` | `undefined` | 过期日期或 getter；返回 `undefined` 就是会话 cookie；**"The getter should be pure (no side effects)."** |
| `httpOnly` | `boolean` | `false` | note：**"compliant clients will not allow client-side JavaScript to see the cookie in `document.cookie`."** |
| `secure` | `boolean` | `false` | note：**"Be careful when setting this to `true`... as compliant clients will not send the cookie back to the server in the future if the browser does not have an HTTPS connection. **This can lead to hydration errors.**"** |
| `partitioned` | `boolean` | `false` | "This is an attribute that has not yet been fully standardized, and may change in the future."；"many clients may ignore this attribute until they understand it."（另有 privacycg/CHIPS 的 proposal 链接） |
| `domain` | `string` | `undefined` | 不设时"most clients will consider applying the cookie only to the current domain" |
| `path` | `string` | `'/'` | 默认是规范里的 "default path" |
| `sameSite` | `boolean \| string` | `undefined` | 五种取值逐条：`true`⇒`Strict`、`false`⇒不设、`'lax'`、`'none'`（explicit cross-site）、`'strict'` |

### 第一处要小心：secure 会用水合错配收场

`secure` 那句是我读这页最意外的地方——**cookie 选项怎么会引起水合错误**：

链路很清楚：设了 `secure` 却没有 HTTPS（本地用 `http://localhost` 之外的地址、或者反代把 TLS 掐在边缘），浏览器就不回传这个 cookie。于是服务端读到 `undefined`、渲染出 A；客户端某一侧读到 B 或仍然拿不到，水合阶段两侧 HTML 不一致（199 那张表的第一类成因正是"两侧数据不一致"）。

我的结论只有一句可执行：**开 `secure` 之前先在目标环境确认 cookie 真的回得来**（浏览器 DevTools 的 Application → Cookies 里看它在不在）。

### 第二处要小心：expires 与 maxAge 同时设

官方引 RFC 6265 的原话：

> The cookie storage model specification states that if both `expires` and `maxAge` is set, then `maxAge` takes precedence, but not all clients may obey this, so if both are set, they should point to the same date and time!

**"不是所有客户端都遵守优先级"** 这半句才是重点：设两个值就要保证两者指向同一时刻。另外还有一句常被忽略：**"If neither of `expires` and `maxAge` is set, the cookie will be session-only and removed when the user closes their browser."** —— 不写就是会话 cookie。

## watch 三档与官方两段示例

**`watch: false`**（Readonly Cookies 一节，注意它同时演示了 `default`）：

```vue [app/app.vue]
<script setup lang="ts">
const user = useCookie(
  'userInfo',
  {
    default: () => ({ score: -1 }),
    watch: false,
  },
)

if (user.value) {
  // the actual `userInfo` cookie will not be updated
  user.value.score++
}
</script>
```

**`watch: 'shallow'`**（Writable Cookies 一节）把"顶层换、嵌套不改"演示得很直白：

```ts
const list = useCookie('list', {
  default: () => [],
  watch: 'shallow',
})

function add () {
  list.value?.push(Math.round(Math.random() * 1000))
  // list cookie won't be updated with this change
}

function save () {
  // the actual `list` cookie will be updated
  list.value &&= [...list.value]
}
```

所以 `'shallow'` 的含义是：**改数组元素不写 cookie，整体替换才写**。`add`/`save` 两个按钮就是官方给的对照。

## refresh 与"过期时间不会自动续"

`refresh: true` 的原文（含那句必须自己赋值）：

> If `true`, the cookie expiration will be refreshed on every explicit write (e.g. `cookie.value = cookie.value`), even if the value itself hasn't changed. Note: the expiration is not refreshed automatically; you must assign to `.value` to trigger it.

配套示例：

```vue [app/app.vue]
<script setup lang="ts">
const session = useCookie('session', {
  maxAge: 60 * 60, // 1 hour
  refresh: true,
  default: () => 'active',
})

// Even if the value does not change,
// the cookie expiration will be refreshed
// every time the setter is called
session.value = 'active'
</script>
```

另一条路是让 `expires` 本身是 getter（Dynamic Expiration 一节）：

> Use a function for `expires` when you want a fresh expiration date every time the cookie is written (for example, sliding sessions or tokens):

```ts
const token = useCookie('token', {
  // Re-evaluated on every write: keep this getter pure
  expires: () => new Date(Date.now() + 60 * 60 * 1000), // 1 hour from now
})

// Assigning a new value also refreshes the cookie expiration
token.value = 'new-token'
```

两者的分工我这样记：**`refresh` 管"同值重写要不要续期"，`expires` 的 getter 管"每次写入时过期时间怎么算"**（两句都强调要赋值才触发）。滑动会话那种需求，官方推荐的是 getter 那一档（"for example, sliding sessions or tokens"）。

## 自查

1. 不改 `encode`/`decode` 时，值是怎么被存取的？（答：默认编码 `JSON.stringify` + `encodeURIComponent`，默认解码 `decodeURIComponent` + destr。）
2. `watch: 'shallow'` 下 `list.value.push(x)` 会不会写 cookie？（答：不会，官方注释 "list cookie won't be updated with this change"；要整体替换才写。）
3. 两处最容易出事的是哪两个键？（答：`secure`（没有 HTTPS 时不回传 ⇒ 官方明说 "This can lead to hydration errors."）与 `expires`/`maxAge` 同时设（优先级不保证一致，应指向同一时刻）。）

## 下一篇

257 - refreshCookie：什么时候才需要手动刷新 cookie 值

## 本文依据

- `api/composables/use-cookie`：Parameters 前两句（"Options to control cookie behavior"、"Most of the options will be directly passed to the cookie package."）；14 行选项表全部原文（含各默认值、v4.4 徽标、`decode` 的抛错回落句、`expires` 的 pure getter 与 RFC 优先级两句、`httpOnly`/`secure`/`partitioned` 三条 note、`sameSite` 五值）；Type 里 `CookieOptions` 自有六键；Readonly Cookies 与 Writable Cookies 两段示例（含四处原注释）；Refreshing Cookies 一节（`refresh` 长说明与 `session` 示例）；Dynamic Expiration 一节（说明句、getter 示例与两条注释）。
- `guide/best-practices/hydration`：两侧数据不一致的成因与 `useState`/`useCookie` 两个解法（199 已逐句引）。
- `api/composables/use-state`：`default` 可返回 Ref、`shallowRef` 一档（251 已逐句引）。
