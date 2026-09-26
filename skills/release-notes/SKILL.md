---
name: release-notes
description: 按 Keep a Changelog 与 SemVer 编写 CHANGELOG 和 GitHub Release 正文，并检查版本号的递增是否符合变更。当用户要写发布说明（release notes）、更新 CHANGELOG、准备某个版本的发版文案、问“这一版改了什么”，或提到 Keep a Changelog 时使用；也用于维护分支上旧版本线的发布说明，以及把维护线版本的 CHANGELOG 段落同步到默认分支。
metadata:
  author: Mashiro0619
  version: 1.0.0
---

# release-notes

输出：CHANGELOG.md 中目标版本的一段（Keep a Changelog 1.1.0 格式），以及由这一段派生的 GitHub Release 正文。

## 何时使用

- 用户说“写发布说明”“更新 changelog”“准备 2.4.0 的说明”“这一版改了什么”。

## 何时不用

- 只要提交消息 → commit-message。
- 判断现在能不能发版 → release-preflight。

## 版本与基线

- 版本比较按 SemVer 2.0.0：tag 去掉 `v` 前缀后比较；构建元数据（`+` 之后的部分）不参与比较；不能解析为版本号的 tag 忽略。
- 目标版本：用户指定；否则读项目的版本字段（package.json、pyproject.toml、Cargo.toml、pubspec.yaml 等）。
- 目标提交：用户指定；否则 HEAD。
- 发布线：目标版本的 major.minor。
- 默认分支：`git symbolic-ref --short refs/remotes/origin/HEAD` 的结果去掉 `origin/`；没有这个引用时从 `git ls-remote --symref origin HEAD` 读取；没有远端时取本地的 main 或 master。
- 最高正式版本：全部 tag（`git tag -l`）中不带预发布标识的最高版本。
- 候选 tag：目标提交历史中版本小于目标的正式版本 tag（不含预发布）。用 `git -c versionsort.suffix=- tag --merged <目标提交> --sort=-v:refname` 列出；`versionsort.suffix=-` 让预发布版本（`v2.0.0-rc.1`）排在对应的正式版本之下。这个排序只方便查看，选取以 SemVer 比较为准：tag 有的带 `v`、有的不带，或带构建元数据时，git 的顺序与 SemVer 不同。
- 基线：候选 tag 中版本最高的一个。没有候选时（首次发布）没有基线：变更范围是目标提交的全部历史，不计算所需递增。
- `git tag --no-merged <目标提交>` 中有与目标同一发布线、版本小于目标的正式版本 tag 时，在输出中列出：这些版本的改动不在目标提交的历史里，可能没有合入。
- 所需递增，按基线到目标提交之间的变更判断：
  - 有不兼容变更（标题带 `!`、脚注有 `BREAKING CHANGE` 或 `BREAKING-CHANGE`，或 diff 显示公开接口被删除、改名、改变语义）→ major；
  - 否则有新功能（feat，或 diff 显示新增公开接口）→ minor；
  - 否则 → patch。
  - 目标是 0.y.z 时，不兼容变更 → minor，新功能 → patch。SemVer 对 0.y.z 不作约束，这里沿用 npm 与 Cargo 的 `^` 版本范围约定。
- 所需版本：基线按所需递增加一级（major：X+1.0.0；minor：X.Y+1.0；patch：X.Y.Z+1）。目标版本不低于所需版本即符合；目标版本带预发布标识时（`2.0.0-rc.1`），用它的核心版本（`2.0.0`）比较。

## 先取事实

