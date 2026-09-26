---
name: release-preflight
description: 在打 tag 前逐项检查能否发布，给出带依据的检查表和下一步命令，不执行发布。当用户准备发版、问现在能不能发布、要在打 tag 前检查一下，或要做发布前核对（release preflight / pre-release checklist）时使用。
metadata:
  author: Mashiro0619
  version: 1.0.0
---

# release-preflight

输出：一张发布前检查表，每项带状态与依据，FAIL 附修复方法；末尾给出下一步的确切命令，不执行。

## 何时使用

- 用户说“准备发 2.4.0”“能发版了吗”“打 tag 前帮我看看”。

## 何时不用

- 写发布说明 → release-notes。
- 用户要求直接执行发布：本 skill 只检查；打 tag、推送、创建 Release 由用户执行。

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

1. 有远端时先 `git fetch --tags origin`，更新 tag 与远端分支。报告 `would clobber existing tag` 时，本地与远端有同名 tag 指向不同的提交：列入“需要处理”，标 FAIL，修复：核对两边指向的提交，保留正确的一个。
2. 目标版本、目标提交、当前分支（`git rev-parse --abbrev-ref HEAD`）、默认分支、最高正式版本、基线、所需版本（见上一节）；变更列表用 `git log <基线>..<目标提交> --no-merges --format='%h %s%n%b'`，没有基线时用 `git log <目标提交> --no-merges --format='%h %s%n%b'`。
3. `.github/workflows/` 中发版会触发的 workflow，逐个读它的步骤：
   - 推 tag 触发：`on.push.tags` 匹配目标 tag；`on.push` 没有 `branches`、`tags` 过滤；`on.create`。
   - 创建 Release 触发：`on.release`。
4. 输出使用 README 的主要语言；PASS、FAIL、WARN、UNKNOWN 不翻译。

## 状态

| 状态 | 含义 |
| --- | --- |
| PASS | 已核实，符合 |
| FAIL | 阻止发布，附修复方法 |
| WARN | 不阻止发布，需要用户确认 |
| UNKNOWN | 无法核实（缺少工具、权限或远端） |

一项中有多条同时命中时，取最重的状态：FAIL 重于 WARN，WARN 重于 PASS。

## 检查项

按顺序执行，每项记录命令、结果与状态。`'@{u}'` 要带引号：PowerShell 会把不带引号的 `@{u}` 解析成哈希表。

1. **工作区干净**：`git status --porcelain` 无输出 → PASS；否则 FAIL，修复：提交或 `git stash`。
2. **目标提交已推送**：`git rev-parse --abbrev-ref --symbolic-full-name '@{u}'` 失败（没有上游）→ FAIL，修复：`git push -u origin <分支>`；`git merge-base --is-ancestor <目标提交> '@{u}'` 失败 → FAIL，修复：`git push`；否则 PASS。
3. **版本字段**：用户指定了目标版本时检查。版本字段（忽略构建元数据）等于目标版本 → PASS；否则 FAIL，修复：修改版本字段，与 CHANGELOG.md 的版本段一起提交 `chore(release): X.Y.Z`。
4. **发布说明**：
   - 项目用工具生成 changelog（`.changeset/`，或 release-please、semantic-release、git-cliff 的配置或 workflow）→ 只检查工具需要的输入（如 changesets 的 `.changeset/*.md`）；判断不了 → UNKNOWN。
   - CHANGELOG.md 有 `## [X.Y.Z]` 段且至少一条条目 → PASS；该段没有条目 → WARN，请用户确认这一版没有用户可见的变化。
   - CHANGELOG.md 没有该段 → FAIL，修复：用 release-notes 写这一段（条目在 `## [Unreleased]` 时一起移入）。
   - 没有 CHANGELOG.md，也不用工具生成 → FAIL，修复：用 release-notes 新建 CHANGELOG.md。
   - 该段日期不是今天、类型名不在 Added / Changed / Deprecated / Removed / Fixed / Security 之内、缺少该版本的链接引用 → WARN。
   - 发布 workflow 用仓库中提交的文件作为 Release 正文（不是 workflow 中更早的步骤生成的）：该文件不存在或为空 → FAIL；该文件在 `<基线>..<目标提交>` 之间没有改动 → WARN，Release 正文会与上一版相同，请用户确认它是固定内容。workflow 从 CHANGELOG.md 截取版本段作为正文：读截取命令，确认它能截到 `## [X.Y.Z]` 这一段，截不到 → FAIL。
