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

这台机器**没有 Rust 工具链**（`~/.cargo`、`~/.rustup`、Homebrew cargo、Xcode 全都没有），`scripts/build-kernel.sh` 用不了。**但不需要装 Rust**——直接复用官方发布包里编译好的二进制：

```bash
VERSION=1.5.0   # 与本地 package.json 的版本保持一致
npm pack @colbymchenry/codegraph-darwin-arm64@$VERSION
tar -xzf colbymchenry-codegraph-darwin-arm64-$VERSION.tgz package/lib/kernel/codegraph-kernel.node
mkdir -p codegraph-kernel/prebuilds/darwin-arm64
cp package/lib/kernel/codegraph-kernel.node codegraph-kernel/prebuilds/darwin-arm64/
```

`codegraph-kernel/prebuilds/` 被 `codegraph-kernel/.gitignore` 忽略，不进版本库。`npm link` 出去的全局 CLI 也能命中这个路径（loader 从 `dist/extraction/kernel/` 上溯三级到仓库根）。

**验证是否生效：**

```bash
CODEGRAPH_KERNEL_DEBUG=1 node -e "console.log(require('./dist/extraction/kernel/loader.js').getKernel() !== null)"
```

2026-07-27 用 1.5.0 的包验证通过：34MB Mach-O arm64，`abiVersion=2`，NodeKind(22)/EdgeKind(12) 表与 `src/types.ts` 逐项一致。

### 什么时候会咬人

**① 静默失效，症状是"变慢"而不是报错。** loader 的 `verifyContract` 校验三项——`KERNEL_ABI_VERSION`（`src/extraction/kernel/layout.ts`）、NodeKind 表、EdgeKind 表。任何一项对不上就**静默忽略 kernel、回落 wasm**。上游发新版后如果动了这三者中任意一个，就必须换对应版本的 `.node`。不用 `CODEGRAPH_KERNEL_DEBUG=1` 查的话，很容易把"突然变慢"归因到别处。

**② 重新 clone 或清理构建产物后要重放一次。** 二进制在 gitignore 目录里，不会跟着仓库走。

**③ kernel 在不在会改变测试覆盖面。** 有 15 个测试文件是 kernel 门控的，二进制缺失时自动跳过：

| | 测试文件 | 测试用例 |
|---|---|---|
| 有 kernel | **161 passed, 0 skipped** | **2701 passed**, 6 skipped |
| 无 kernel | 146 passed, 15 skipped | 2532 passed, 175 skipped |

看到 `146 passed | 15 skipped` 就宣布"全量通过"的话，实际漏了 169 个用例。**判断全量绿之前先确认 kernel 在位。**

---

## 3. 本 fork 自带的补丁

### 3.1 解析阶段超大文件 OOM 守卫

提交 `de6a8bf`（合并提交 `f48dc48`）。

`ReferenceResolver.readFileCached()` 跟踪 import 目标时会整读文件。遇到超大资产（真实案例：`import video from "./intro.mp4"`，240MB）会解码成 UTF-8 再正则扫描，打爆 8GB 堆——`codegraph init` 死在 "Resolving refs" 约 65% 处。修复是照抄 extraction 的 `MAX_FILE_SIZE`（1MB）门禁：先 `statSync`，超限就当作读不到（缓存 `null`），让超大文件对解析和对提取一样不可见。

**一处守卫覆盖全部读取方**——主线程、resolver pool worker、C 函数指针合成器都通过 `ctx.readFile` → `readFileCached` 拿文件内容，不要在别处补第二处。

1.5.0 的并行 resolver pool 让问题更严重：每个 worker 各自 `new ReferenceResolver`、各自持有 `fileCache`，同一个超大文件可能被解码多份。

**状态**：上游截至 1.5.0 **仍未修复**，此前提的 PR 未被合并。`fix/resolution-oversized-file-oom-v2` 是基于 1.5.0 重开 PR 用的干净分支。

**什么时候会咬人**：将来上游若自己修了这个问题（或合并了 PR 但按 review 意见改过内容），merge `main` 到 `main-xw` 时会在 `readFileCached()` 上冲突。**直接采用上游版本、丢弃我们的即可**——不是问题，只是届时不要慌。如果上游原样合入，git 会发现两边改动完全相同，自动消解不冲突。

---

## 4. 本地安装与回滚

```bash
./scripts/local-install.sh          # 构建 + npm link 到全局
pkill -f "codegraph serve --mcp"    # 杀掉还在跑旧版本的常驻 daemon
codegraph --version
```

MCP 配置和 `UserPromptSubmit` hook 用的都是裸命令 `codegraph`（走 PATH），`npm link` 只是在同一个 nvm bin 目录换符号链接，**路径不变，配置不用改**。

回滚：`./scripts/local-install.sh --undo`

**什么时候会咬人**：

- `--undo` 装回来的是 **npm 上的官方最新版**，**不带第 3 节那些补丁**。
- `~/.claude/settings.json` 里有 `UserPromptSubmit` hook 执行 `codegraph prompt-hook`。链接的构建如果是坏的，**每次提交 prompt 都会受影响**——不只是 codegraph 功能不可用。
- 换版本后各项目的索引仍是旧引擎建的，需要 `codegraph index -f` 重建才能吃到新 grammar 的提取改进（`codegraph status` 会标出落后的索引）。