1. 目标版本、目标提交、基线与所需递增（见上一节）。
2. 变更：
   ```sh
   git log <基线>..<目标提交> --no-merges --format='%h %s%n%b'
   git diff --stat <基线>..<目标提交>
   ```
   没有基线时只运行 `git log <目标提交> --no-merges --format='%h %s%n%b'`。“写到哪一段”中命中的行另外规定了变更范围时，按那一行。写入 `[Unreleased]` 时，工作区中有还没有提交的改动（`git diff HEAD`，以及 `git status --porcelain` 中 `??` 的文件）→ 先列出这些文件，问用户是否计入，得到回答前不写文件：计入时，条目与所需递增都算上这些改动；不计入时都不算，对照“写到哪一段”时也按工作区干净处理。
   涉及公开接口的文件（接口定义、路由、命令行参数、配置项与环境变量、数据库迁移、对外导出的函数）读具体 diff，确认不兼容变更和升级时需要的操作。
3. 写入位置：
   - 项目用工具生成 changelog → 不手写 CHANGELOG.md。判断依据：`.changeset/` 目录；`release-please-config.json`；semantic-release 的 `.releaserc*`、`release.config.*` 或 package.json 的 `release` 字段；`cliff.toml`；`.github/workflows/` 中用到 `release-please-action`、`semantic-release`、`changesets/action` 或 `git-cliff`。
     - changesets：为还没有 changeset 的改动新建 `.changeset/*.md`。
     - release-please、semantic-release、git-cliff 从提交消息生成：不写文件，列出基线以来不符合 Conventional Commits、会被工具漏掉或归错类的提交。
   - 否则以仓库根目录的 `CHANGELOG.md` 为准，不存在就新建。
   - `.github/workflows/` 中由 tag 触发的 workflow 用文件作为 Release 正文（`--notes-file`、`body_path`、`bodyFile`）：
     - 该文件是仓库中提交的文件、不是 CHANGELOG.md，且 workflow 中没有更早的步骤生成它 → Release 正文写入那个文件；workflow 自动追加的内容不写。
     - workflow 从 CHANGELOG.md 截取版本段作为正文 → 版本段就是 Release 正文，不另写文件。
     - 文件由 workflow 中更早的步骤生成 → 不写这个文件。
4. 语言：已有 CHANGELOG.md 时跟随其中已有条目的语言；新建时用 README 的主要语言。Release 正文与 CHANGELOG 同语言。`[Unreleased]`、类型名（Added 等）、`**BREAKING:**` 不翻译。

## CHANGELOG.md

新建文件的模板与完整示例见 [references/formats.md](references/formats.md)。

- 结构：`# Changelog`，一段说明（遵循 Keep a Changelog 与 SemVer），`## [Unreleased]`，各版本段，文件末尾的链接引用。
- 版本段里只有下表的类型；升级步骤写在 Release 正文。
- 版本段标题：`## [X.Y.Z] - YYYY-MM-DD`；撤回的版本：`## [X.Y.Z] - YYYY-MM-DD [YANKED]`。
- 版本段按 SemVer 从高到低排列，维护线的版本插在对应位置，例如 5.2.1、5.2.0、5.1.4、5.1.3。
- 类型按下表顺序，没有条目的类型不写：

| 类型 | 写什么 |
| --- | --- |
| Added | 新功能 |
| Changed | 已有功能的行为变化 |
| Deprecated | 仍可用、计划移除的功能；写替代方式，已确定时写移除版本 |
| Removed | 已移除的功能 |
| Fixed | 修正的错误；写用户遇到的现象 |
| Security | 安全修复；写影响范围；CVE 编号只写提交、依赖的发布说明或安全公告中查得到的；不写利用方法 |

- 链接引用：`[X.Y.Z]: <仓库地址>/compare/<该版本的基线 tag>...<该版本的 tag>`；最早的版本 `[X.Y.Z]: <仓库地址>/releases/tag/<tag>`；`[unreleased]: <仓库地址>/compare/<当前分支上版本号最高的 tag>...HEAD`（`git tag --merged HEAD` 列出当前分支上的 tag）。
- 仓库地址：`git remote get-url origin` 转成网页地址，例如 `git@github.com:owner/repo.git` → `https://github.com/owner/repo`。上面是 GitHub 的链接格式；GitLab 为 `<仓库地址>/-/compare/<a>...<b>` 与 `<仓库地址>/-/releases/<tag>`。没有远端时沿用 CHANGELOG.md 已有链接中的地址；也没有已有链接时不写链接引用，在收尾说明。