5. **tag 可用**：本地 `git tag -l v<X.Y.Z> <X.Y.Z>` 为空，远端 `git ls-remote --tags origin` 中也没有，且目标版本大于全部 tag（`git tag -l`）中同一发布线上的最高者（含预发布）→ PASS；没有远端时只查本地，在依据中注明。tag 已存在 → FAIL，修复：提升版本号（已发布的版本不能修改，不移动已有 tag）。不大于同线最高 tag → FAIL，修复：提升版本号。
6. **版本递增符合变更**：目标版本不低于所需版本 → PASS；低于 → FAIL，修复：改为所需版本，同时更新版本字段与发布说明。没有基线（首次发布）→ PASS。
7. **没有漏发的版本**：发布说明里有低于目标版本、在全部 tag（`git tag -l`）中都没有对应 tag 的版本段 → WARN，列出这些版本，请用户确认它们是否发布过；否则 PASS。
8. **CI**：`gh run list --commit <目标提交的完整 SHA> --limit 100 --json workflowName,status,conclusion,createdAt`，每个 workflow 只看最新的一次运行：
   - 通过：conclusion 为 `success`、`skipped`、`neutral`；
   - 失败：`failure`、`timed_out`、`startup_failure`；
   - 待定：`cancelled`、`action_required`、`stale`，或 status 还不是 `completed`。

   全部通过 → PASS；有失败 → FAIL，修复：修好后重新推送；有待定，或这个提交没有任何运行记录 → WARN；没有 gh、未登录或没有远端 → UNKNOWN。
9. **发版会触发什么**：列出“先取事实”第 3 步找到的每个 workflow：触发方式（推 tag 或创建 Release）、名字、它做的事（构建、发布包、推镜像、创建 Release、部署）。Release 正文由 `--generate-notes` 或 `generate_release_notes` 自动生成时，注明正文不来自 CHANGELOG。没有这样的 workflow → 写“发版不触发 workflow，Release 需要手动创建”。信息行，不设状态。
10. **latest 与维护线**：下面两件事各自按条件检查，状态取较重的一个；两件事都不需要检查 → PASS。
    - latest 的指向（目标版本是预发布版本，或低于最高正式版本时检查）。第 9 项的 workflow 中，下列步骤会移动 latest：
      - 推送镜像时带 `latest` 标签：显式写了 `:latest`；或用 `docker/metadata-action` 且 `flavor` 没有 `latest=false`。`flavor` 默认为 `latest=auto`：`type=ref,event=tag` 给每个 tag 都加 `latest`；`type=semver` 只给不带预发布标识的版本加，维护线版本也会加。没有 `tags` 输入时，默认的 tags 包含 `type=ref,event=tag`。
      - `npm publish` 没有 `--tag`：npm 10 及更早会把 latest 指向这个版本，npm 11 起拒绝发布。
      - workflow 创建 GitHub Release（`gh release create`、`softprops/action-gh-release`、goreleaser）：维护线版本没有 `--latest=false` 或 `make_latest: false`；预发布版本没有 `--prerelease` 或 `prerelease: true`。

      按目标版本逐个检查这些步骤的行为：
      - 步骤前没有判断，或判断对这个版本不生效（例如只比较 tag 顺序，而预发布 tag 排在最高）→ FAIL，latest 会指向这个版本。修复：维护线版本与预发布版本跳过 latest；npm 用 `npm publish --tag <dist-tag>`，预发布用 `next`，维护线用不能解析为版本范围的名字，例如 `release-3.6`（npm 拒绝 `3.6`、`v3.6` 这类名字）；Release 加 `--latest=false`，预发布版本加 `--prerelease`。
      - 有“版本倒退就退出”一类的判断且对这个版本生效 → 列出退出后被跳过的步骤。只跳过 latest 相关步骤 → WARN：发布内容完整、latest 不变，但这次运行会显示失败；修复：把退出改为只跳过 latest 那一步。还跳过了其他发布步骤 → FAIL，列出这些步骤。
      - 没有这类步骤 → PASS。
    - 修复是否已进入默认分支（目标提交不在默认分支的历史中时检查，即 `git merge-base --is-ancestor <目标提交> origin/<默认分支>` 失败；没有远端时，两条命令都改用本地的 `<默认分支>`）：`git cherry -v origin/<默认分支> <目标提交>` 中以 `+` 开头、不是版本号变更的提交，在默认分支上没有等价提交 → WARN，列出这些提交，提醒移植到默认分支；否则用户从这一版升级到更高版本时会重新遇到这些问题。
