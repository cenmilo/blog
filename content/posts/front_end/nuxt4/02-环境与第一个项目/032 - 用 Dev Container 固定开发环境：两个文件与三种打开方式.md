---
url: /posts/d2815035343ec26180b69770b0a9e1dc/
title: 用 Dev Container 固定开发环境：两个文件与三种打开方式
date: 2026-10-22T11:00:00+08:00
updated: 2026-10-22T11:00:00+08:00
author: cmdragon

summary: >-
  摘要：官方的 Dev Containers 页解决的是"我这台机器上跑得好好的"这类问题，成本只有两个文件：在项目根建 .devcontainer/ 目录，放 devcontainer.json 与 Dockerfile。官方那份配置用 FROM node:lts（正好落在 Installation 页"推荐 active LTS"那条上）、用 corepack enable 打开 pnpm、forwardPorts 转发 3000、把 node_modules 挂成 Docker 卷以免容器重启后重装依赖、postStartCommand 在启动后跑安装。已有 dev container 配置的项目有三种打开方式：VS Code 右下角的 Reopen in Dev Containers 提示、命令面板里的 Dev Containers: Reopen in Container、以及面向 CI 的 @devcontainers/cli（devcontainer up --workspace-folder . 与改动 .devcontainer 后的 devcontainer build）。前置条件是 VS Code 的 Dev Containers 扩展加上 Docker Desktop 或 Docker Engine。

categories:
  - 前端开发

tags:
  - nuxt4
  - devcontainer
  - docker
  - 环境
  - vscode

---

<img src="https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg" title="cmdragon_cn.png" alt="cmdragon_cn.png"/>