写到哪一段：按下表从上往下判断，命中一行即停。

| 情况 | 写法 |
| --- | --- |
| 用户要求撤回某个已发布的版本 | 只在该版本段标题末尾加 ` [YANKED]`，段落内容不改 |
| 用户要把其他分支或 tag 上某个版本的段落同步到默认分支 | 当前分支不是默认分支时，不改文件，请用户提交当前的改动、切换到默认分支后再同步。当前分支是默认分支时：用 `git show <该版本的 tag 或所在分支>:CHANGELOG.md` 取出该版本段，原样插入，位置按版本号，该版本的链接引用一起加；默认分支上已有这一段时整段替换，链接引用不重复加；`[Unreleased]`、`[unreleased]` 链接与版本字段不动；不按当前分支计算基线、所需版本与未合入的 tag。该版本还没有 tag 时照样同步，在收尾说明：它还没有发布，发布前段落或日期有改动时要再同步一次；比较链接在 tag 推送后才能打开 |
| 用户没有指定版本，版本字段对应的 tag 已存在，且该 tag 之后有提交或工作区有改动 | 按暂不发布处理，写入 `## [Unreleased]`；变更范围是 `<该 tag>..<目标提交>`，不用基线。所需递增按这个范围判断（用户在第 2 步同意计入的未提交改动一并算上），收尾给出下一版的所需版本（该 tag 的版本加一级）；用户要发布时，请用户确认版本号，版本字段改好后按“用户要发布目标版本”一行写版本段 |
| 目标版本的 tag 已存在 | 用户问这一版改了什么 → 给出该版本段的内容（没有这一段时，整理基线到该 tag 的变更），不改文件，不写 Release 正文，不做收尾汇报。其他情况 → 先问用户是否要修改已发布版本的说明 |
| 目标版本低于所需版本 | 新条目写入 `[Unreleased]`，不新建也不修改版本段；收尾给出所需版本，请用户确认；版本号确认、版本字段改好后，按“用户要发布目标版本”一行处理 |
| 目标版本已有段落，但还没有对应的 tag | 更新原段落，日期改为今天 |
| 用户要发布目标版本 | 新建 `## [X.Y.Z] - <今天>`，`[Unreleased]` 中已有的条目一起移入，`[unreleased]` 链接改为从新版本开始 |
| 用户只要记录改动、暂不发布 | 按暂不发布处理，写入 `## [Unreleased]` |

新建 CHANGELOG.md 且已有更早的正式版本 tag 时，在上表的写法之外，为每个更早的版本补一段：条目从该版本的提交整理，日期取 `git for-each-ref --format='%(creatordate:short)' refs/tags/<tag>`。更早的版本超过 5 个时，先问用户是全部补写，还是只从目标版本开始记录。

## 条目

- 一条一句，写用户能观察到的变化：能做什么、行为如何、修好了什么现象。不抄提交标题，不写实现方式，不附提交哈希。
- 条目只写能从代码、提交消息或项目文档确认的行为；提交消息或文档说的行为在代码里找不到时，以代码为准，按确认不了处理。确认不了的内容（新配置项的确切语义、是否已经生效、会不会破坏已有用法），条目只写已确认的部分（例如“新增 `X` 配置项，默认 120”）；会不会破坏已有用法确认不了时不标 `**BREAKING:**`（已确认的删除、改名照常标）；其余在收尾列为需要确认，并说明确认后是否需要改成不兼容变更。
- 多个提交属于同一个变化时，合并成一条。
- 不写：只影响内部的改动（refactor、test、ci、style、构建脚本）、文档改动、不改变行为的依赖升级、版本号提交、合并提交。
- 依赖升级修复了漏洞 → Security；提高了运行环境的最低要求 → Changed，按不兼容变更写。
- 不兼容变更放在 Changed 或 Removed，以 `**BREAKING:**` 开头（本 skill 的约定，Keep a Changelog 没有规定），写改了什么、影响谁、怎么迁移。
- 每条只写已发生的变化；兼容性细节写进该条的迁移说明，或 Release 正文的升级步骤。
- Release 页面不使用派生正文时（workflow 用 `--generate-notes` 或 `generate_release_notes` 生成正文，或从 CHANGELOG.md 截取版本段），用户需要做的手动操作写在相关条目的末尾，例如“升级后运行 `<命令>`”。

