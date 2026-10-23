---
url: /posts/0d6e2831aa652f4351a188e1a385f7cd/
title: 735 - 工程把关（4）：npm run check：一条命令跑完该跑的全部
date: 2026-10-19T12:00:00+08:00
updated: 2026-10-19T12:00:00+08:00
author: cmdragon

summary:
  摘要：本地和 CI 用同一份清单，就不存在"本地绿 CI 红"这种只能靠记忆对齐的入口。这一站把 `npm run check` 摊开：六项按便宜的先跑、任何一项红就当场停并点名是哪项，并用两次故意跑歪验证这两句话——lint 红的时候 3.4 秒就结束，构建那 24 秒压根没跑。顺带把 Nuxt 命令行本身的几处脾气量清楚，包括一个会让 CI 悄悄放行的坑。

categories:
  - 前端开发

tags:
  - nuxt4
  - nuxt4项目实战
  - 聚合脚本
  - Nuxt CLI
  - CI
---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>

扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`

## 三十秒换一张表

交付状态跑一次 `npm run check`，终端最后给的是这张表：

```text
--- 逐项结果 ---
  ok    node        rc=0       0 ms
  ok    lint        rc=0    2781 ms
  ok    typecheck   rc=0    6402 ms
  ok    test:unit   rc=0    1570 ms
  ok    build       rc=0   24032 ms
  ok    test:nuxt   rc=0    3628 ms

全部通过
```

这一趟外面量到 **38864 ms**，同一份代码稍早还跑过一次，是 **38851 ms**——两次差 13 毫秒。表里的 build 一项就占 24032 毫秒，也就是整体的六成；前面四项加起来 10753 毫秒。所以这张表的第一条设计原则很朴素：**便宜的先跑**。静态检查、不起服务的单测放前面，最贵的那一步放后面，这样失败的时候你花的钱最少。

脚本本体不到一百行，清单就是四个字符串：

```js [scripts/check.mjs（节选）]
const npmSteps = ['lint', 'typecheck', 'test:unit', 'build', 'test:nuxt']

function runNpmStep (name) {
  console.log(`\n=== ${name} ===`)
  const started = Date.now()
  try {
    execSync(`npm run ${name}`, { stdio: 'inherit' })
    record(name, 0, Date.now() - started)
    return true
  } catch (error) {
    const status = error instanceof Error && 'status' in error ? Number(error.status) : -1
    record(name, Number.isFinite(status) ? status : -1, Date.now() - started)
    console.error(`FAILED: ${name}`)
    return false
  }
}
```

`npmSteps` 里每一项就是 `package.json` 里那条脚本的名字，跑的是 `npm run <名字>`；`stdio: 'inherit'` 让子进程直接接上终端，所以报错的时候屏幕上就是它自己的原话，不用二次转述。**命令串只来自这份写死的名单，不掺任何外部输入**——`execSync` 走的是 shell，把用户数据拼进命令串就是注入口袋，这一点 Node 自己也在警告里说得很直白。

名单最前面还有一项不在数组里：`runNodeCheck()`，就是第一篇之后立起来的那道版本闸门（只比主版本号，当前要求 22）。它排第一的理由不是"重要"，是它几乎不要钱——表里那一行写的是 `0 ms`。

## 故意跑歪两次，看它停在哪

原则讲完就得验。第一次把一条 lint 违规放回去：

```js [scripts/lint-probe.mjs]
export function probe () {
  const unused = 1
  debugger
  return 1
}
```

```text
$ npm run check
 3 problems (2 errors, 1 warning)
FAILED: lint
  ok    node        rc=0       0 ms
  FAIL  lint        rc=1    2922 ms

停止在 lint
退出码 1        总耗时 3352 ms
```

三件事同时成立：表**只有两行**，后面四项一行都没出现；整体退出码 1；而且最后一行点名是 `停止在 lint`。那行的 `3 problems (2 errors, 1 warning)` 里，两条 error 是探针，一条 warning 是上一站留下的 `vue/no-v-html`——它一直挂着，不影响退出码，但会跟着每次输出走。删掉探针后 34 秒省下来了，这一趟 3.3 秒就结束。

第二次换成一句只有类型检查才会管的东西（`export const probeValue: number = '这不是数字'` 放进 `app/utils/`），于是 lint 绿、typecheck 红：

```text
$ npm run check
app/utils/ts-probe.ts(1,14): error TS2322: Type 'string' is not assignable to type 'number'.
FAILED: typecheck
  ok    node        rc=0       0 ms
  ok    lint        rc=0    2708 ms
  FAIL  typecheck   rc=2    6449 ms

停止在 typecheck
退出码 1        总耗时 9614 ms
```

注意 `FAIL typecheck rc=2`——上一站说过，类型检查失败给的是 **2**，不是 1。这一位没被聚合脚本吞掉，靠的是 `execSync` 抛错时 `error.status` 就是子进程真正的退出码。当初为什么不用 `spawnSync`，本机重新验了两种写法：

```text
spawnSync('npm', ['-v'], { shell: true })   → rc=0，stdout 是 11.17.0
        但父进程 stderr 上会多一行：
        [DEP0190] DeprecationWarning: Passing args to a child process with shell option true
        can lead to security vulnerabilities, as the arguments are not escaped, only concatenated.
