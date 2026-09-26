---
name: commit-message
description: 根据实际改动编写符合 Conventional Commits 的提交消息，并判断是否需要拆分提交。当用户要写提交消息（commit message）、要提交暂存区的改动、问这次改动的提交消息怎么写、问一次改动要不要拆成多个提交，或提到 Conventional Commits 时使用。
metadata:
  author: Mashiro0619
  version: 1.0.0
---

# commit-message

输出：一条符合 Conventional Commits 1.0.0 与 git 提交消息惯例的提交消息，内容只来自这次提交的实际改动；改动包含几件互不依赖的事时，先给出拆分方案。

## 何时使用

- 用户说“写提交消息”“帮我提交”“这次的提交消息怎么写”。
- 用户问一次改动要不要拆开提交。

## 何时不用

- 改写已有提交（rebase、修改已推送的提交）。
- cherry-pick：保留原消息；需要记录来源时用 `git cherry-pick -x`。

## 先取事实

1. `git status --porcelain`：第一列有字母是已暂存，第二列有字母是未暂存，`??` 是未跟踪。
2. 有已暂存内容 → `git diff --cached --stat` 看涉及哪些目录，再按目录分组读 `git diff --cached`。
   没有已暂存内容 → 读 `git diff`，并在输出开头说明“当前没有暂存内容，以下按工作区改动生成”；未跟踪的文件不在 `git diff` 里，列出它们并说明没有计入。
3. 项目明文规定的格式优先于本 skill 的默认规则：
   - commitlint 配置（`commitlint.config.*`、`.commitlintrc*`、package.json 的 `commitlint` 字段）→ type 与 scope 只用配置允许的值；配置了长度上限（如 `header-max-length`，按字符计）时，同时满足这个上限。
   - CONTRIBUTING.md 等贡献指南写明了提交格式 → 按指南。
4. description 与正文使用 README 的主要语言（没有 README 时用与用户对话的语言）；type、scope、`BREAKING CHANGE` 不翻译。

提交历史只用来了解背景，不作为格式依据。

## 标题

```
<type>[(<scope>)][!]: <description>
```

| type | 用于 |
| --- | --- |
| feat | 新增功能 |
| fix | 修正错误行为 |
| perf | 提升性能，行为不变 |
| refactor | 调整代码结构，既不修正错误也不增加功能 |
| style | 只改代码格式（空白、缩进、分号），不影响代码含义；界面样式的调整按 feat 或 fix 判断 |
| test | 只增加或修正测试 |
| docs | 只改文档 |
| build | 构建系统或外部依赖 |
| ci | CI 配置与脚本 |
| chore | 不改源码与测试的其他维护，包括版本号变更 `chore(release): X.Y.Z` |
| revert | 撤销之前的提交，脚注写 `Refs: <被撤销提交的 SHA>` |

- scope：改动集中在一个代码区域时，写该区域在代码里的名字，小写——monorepo 的包名、顶层目录名或模块名，例如 `parser`、`cli`、`api`。改动跨多个区域时省略 scope。
- description：动词开头，写改了什么对象、变成什么样；英文用祈使语气、小写开头；结尾不加句号。
- 长度：整行（含 type 与 scope）以 50 列为宜（git 文档的建议），最多 72 列（常见约定）；一个汉字按 2 列计（本 skill 的约定，与等宽字体中的显示宽度一致）。
- 不兼容变更在冒号前加 `!`，并写 `BREAKING CHANGE:` 脚注。不兼容变更包括：删除或改名公开接口（API、命令行参数、配置项、环境变量），改变其语义使已有用法出错，改变数据格式使旧数据不能直接使用，提高运行环境的最低要求。Conventional Commits 允许有 `!` 时省略脚注；本 skill 两者都写，由脚注说明迁移方法。

## 一次提交一件事

下列任一情况成立时，改动包含多件事：

- 标题需要用“并”“和”“以及”“and”连接两件互不依赖的改动；
- 改动同时符合多个 type（Conventional Commits FAQ：尽量拆成多个提交）。