扫描[二维码](https://api2.cmdragon.cn/upload/cmder/20250304_012821924.jpg)关注或者微信搜一搜：`编程智域 前端至全栈交流与成长`


023 讲前置条件时说得很宽松（"没有 IDE 要求"）。宽松的另一面是：一台机器 Node 22、另一台 Node 24，一个用 pnpm 一个用 npm，然后开始互相甩锅。官方给这类问题准备的解法就是 dev container，而且**它只要两个文件**。

## 官方在什么时候推荐它

Installation 页在起完开发服务之后留了这条 tip：

> 如果你在用 Docker，或者想让开发环境在多台机器上保持一致，考虑使用 dev container。

Dev Containers 页自己的标题也说明了它的定位：

> 为一致的 development environment 而设置或打开一个 Nuxt 项目的 dev container。

关键词是 **consistent（一致）**。这一页解决的不是性能、不是部署，是"两台机器行为不一样"。

## 前置条件：两样

官方列的 prerequisites 只有两条：

> - Visual Studio Code，带 Dev Containers 扩展
> - Docker Desktop 或 Docker Engine

扩展的标识是 `ms-vscode-remote.remote-containers`。**没有 Docker 就没法往下走**（Desktop 或 Engine 二选一），这跟"用不用 Nuxt"无关，是 dev container 本身的依赖。

## 文件一：`.devcontainer/devcontainer.json`

官方原话：

> 在你的项目根目录创建一个 `.devcontainer/` 文件夹，包含这两个文件。

第一份：

```json [devcontainer.json]
{
  "name": "nuxt-devcontainer",
  "build": {
    "dockerfile": "Dockerfile",
    "context": "../"
  },
  "forwardPorts": [3000],
  "portsAttributes": {
    "3000": {
      "label": "Application",
      "onAutoForward": "openPreview"
    }
  },
  "mounts": [
    "type=volume,target=${containerWorkspaceFolder}/node_modules"
  ],
  "postStartCommand": "pnpm install && pnpm dev:prepare"
}
```

逐块看官方给的注释性说明：

> 这份配置使用 Node.js LTS，通过 corepack 启用 pnpm。它为 Nuxt 开发服务器转发端口 3000，并把 `node_modules` 持久化在一个 Docker 卷里，以避免容器重启时重新安装。

三个字段值得单独说：

- **`forwardPorts: [3000]` + `portsAttributes`**：`label` 给这个端口起名"Application"，`onAutoForward: "openPreview"` 表示自动转发时打开预览。3000 这个数字你早就认识了——024 里官方承诺浏览器会打开 `localhost:3000`，029 里生产服务器默认也监听 3000。
- **`mounts` 那一行**：`type=volume,target=${containerWorkspaceFolder}/node_modules`。官方的解释就是上面那句"避免容器重启后重新安装"。**这条对体验影响很大**：没有它，每次重建容器都要重装依赖。
- **`build.context: "../"`**：因为 Dockerfile 在 `.devcontainer/` 里，而 `COPY package.json ...` 引用的路径是相对项目根的，这个 context 就是把它们对上的。

## 文件二：`.devcontainer/Dockerfile`

```dockerfile [Dockerfile]
FROM node:lts

WORKDIR /app

RUN npm i -g corepack && corepack enable

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml  ./
RUN pnpm install --frozen-lockfile

COPY . .
```

五行各自的用途：

| 行 | 作用 |
| --- | --- |
| `FROM node:lts` | 用 Node 的 LTS 镜像 |
| `WORKDIR /app` | 容器里的工作目录 |
| `RUN npm i -g corepack && corepack enable` | 官方说明："通过 corepack 启用 pnpm" |
| `COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./` | 先把三个清单文件拷进去 |
| `RUN pnpm install --frozen-lockfile` | 按锁文件安装，不修改它 |
| `COPY . .` | 再拷其余源码 |

**先拷清单、再装依赖、最后拷源码**——这个顺序是为了让依赖层能被缓存住（改一行代码不必重装依赖）。官方在这里没有明说缓存，但从 `COPY` 的拆分和 `--frozen-lockfile` 的用法可以直接看出这个意图（这句是我的解读）。

顺便把 023 那条前置对上：`FROM node:lts` 正是"推荐 active LTS"那条的落地写法。而 030 讲过的锁文件名字，在这里也出现了（`pnpm-lock.yaml`），并且多出一个 `pnpm-workspace.yaml`——**这份配置默认你是 pnpm 项目**。

## 换包管理器：官方给的是两处一起改

> 要使用别的包管理器，把 `corepack enable` 换成你偏好的管理器（例如 `npm install -g yarn`），并相应地更新 `postStartCommand`。

也就是说改动点是**两个**：Dockerfile 里启用管理器那一行，和 `devcontainer.json` 里的 `postStartCommand`。**别忘了 Dockerfile 那两行 `COPY` 与 `install` 里也写着 pnpm 的文件名**（`pnpm-lock.yaml`、`pnpm install --frozen-lockfile`）——官方那句话没有把它们列出来，但按 030 那张锁文件表，换管理器时它们必然要跟着换（这半句是我按事实推的操作项）。

## 三种打开已有的 dev container

官方明确分了三种场景：

**一、VS Code 提示（最省事）**

> 打开项目时，你应该会在右下角看到一个通知："Reopen in Dev Containers"。点击这个按钮即可构建并在 dev container 中打开项目。

**二、命令面板（你不小心点掉了提示）**

> 1. 打开命令面板（Mac 上 `Cmd+Shift+P`，Windows/Linux 上 `Ctrl+Shift+P`）
> 2. 搜索 **"Dev Containers: Reopen in Container"**
> 3. 选中它

官方补了一句结果：VS Code 会构建容器并重新打开你的项目。

**三、Dev Containers CLI（高级用户或 CI）**

```bash
# Install the CLI (if not already installed)
npm install -g @devcontainers/cli

# Build and open the project in a container
devcontainer up --workspace-folder .

# After making changes to .devcontainer, rebuild
devcontainer build
```

注意官方那行注释：**改完 `.devcontainer` 之后要 `devcontainer build` 重建**。这条对前两种打开方式同样成立——配置文件不是热更新的东西。

## 打开之后跑什么

官方在 Next Steps 就给了一条命令和一个地址：

```bash
pnpm dev
```

> 你的 Nuxt 应用会可用在 <http://localhost:3000>。

## 一处需要你注意的对不上

官方这份配置里的 `postStartCommand` 写的是 `pnpm install && pnpm dev:prepare`。但 024 我们看的那份**最小 `package.json` 里没有 `dev:prepare` 这个脚本**（只有 `build`、`dev`、`generate`、`preview`、`postinstall`）。

原因在别处：**`dev:prepare` 是"开发一个 Nuxt 模块"那条流程里的脚本名**——官方在模块起步页的"下一步"清单第 3 条写的就是 `npm run dev:prepare`（为本地开发准备文件），同一页还有 `npm run dev` 与 `npm run dev:build` 这对 playground 用的脚本（049 会展开这件事）。

所以这份 dev container 配置演示的不是最小应用项目。**落地时要做的动作很明确：确认 `dev:prepare` 在你项目的 `package.json` 里真的存在；不存在就把它换成你有的那条**——应用项目里对应的是 `postinstall` 已经在跑的 `nuxt prepare`（024、035）。**改完在容器里跑一次，看它是否真的执行成功——别把一条从没跑过的命令留在配置里。**

## 自查

- [ ] 你的问题是不是"两台机器不一致"这一类？如果不是，dev container 帮不上你。
- [ ] 前置条件里除了 VS Code 扩展，还需要什么？
- [ ] `mounts` 那一行解决的是什么体验问题？
- [ ] 换成 yarn 时，官方让你改哪两处？还有哪两处按锁文件名也必须改？
- [ ] 改完 `.devcontainer/` 里的文件，第三种方式里官方要求跑什么命令？
- [ ] `postStartCommand` 里那条命令在你项目的 `package.json` 里真的存在吗？

## 下一篇

033 给第一个项目做一次验收：端口对不对、首屏是不是构建产物、报错该按什么顺序读，以及报障时官方让你先收集哪些信息。

## 本文依据

- `guide/best-practices/devcontainers`（标题定位：Set up or open a Nuxt project in a dev container for a consistent development environment；Setting Up 一节"如果你新起 Nuxt 项目并想在 dev container 里开发，可以自己添加配置"；Prerequisites：VS Code + Dev Containers 扩展（ms-vscode-remote.remote-containers）、Docker Desktop 或 Docker Engine；Create the Configuration：在项目根创建 .devcontainer/ 目录含两个文件，devcontainer.json 全文（name nuxt-devcontainer、build.dockerfile Dockerfile、context ../、forwardPorts [3000]、portsAttributes 3000 的 label Application 与 onAutoForward openPreview、mounts type=volume target=${containerWorkspaceFolder}/node_modules、postStartCommand "pnpm install && pnpm dev:prepare"）与 Dockerfile 全文（FROM node:lts、WORKDIR /app、RUN npm i -g corepack && corepack enable、COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./、RUN pnpm install --frozen-lockfile、COPY . .）；说明"该配置使用 Node.js LTS 并通过 corepack 启用 pnpm，转发 3000 端口给 Nuxt dev server，把 node_modules 持久化在 Docker 卷中以避免容器重启时重装"；tip 换包管理器时把 corepack enable 换成偏好管理器（例如 npm install -g yarn）并相应更新 postStartCommand；Opening an Existing Dev Container 三种方式：VS Code 右下角 "Reopen in Dev Containers" 提示、命令面板 Cmd+Shift+P / Ctrl+Shift+P 搜 Dev Containers: Reopen in Container、Dev Containers CLI（npm install -g @devcontainers/cli、devcontainer up --workspace-folder .、改动 .devcontainer 后 devcontainer build）；Next Steps：pnpm dev，应用可用在 http://localhost:3000）
- `getting-started/installation`（dev container tip：使用 Docker 或希望多台机器开发环境一致时考虑 dev container；浏览器自动打开 http://localhost:3000）
- `directory-structure/package`（最小 package.json 的 scripts 仅 build/dev/generate/preview/postinstall 五项，用于对照 postStartCommand 中 dev:prepare 的存在性）