spawnSync('npm.cmd', ['-v'])  不带 shell     → rc=null，error.code = EINVAL
```

`spawnSync` 加 `shell: true` 结果是对的，可每次起进程都附一行弃用警告，而且那行是**父进程**打出来的、不在返回值的 `stderr` 里；直接叫 `npm.cmd` 又不给 shell，在 Windows 上连进程都起不来。两条都不如 `execSync` 干净。

结尾还有一条小选择：脚本用 `process.exitCode = 1` 而不是 `process.exit(1)`。区别在输出被重定向到文件的时候——`exit()` 会立刻退出，还没写完的 stdout 会丢几行，那张逐项表就缺尾巴。

## 端到端为什么不在名单里

名单里只有六项，`test:e2e` 不在。这一层本轮单独量过：

```text
$ npm run test:e2e -- test/e2e/console-gate.spec.ts
Running 4 tests using 1 worker
3 passed ／ 1 skipped（25.1s）
退出码 0        总耗时 25840 ms
```

**四条断言要 25.8 秒**，因为这层跑之前必须先把项目构建一遍、再把产物服务起起来；上一站量的整层是 172 条用例、25048 毫秒。也就是说，把端到端塞进名单，等于让每次"想快速自查一下"都付一次构建外加浏览器依赖（本机的 Playwright 二进制是几百兆量级）。所以它的去处是 CI 里单独一层，本地要跑就点名。

名单里也没有第三篇补的那个 `check:env`。原因和上面相反：那道校验在没有配环境变量的机器上（比如刚克隆下来的 CI）**必然红**，把它放进聚合入口会把"这项目能不能构建"和"这台机器配没配库"两件事绑死。它是给人跑的诊断命令，不是给流水线跑的门槛。

## 这条命令背后那套 CLI

前面所有 `nuxt` 开头的命令都来自 `@nuxt/cli`——它就是那个叫 `nuxt` 的命令，负责起开发服务、构建、脚手架、管模块，也是 `nuxt` 包的依赖，所以任何 Nuxt 项目里天然有它。这一站顺手把它身上几件和本地/CI 一致性有关的事量了：

```text
$ npx nuxt --version
3.37.0

node_modules/@nuxt/cli/package.json 的 bin：nuxi / nuxi-ng / nuxt / nuxt-cli
                                     四个名字全指向 bin/nuxi.mjs