这时先给拆分方案，再给不拆时的整体消息，由用户决定：

```
建议拆成 3 个提交：
1. fix(parser): 保留引号内的连续空格 —— src/parser/tokenize.ts、test/parser/tokenize.test.ts
2. feat(cli): 增加 --json 输出选项 —— src/cli/options.ts、src/cli/print.ts、docs/cli.md
3. chore(release): 2.4.0 —— package.json

不拆时的整体消息：
（type 取对版本号影响最大的改动：feat 优先于 fix，fix 优先于其他 type；
有不兼容变更时加 `!` 和脚注。其余改动写进正文）
```

- 同一件事的源码、测试、文档放在同一个提交里，不按文件类型拆。
- 版本号变更单独一个提交；CHANGELOG.md 中该版本的段落放进同一个提交。
- 同一个文件里混有两件事时，说明用 `git add -p` 按块暂存。
- 用户选择拆分时，按提交顺序列出每一步的暂存命令（`git restore --staged <文件>`、`git add <文件>`）。

## 正文

- 改动涉及多个文件，或者从标题看不出为什么要改 → 写正文；单个文件里一眼能看懂的改动只写标题。
- 与标题之间空一行。
- 写改之前的问题或需求、为什么这样改、有什么影响或取舍；怎么改的由 diff 说明。
- 问题按改动前的代码陈述，用现在时（“X 时会 Y”），不加“原先”“目前”；改法用祈使语气（“改为……”）。
- 一到三段，每段一到两句。
- 每行不超过 72 列（汉字按 2 列）。

## 脚注

- 与正文之间空一行。每条脚注写成 `Token: value` 或 `Token #value`（Conventional Commits 的脚注格式，仿照 git trailer）；token 中的空格写成 `-`（如 `Reviewed-by`），`BREAKING CHANGE` 除外。
- 不兼容变更：`BREAKING CHANGE: <改了什么、谁受影响、怎么迁移>`，全大写（Conventional Commits 中 `BREAKING-CHANGE` 与它等价，本 skill 统一写 `BREAKING CHANGE`）。
- `Refs: #123`、`Closes #123`、`Co-authored-by: 姓名 <邮箱>` 只写用户提供的信息。

## 执行提交

- 默认只输出消息，不执行 git 写命令。
- 给出了拆分方案、用户还没有选择时，不提交。
- 用户明确要求提交，且暂存区非空：
  - 不需要拆分 → 用这条消息提交；用户选择不拆 → 用整体消息提交。
  - 用户选择拆分 → 暂存区恰好是第一步的改动时，用第一步的消息提交；否则不提交。两种情况都列出其余各步的暂存命令与消息。
  - 提交方式：把消息写入临时文件，`git commit -F <文件>`，然后输出 `git log -1 --stat`。
- 用户要求提交，但暂存区为空 → 输出消息，说明需要先用 `git add` 暂存要提交的文件；不代为暂存。
- 不 push。

## 示例

暂存区：`src/parser/tokenize.ts` 改为先识别引号再按空白切分，`test/parser/tokenize.test.ts` 增加两个用例。README 为中文，没有 commitlint 配置。

```
fix(parser): 保留引号内的连续空格

tokenize 先按空白切分再处理引号，引号内的连续空格会被合并成一个，
路径含多个空格的文件因此无法打开。改为先识别引号再切分。
```

英文项目：

```
fix: prevent racing of requests

When a slow response to an earlier request arrives after the response
to a later one, the list shows stale results. Track the latest request
id and ignore responses to any other request.

Reviewed-by: Z
Refs: #123
```

不兼容变更：

```
feat(config)!: 改用 TOML 格式的配置文件

INI 不支持嵌套结构，按目录设置的规则无法表达。

BREAKING CHANGE: 不再读取 config.ini，使用它的部署都受影响。升级后
运行 acme config migrate，把 config.ini 转换为 config.toml。
```

## 收尾汇报

一句话说明：消息依据的是暂存区还是工作区；是否建议拆分；是否执行了提交；有项目配置或贡献指南时，按它调整了哪些规则。