11. **项目自带的检查**：package.json scripts 或 Makefile 中有 `preflight`、`prerelease`、`release:check`、`verify`、`check` 一类的命令 → 先读命令内容。只做检查的 → 运行，成功 PASS，失败 FAIL 并附输出。会写文件、改版本号、提交、打 tag、推送或发布的（`npm version`、`npm publish`、`git commit`、`git push` 等）→ 不运行，列为信息行。没有这类命令 → 此项不列出。

## 输出格式

````
## 发布前检查：<项目> <目标版本>（<分支> @ <短 SHA>）

基线：<tag>（<选取说明>）；所需递增：<major|minor|patch>，所需版本 <X.Y.Z>

| # | 检查 | 状态 | 依据 |
| --- | --- | --- | --- |
| 1 | 工作区干净 | PASS | git status --porcelain 无输出 |
| … | | | |

### 需要处理
- [FAIL] <编号> <检查>：<问题>。修复：<命令或操作>
- [WARN] <编号> <检查>：<需要确认的事>

### 下一步（未执行）
```sh
git tag -a v<X.Y.Z> -m "v<X.Y.Z>"
git push origin v<X.Y.Z>
```
````

- tag 名的前缀跟随仓库已有的 tag；没有 tag 时用 `v`。
- 目标提交不是 HEAD 时，tag 命令写上目标提交：`git tag -a v<X.Y.Z> <目标提交> -m "v<X.Y.Z>"`。
- 有 FAIL 时，“下一步”只写“先处理上面的 FAIL，然后重新检查”，不给 tag 命令，也不预先列出修好之后要执行的命令。
- 第 9 项中没有创建 Release 的 workflow 时，“下一步”加一行 `gh release create v<X.Y.Z> --verify-tag --title "v<X.Y.Z>" --notes-file <写有 Release 正文的文件>`；目标版本低于最高正式版本时加 `--latest=false`，是预发布版本时加 `--prerelease`。
- 清单之外发现的阻塞问题或风险（例如发布 workflow 依赖的文件缺失），也列入“需要处理”，标 FAIL 或 WARN。

## 示例

acme-cli 准备在 main 上发布 2.4.0。基线 v2.3.0；v2.3.0 之后有 feat，没有不兼容变更，所需版本 2.4.0。

```
## 发布前检查：acme-cli 2.4.0（main @ 3f2c1a9）

基线：v2.3.0（目标提交历史中低于 2.4.0 的最高正式版本 tag）；所需递增：minor，所需版本 2.4.0

| # | 检查 | 状态 | 依据 |
| --- | --- | --- | --- |
| 1 | 工作区干净 | PASS | git status --porcelain 无输出 |
| 2 | 目标提交已推送 | PASS | HEAD 已在 origin/main 中 |
| 3 | 版本字段 | PASS | package.json 为 2.4.0 |
| 4 | 发布说明 | PASS | CHANGELOG.md 有 [2.4.0] - 2026-09-24，5 条 |
| 5 | tag 可用 | PASS | v2.4.0 本地与远端都不存在；2.4 线上还没有 tag |
| 6 | 版本递增符合变更 | PASS | 所需版本 2.4.0 |
| 7 | 没有漏发的版本 | WARN | CHANGELOG.md 有 [2.3.1]，没有 v2.3.1 |
| 8 | CI | FAIL | ci / test (windows-latest) 失败 |
| 9 | 发版会触发什么 | — | release.yml（推 tag）：npm publish，gh release create --notes-file |
| 10 | latest 与维护线 | PASS | 2.4.0 是正式版本，不低于最高正式版本 2.3.0；目标提交在 main 上 |
| 11 | 项目自带的检查 | PASS | npm run check 退出码 0 |

### 需要处理
- [FAIL] 8 CI：windows-latest 的 test 任务失败。修复：用 `gh run view <run-id> --log-failed` 查看原因，修好后推送。
- [WARN] 7 没有漏发的版本：2.3.1 有版本段但没有 tag，请确认它是否发布过。

### 下一步（未执行）
先处理上面的 FAIL，然后重新检查。
```

## 收尾汇报

输出就是汇报，之后只加两句：FAIL、WARN、UNKNOWN 各有几项；下一步命令能否直接执行。