## Release 正文

- 按暂不发布处理、撤回版本、同步其他分支的版本段时，不写 Release 正文，也不给 Release 命令。
- 内容依次为：该版本段中的各类型（标题与条目原样）；用户需要手动操作时（先备份、运行迁移、修改配置、替换参数），加一节“升级步骤”，编号列出；末尾一行 `**Full Changelog**: <仓库地址>/compare/<基线 tag>...<目标 tag>`，首次发布为 `<仓库地址>/commits/<目标 tag>`。
- 不写版本号标题，Release 页面自带标题。
- 写到哪里：按“先取事实”第 3 步的写入位置。没有 workflow 读取的文件时，正文放在回复里：
  - 发布 workflow 会创建 Release（`gh release create`、`softprops/action-gh-release`、goreleaser）→ 不给创建命令；需要替换自动生成的正文时，给出 `gh release edit <tag> --notes-file <文件>`，在 workflow 创建 Release 之后执行。
  - 没有 workflow 创建 Release → 给出创建命令（只给出，不执行），`<文件>` 是用户保存这段正文的路径：
    `gh release create <tag> --verify-tag --title "<tag>" --notes-file <文件>`
    - 远端还没有这个 tag 时，`--verify-tag` 让命令失败，而不是在默认分支上新建 tag。
    - 目标版本低于最高正式版本时加 `--latest=false`；目标版本是预发布版本时加 `--prerelease`。
- 发布 workflow 用 `--generate-notes` 或 `generate_release_notes` 自动生成正文时，在收尾说明 Release 页面不会使用这份正文。

## 示例

见 [references/formats.md](references/formats.md) 的“完整示例”。

## 收尾汇报

- 基线、目标提交、写入的文件。
- 怎样提交 CHANGELOG.md 的改动：
  - 为当前分支要发布的版本写入了版本段 → 打 tag 前先提交：版本号变更还没有提交时，两者放进同一个 `chore(release): X.Y.Z` 提交；版本号变更已经提交时，CHANGELOG.md 单独提交，type 用 `docs`。
  - 按暂不发布处理 → CHANGELOG.md 与它记录的改动放进同一个提交；这些改动已经提交时，CHANGELOG.md 单独提交，type 用 `docs`。
  - 撤回版本、同步其他分支的版本段 → CHANGELOG.md 单独提交，type 用 `docs`。
- 所需递增与所需版本，以及目标版本是否符合。
- 列为不兼容变更的条目；被移除的功能之前是否在 Deprecated 中出现过。
- 本次为要发布的版本写入了版本段且当前分支不是默认分支时，说明默认分支的 CHANGELOG.md 怎样得到这一段：
  - 目标版本低于最高正式版本（维护线）→ 在默认分支上把这一段按版本号插入，链接引用一起加；只同步 CHANGELOG.md，不同步版本字段。
  - 其他情况 → 当前分支合入默认分支时这一段随之带入，不另外同步；不打算合入时按上一条处理。
- 已有 CHANGELOG.md 与 Keep a Changelog 不一致的地方：类型名、版本段标题格式、缺少链接引用或 `[Unreleased]`、有版本段却没有对应的 tag。已发布的段落不改写。
- 已有版本段按发布日期排列、与版本号顺序不同时，说明本次按版本号插入的位置。