它的 engines 字段：^16.14.0 || >=18.0.0
```

这里出现了**第三套版本口径**。讲 Nuxt CLI 的那一节写的是"这个命令行跑在 Node v22.21+、v24.11+ 或 v26+ 上，并且在这些版本上原生 import TypeScript 配置"；而它自己 `package.json` 的 `engines` 写的是 16.14 起。上一篇量过 `nuxt` 包的 `^22.19.0 || ^24.11.0 || >=26.0.0`——三个数字互不相同，`engines` 默认又只发警告（那是第一篇的内容）。本机 Node v24.11.1 三套都满足，所以看不出问题；一旦有人拿 Node 22.0 来跑，`engines` 不会拦，而 CLI 那一节的要求已经不满足了。

跑法上有两条规矩值得记：在项目里由包管理器执行，跑的是**这个项目依赖的那个版本**（`npm exec nuxt dev`、`pnpm nuxt dev` 都是）；在项目外面或全局安装时用 `npx nuxi`，它会把命令转交给项目里的 `@nuxt/cli`，好让执行的代码和旁边的 Nuxt 版本一致——但三种情况它会退回用自己带的那份：项目里没有 CLI、项目那份早于 v3.26、或者项目那份不认识这条命令。`--cwd` 可以写在命令名前面也可以写在后面（monorepo 里有用），带 `ROOTDIR` 位置参数的命令会把显式的 `--cwd` 当作对它的覆盖。

最坑的一条是拼错命令。那一节承诺：认不出的命令会拿去和已有命令做匹配，认不出的旗标会和这条命令声明的旗标做匹配，交互式终端里给你一条改好的命令，非交互就报成 warning。本机这一版没走出来：

```text
$ npx nuxt prepare --nope     → rc=0，stdout 45 字节（只有那句 Types generated），stderr 0 字节
$ npx nuxt typecheck --nope   → rc=0，输出 0 行
$ npx nuxt preparee           → 屏幕上出现 'nuxt-preparee' 找不到命令的报错，rc=0
```

第三条不是玄学：那一节同时说了，CLI 没有的命令会被当成项目里的 `nuxt-<command>` 二进制去找（`nuxt build-module` 就是这么工作的），于是拼错的命令名被拿去执行了一个不存在的程序，而**退出码是 0**。第一条和第二条是同一件事的另一面：多余的旗标没被报出来。**为什么这一版没按承诺报 warning，本轮没有取证**，不猜原因；但结论对写流水线是有用的：不要把"命令拼错"寄希望于退出码来发现，聚合脚本里的名单要写死、要人复核。

另外两件会影响 CI 日志的：这个命令行每天在后台查一次有没有更新的 Nuxt 版本，查到就在命令跑完之后提一句 `nuxt upgrade`；它读的是你 `.npmrc` 里配的 registry，并把结果缓存到**用户级的 `.nuxtrc`**（上一篇那个 `.nuxtrc` 除了 `setups` 还有这一位）。预发布版本不提示，同一个 minor 里的补丁版本除非你落后五个以上也不提示，而且在 CI、StackBlitz 和输出不是终端的时候直接跳过这一查。要临时关掉用 `NUXT_IGNORE_UPDATE_CHECK=1` 或 `NO_UPDATE_NOTIFIER=1`，要永久关就在用户级 `.nuxtrc` 里加 `updateCheck.enabled=false`。

代理这块有两条硬信息：会走网络的动作（装模块、搜模块、下模板、检查更新）失败时会报"为什么失败"并给一条大概率可行的重试建议，遇到重新签发 TLS 的代理就指 `NODE_EXTRA_CA_CERTS` 指向组织根证书；而 Node 只有在设了 `NODE_USE_ENV_PROXY` 时才读代理环境变量，CLI 会把这一位继续传给它的子进程，所以 `nuxt add` 触发的包管理器安装能看到同一套代理配置。其余像 `nuxt complete <shell>` 出补全脚本、`@nuxt/cli-nightly` 走每晚频道，都属"知道有这回事"的范围。

## 这一步的自查清单

1. 让本地与 CI 跑同一条命令，不要一个跑六项、另一个跑五项；清单只写在 `check.mjs` 一个地方。
2. 顺序按"便宜的先跑"排，把最贵的一项留在中间偏后，这样失败通常花不到它的钱。
3. 保留子进程的真实退出码（`execSync` 抛错时的 `error.status`），别统一压成 1——上一站那条 `rc=2` 就是靠这个才没丢。
4. 结尾用 `process.exitCode`，不要用 `process.exit()`；否则重定向时最后一张表会缺行。
5. 端到端不进快速名单：它要先构建、还要浏览器二进制。想让它当门槛就放 CI 单独一层。
6. 名单写死并复核：本机的 CLI 对拼错的命令和多余的旗标都给的是退出码 0，指望不上。

| 现象 | 先查什么 |
| --- | --- |
| 表里少了几行就退出了 | 正常，它按顺序跑到第一项红就停 |
| 一项红但你不知道哪项红 | 看最后一行 `停止在 <项名>`；没有这行说明脚本被改过 |
| 本地绿、CI 红 | 两边是不是同一条命令；CI 上有没有那台机器独有的环境变量（`check:env` 就是给这件事准备的） |
| 每次 CI 都比本地慢很多 | 是不是把端到端也放进了快速名单；或者缓存放的是冷缓存 |
| 命令拼错却一路绿灯 | 这就是本机 CLI 的行为，只能靠写死名单拦住 |

下一步 [736《工程把关（5）：CI 工作流：把这条命令搬到云端并本地等价验证》](/posts/5e6afc87d39b9fece0df1449c66b1cd4/)：把这条命令搬上 GitHub Actions，顺便处理一件容易忽略的事——工作流里那份步骤清单和本地名单是两处写入口，得合成一处；再说不 push 上去能验到什么。

## 本文依据

- `@nuxt/cli` 是什么、四个 binary 名、`create-nuxt` 与独立 `nuxi` 的分工、Node 版本要求（v22.21+/v24.11+/v26+）与原生 import TS 配置：`https://nuxt.com/docs/4.x/api/commands/overview`
- 项目内由包管理器跑、项目外 `npx nuxi` 转交与三种退回条件、`--cwd` 位置与对 `ROOTDIR` 的覆盖、`--help`／`--version`：`https://nuxt.com/docs/4.x/api/commands/overview`
- 拼错命令与旗标的处理、未知命令按 `nuxt-<command>` 二进制查找（`nuxt build-module` 的机制）：`https://nuxt.com/docs/4.x/api/commands/overview`
- 每日更新检查的读取来源与缓存位置（用户级 `.nuxtrc`）、不提示的三种情形与两个关闭开关、`updateCheck.enabled=false`：`https://nuxt.com/docs/4.x/api/commands/overview`
- 网络失败的解释与建议、`NODE_EXTRA_CA_CERTS`、`NODE_USE_ENV_PROXY` 及其向子进程传递：`https://nuxt.com/docs/4.x/api/commands/overview`
- 补全脚本与 nightly 频道（本篇只提及）：`https://nuxt.com/docs/4.x/api/commands/overview`
- 本篇所有读数来自同一个仓库的连续多次运行（Nuxt 4.5.2、`@nuxt/cli` 3.37.0、Node v24.11.1、npm 11.17.0）；两处故意跑歪用的探针文件与临时启动器探针都已删除，仓库脚本未改动
