# MAINTENANCE-XW

这个 fork（`li1164267803/codegraph`）自用的维护手册。**仅存在于 `main-xw` 分支**，上游没有这个文件，所以合并上游时永远不会冲突。

上游文档（`CLAUDE.md`、`docs/`）描述的是项目本身；这里只记录**"因为我们是 fork、因为我们本地这台机器"而产生的额外维护事项**——那些不写下来就会在几周后重新踩一遍的坑。

新增条目请沿用下面的格式：**做什么** / **为什么** / **什么时候会咬人**。最后一项最重要——大部分维护坑的代价不是"不会做"，而是"不知道该做了"。

---

## 目录

- [1. 分支模型与上游同步](#1-分支模型与上游同步)
- [2. 本地 Rust kernel 二进制](#2-本地-rust-kernel-二进制)
- [3. 本 fork 自带的补丁](#3-本-fork-自带的补丁)
- [4. 本地安装与回滚](#4-本地安装与回滚)
- [5. 构建依赖：pnpm 与 npm workspaces 的错配](#5-构建依赖pnpm-与-npm-workspaces-的错配)
- [6. 本机测试基线（`npm test` 不是全绿的）](#6-本机测试基线npm-test-不是全绿的)

---

## 1. 分支模型与上游同步

**三层分支：**

| 分支 | 角色 | 规则 |
|---|---|---|
| `main` | 上游的纯净镜像 | **永不在上面提交**，只做快进同步 |
| `main-xw` | 长期自用分支，所有自定义开发 | merge `main` 同步上游，merge `fix/*` 拿改动 |
| `fix/*` | 给上游提 PR 的分支 | 基于 `main`，保持单一干净提交 |

`git log main..main-xw` 就是这个 fork 相对上游多出来的全部内容。

**同步流程**（上游同步在 GitHub 网页端手动做，本地没配 `upstream` remote）：

```bash
git checkout main && git pull        # 永远是快进，因为 main 上没有我们的提交
git checkout main-xw && git merge main
npm run build && npx vitest run
```

**为什么长期分支用 merge 不用 rebase**：`main-xw` 会被日常使用甚至安装到全局，rebase 每次都重写历史、需要 force-push，本地和远端会反复打架。历史图难看是可以接受的代价。

**什么时候会咬人**：只要有人（包括未来的自己）在 `main` 上直接提交过一次，第一步的快进就会失败，从此每次同步都要处理冲突。发现 `git pull` 在 `main` 上产生了 merge commit，就说明这条规矩被破坏了。

---

## 2. 本地 Rust kernel 二进制

1.5.0 起，20 种语言的解析走原生 Rust kernel。**官方发布包自带预编译好的 `.node`，普通用户完全无感**；但**从源码构建只跑 `tsc`，不编译 Rust crate**，所以本地构建默认没有 kernel、全程回落 wasm——1.5.0 的性能提升全部丢失。

**做法：本地编译（2026-09-11 起）。** 这台机器装了 rustup（`rustc 1.98.1`，`--no-modify-path` 装的，所以 PATH 要自己带上）：

```bash
export PATH="$HOME/.cargo/bin:$PATH"
bash scripts/build-kernel.sh
```

产物直接落到 `codegraph-kernel/prebuilds/darwin-arm64/codegraph-kernel.node`。全量编译约 **45 秒**（grammar 是 C 代码，比预想快得多），增量更快。

`codegraph-kernel/prebuilds/` 被 `codegraph-kernel/.gitignore` 忽略，不进版本库。`npm link` 出去的全局 CLI 也能命中这个路径（loader 从 `dist/extraction/kernel/` 上溯三级到仓库根）。

**验证是否生效：**

```bash
CODEGRAPH_KERNEL_DEBUG=1 node -e "console.log(require('./dist/extraction/kernel/loader.js').getKernel() !== null)"
```

2026-09-11 验证通过：`abiVersion=2`、NodeKind(23)/EdgeKind(13)、20 种语言全部在位。

### 曾经的做法（已废弃）：复用官方发布包的二进制

2026-07-27 到 2026-09-11 之间用的是 `npm pack @colbymchenry/codegraph-darwin-arm64@<version>` 解出 `.node` 直接放进 `prebuilds/`。**这条路在"源码跟着上游 main 走"的前提下是死的**，原因见坑 ①。留在这里只是为了说明为什么不要再试一次。

### 什么时候会咬人

**① 预编译二进制只匹配「发布 tag 那一刻」的源码，不匹配 main 上的后续提交。** 这不是版本号对不对得上的问题——2026-09-11 实测：`package.json` 是 1.6.0、npm 上最新也是 1.6.0，但 `main-xw` 跟到了 1.6.0 发布（8/26）之后的 main，而上游在 9/3 的 Expo Router 提交里给 `EDGE_KINDS` 加了第 13 项 `navigates`。1.6.0 的 kernel 只有 12 项 → `verifyContract` 直接拒绝。**只要这个 fork 继续跟 main 走（而不是停在最后一个 release tag），就永远没有现成二进制可用**，必须本地编译。这是改用本地编译的根本原因。

**② 静默失效，症状是"变慢"而不是报错。** loader 的 `verifyContract` 校验三项——`KERNEL_ABI_VERSION`（`src/extraction/kernel/layout.ts`）、NodeKind 表、EdgeKind 表，且是**长度相等 + 逐项相等**的严格比对（`src/extraction/kernel/loader.ts` 的 `sameTable`）。任何一项对不上就**静默忽略 kernel、回落 wasm**。不用 `CODEGRAPH_KERNEL_DEBUG=1` 查的话，很容易把"突然变慢"归因到别处。**每次 merge 上游后都要重编一次 kernel 并验证**——`git diff <上次编译的commit>..HEAD -- src/types.ts src/extraction/kernel/layout.ts` 有输出就必须重编。

**③ 绝对不要用 `cp` 原地覆盖 `.node`。** 2026-09-11 踩到：有常驻 daemon 映射着那个 inode 时，`cp` 原地改写页内容，映射该文件的进程（以及之后加载它的新进程）会在**执行到被改写的页时**被内核 `SIGKILL`。崩溃报告（`~/Library/Logs/DiagnosticReports/node-*.ips`）里是 `EXC_BAD_ACCESS / SIGKILL (Code Signature Invalid)`、`termination: CODESIGNING / Invalid Page`。症状极具误导性——看起来像"下载的二进制坏了"，`codesign -v` 还会报 `valid on disk`。正确顺序：

```bash
pkill -f "codegraph serve --mcp"     # 先断开所有映射
rm -f codegraph-kernel/prebuilds/darwin-arm64/codegraph-kernel.node   # 换 inode
bash scripts/build-kernel.sh          # 或 cp 新文件
```

`scripts/build-kernel.sh` 自己是 `install` 语义（换 inode），所以只要 daemon 杀干净就安全。

**④ 重新 clone 或清理构建产物后要重编一次。** 二进制在 gitignore 目录里，不会跟着仓库走。

**⑤ kernel 在不在会改变测试覆盖面。** 有 15 个测试文件是 kernel 门控的，二进制缺失时自动跳过。2026-07-27 在 1.5.0 上的对照：

| | 测试文件 | 测试用例 |
|---|---|---|
| 有 kernel | **161 passed, 0 skipped** | **2701 passed**, 6 skipped |
| 无 kernel | 146 passed, 15 skipped | 2532 passed, 175 skipped |

看到 `146 passed | 15 skipped` 就宣布"全量通过"的话，实际漏了 169 个用例。**判断全量绿之前先确认 kernel 在位**（`skipped` 是 11 还是 175，一眼就能分辨）。

---

## 3. 本 fork 自带的补丁

### 3.1 解析阶段超大文件 OOM 守卫

提交 `de6a8bf`（合并提交 `f48dc48`）。

`ReferenceResolver.readFileCached()` 跟踪 import 目标时会整读文件。遇到超大资产（真实案例：`import video from "./intro.mp4"`，240MB）会解码成 UTF-8 再正则扫描，打爆 8GB 堆——`codegraph init` 死在 "Resolving refs" 约 65% 处。修复是照抄 extraction 的 `MAX_FILE_SIZE`（1MB）门禁：先 `statSync`，超限就当作读不到（缓存 `null`），让超大文件对解析和对提取一样不可见。

**一处守卫覆盖全部读取方**——主线程、resolver pool worker、C 函数指针合成器都通过 `ctx.readFile` → `readFileCached` 拿文件内容，不要在别处补第二处。

1.5.0 的并行 resolver pool 让问题更严重：每个 worker 各自 `new ReferenceResolver`、各自持有 `fileCache`，同一个超大文件可能被解码多份。

**状态**（2026-09-11 复核）：上游**仍未修复**。PR [#1288](https://github.com/colbymchenry/codegraph/pull/1288) 自 2026-07-15 起一直是 OPEN，只有仓库主的自动化 review bot 发过两条机器审查（风险 🟡 Low，无反对意见），没有人工回复；2026-09-11 在 PR 下 ping 过一次。

复核方法（下次直接照做，别只看版本号）：对着上游 main 建一个临时 worktree，把 `__tests__/resolution-oversized-file.test.ts` 拷进去跑——`readFileCached()` 仍是裸 `readFileSync(..., 'utf-8')`，re-export 那条断言仍然失败。注意 worktree 里要 `rm vitest.workspace.mts` 并 `--config vitest.config.mts`，否则会卡在 svelte 插件解析上（见第 5 节）。

`fix/resolution-oversized-file-oom-v2` 是基于 1.5.0 重开 PR 用的干净分支。

**什么时候会咬人**：将来上游若自己修了这个问题（或合并了 PR 但按 review 意见改过内容），merge `main` 到 `main-xw` 时会在 `readFileCached()` 上冲突。**直接采用上游版本、丢弃我们的即可**——不是问题，只是届时不要慌。如果上游原样合入，git 会发现两边改动完全相同，自动消解不冲突。

---

## 4. 本地安装与回滚

```bash
./scripts/local-install.sh          # 构建 + npm link 到全局
pkill -f "codegraph serve --mcp"    # 杀掉还在跑旧版本的常驻 daemon
codegraph --version
```

MCP 配置和 `UserPromptSubmit` hook 用的都是裸命令 `codegraph`（走 PATH），`npm link` 只是在同一个 nvm bin 目录换符号链接，**路径不变，配置不用改**。

**关于 `preuninstall`**：本包的 `preuninstall` 钩子（`dist/bin/uninstall.js`）会遍历所有 agent target 执行全局 `uninstall()`，也就是从 `~/.claude.json`、Cursor、Codex、opencode 里**删掉 codegraph 的 MCP 配置**。2026-07-27 实测：`npm link` 替换已有全局安装时**不会**触发它（npm 报 `removed 1 package` 但配置完好无损）。不过 `npm uninstall -g` 一定会触发——**手动卸载前先备份那几个配置文件**。

回滚：`./scripts/local-install.sh --undo`

**什么时候会咬人**：

- `--undo` 装回来的是 **npm 上的官方最新版**，**不带第 3 节那些补丁**。
- `~/.claude/settings.json` 里有 `UserPromptSubmit` hook 执行 `codegraph prompt-hook`。链接的构建如果是坏的，**每次提交 prompt 都会受影响**——不只是 codegraph 功能不可用。
- 换版本后各项目的索引仍是旧引擎建的，需要 `codegraph index -f` 重建才能吃到新 grammar 的提取改进（`codegraph status` 会标出落后的索引）。


---

## 5. 构建依赖：pnpm 与 npm workspaces 的错配

**上游仓库是 npm workspaces**（根 `package.json` 的 `workspaces: ["ui"]` + `package-lock.json`），但这台机器的 `node_modules` 是 **pnpm** 装的。**pnpm 只读 `pnpm-workspace.yaml`，不读 `package.json` 的 `workspaces` 字段**，所以根目录的 `pnpm install` **不会**装 `ui/` 的依赖。两边要各装一次：

```bash
pnpm install            # 根
cd ui && pnpm install   # workspace（pnpm 不会替你做）
```

（`Ignored build scripts: esbuild@...` 的警告可以无视，构建照常。）

### ① `npm run build` 现在包含 `build:ui`，缺 ui 依赖会整个失败

`build` = `tsc && copy-assets && build:ui`。`build:ui` 用 vite 把 Svelte viewer 打进 `dist/viewer/`，再由 `scripts/check-ui-build.mjs` 断言产物齐全。**这一步是上游在 2026-07 之后加的**——所以 7 月那次本地构建时它还不存在，`dist/viewer/` 从来没有被本地构建出来过（全局安装的那份也没有）。2026-09-11 第一次跑新 `build` 直接死在 `Cannot find package 'vite'`，就是 `ui/node_modules` 压根不存在。

### ② 根和 `ui/` 各自声明的 `@sveltejs/vite-plugin-svelte` 是**两个不同大版本，且是故意的**

| 位置 | 版本 | 配谁 |
|---|---|---|
| 根 `package.json` | `^4.0.4` | vitest 2.1.9 内置的 **vite 5**，给 `vitest.workspace.mts` 的 `ui` 测试项目用 |
| `ui/package.json` | `^6.2.4` | 浏览器构建的 **vite 7** |

**不要为了"让根能解析到"而把根软链到 `ui/node_modules` 里的那份**。2026-09-11 试过，结果是插件 6 在 vitest 的 vite 5 里跑 `configureServer`，直接 `TypeError: Cannot convert undefined or null to object`。正确做法是在根装它自己声明的 4.x。

### ③ 根测试要用的 `svelte` 根本没在根 `package.json` 里声明

`__tests__/ui-package.test.ts`（`ui` 测试项目那一个文件）`import ... from "svelte"`，但**根 `package.json` 的 devDependencies 里没有 `svelte`**——上游靠 npm workspaces 把 `ui/` 的 svelte **提升（hoist）到根 `node_modules`**，测试才解析得到。pnpm 不做这种提升，于是报 `Failed to resolve import "svelte"`。

本地的修法是手工补上 npm 会做的那个提升：

```bash
ln -sfn "$(cd ui/node_modules/svelte && pwd -P)" node_modules/svelte
```

**这跟 ② 里"不要软链 vite-plugin-svelte"不矛盾**：`svelte` 根本没有自己的版本要求（根没声明），软链到 ui 的那份正是 npm 的等效行为；而 `vite-plugin-svelte` 根**自己声明了不同的大版本**，软链过去就是版本错配。区别就在于"根有没有自己声明"。

修完 `npx vitest run --project ui` → 16 passed。

### 什么时候会咬人

- **`npm test` 会因为这个完全跑不起来，且报错跟测试无关。** 根缺 `@sveltejs/vite-plugin-svelte` 时，vitest 加载 `vitest.workspace.mts` 就失败（`Failed to load url @sveltejs/vite-plugin-svelte`），**连 `--project engine` 也一样**——workspace 文件是整体求值的，engine 测试一个都不会跑。临时只想跑 engine 可以 `npx vitest run --config vitest.config.mts <file>`，绕过 workspace 文件。
- **两条软链在重装依赖后会消失。** `node_modules/svelte`（③）是手工建的，根目录任何一次 `pnpm install` 都可能把它清掉，症状就是 `ui-package.test.ts` 又报 `Failed to resolve import "svelte"`。重建一次即可。
- **merge 上游后根的依赖声明也会变。** 这次 merge 就给根新增了 `@sveltejs/vite-plugin-svelte ^4.0.4`，而 7 月装的 `node_modules` 里没有它。**每次 merge 完，根和 `ui/` 都要重跑一次 `pnpm install`**，不要假设"依赖没动"。


---

## 6. 本机测试基线（`npm test` 不是全绿的）

**2026-09-11 在 `main-xw`（上游 `3ed73bc` + 我们的补丁）、kernel 在位的实测：**

```
Test Files  13 failed | 255 passed (268)
Tests       21 failed | 4579 passed | 11 skipped (4611)
```

**这 21 个失败没有一个是我们的补丁或自编译 kernel 引起的**，逐类核实过：

| 类别 | 数量 | 怎么核实的 | 结论 |
|---|---|---|---|
| 上游 main 同样失败 | 18 | 拿 `main` 建 worktree（共用同一个 kernel）跑同一批文件，失败清单逐条相同 | pre-existing，上游自己的问题 |
| `sync` / `sync-rebuild-convergence` / `multi-repo-workspace` | 3 | 单独跑这 3 个文件：**70 passed** | 全量 268 文件并发下的时序抖动（flaky），不是回归 |
| `ui-package` | 1 | 报 `Failed to resolve import "svelte"` | 本地依赖布局问题，见第 5 节 ③ |

**kernel 不是任何一个失败的原因**：用 kill switch `CODEGRAPH_KERNEL=0` 跑同一批，**得到完全相同的 18 个失败**。

### 那 18 个 pre-existing 失败为什么上游 CI 是绿的

集中在 `ui-steps-*`（6）、`nextjs`（2）、`kernel-dart-parity`（4）、`installer-targets`（2）、`object-literal-methods`、`react-native-bridge`、`mcp-callers-truncation`。其中 `kernel-dart-parity` 比对 kernel 与 wasm 的 Dart 提取结果，差 1 个节点（83 vs 82）——**这类 kernel 门控测试在没有 kernel 的环境里直接 skip**，而上游 CI 的 `npm test` 并不先编译 Rust kernel，所以它在上游从未跑过。换句话说：**我们本地因为编了 kernel，反而比上游 CI 覆盖得更全，也因此看得见上游看不见的失败。**

### 什么时候会咬人

- **看到一片红不要以为是自己弄坏的。** 改动之后先拿这张表比对：失败集合如果和上面一致，就是基线，不是回归。
- **判断回归的正确方法**：`git worktree add --detach /tmp/cg-upstream-main main`，软链根 `node_modules`、`rm vitest.workspace.mts`，然后 `npx vitest run --config vitest.config.mts <失败的文件>` 跑同一批，比对失败清单。
- **`sync` 那三个不要单看全量结果定罪**，它们并发敏感，单独重跑一次再说。
