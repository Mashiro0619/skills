# skills

四个面向仓库日常维护的 Agent Skill：写提交消息、写发布说明、改完代码同步文档、发布前检查。遵循 [Agent Skills](https://agentskills.io) 规范，Claude Code、Codex 等支持该规范的 agent 都能加载。

## 安装

用 skills CLI：

```sh
npx skills add Mashiro0619/skills
```

或者把 `skills/` 下的目录复制或链接到 agent 的 skills 目录：

| Agent | 用户级 | 项目级 |
| --- | --- | --- |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.agents/skills/` | `.agents/skills/` |

## 包含的 skill

### commit-message

按暂存区的实际改动写 Conventional Commits 格式的提交消息；一次改动包含几件互不依赖的事时，先给出拆分方案。默认只输出消息，明确要求提交时才执行 `git commit`。

### release-notes

整理上一个正式版本以来的变更，按 Keep a Changelog 写入 CHANGELOG.md，并生成 GitHub Release 正文；同时检查版本号的递增是否符合 SemVer。

### sync-docs

改完代码后主动检查文档：根据 diff 中的环境变量、参数、接口和默认值，找到受影响的 README、使用说明和配置表，经用户同意后修改。文档只描述当前行为，不写“目前支持”“已改为”等描述版本变化的说法；代码中无法确认的内容交由用户确认。

### release-preflight

发布前逐项检查：工作区、推送状态、版本字段、发布说明、tag、版本递增、漏发的版本、CI、发版会触发的 workflow、预发布与维护线发布时 latest 的指向、修复是否已进入默认分支，以及项目自带的检查。输出 PASS / FAIL / WARN / UNKNOWN 检查表，并列出下一步命令（不执行）。

release-notes 与 release-preflight 支持同时维护多条发布线的项目（例如 3.7.1 发布后再发布 3.6.8）：以当前分支上的上一个正式版本为基线，CHANGELOG 按版本号排序，并检查发布 workflow 是否会把 latest 指向旧版本或预发布版本。

## 遵循的标准

- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/zh-hans/v1.0.0/)，type 取自 [@commitlint/config-conventional](https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-conventional)
- git 提交消息惯例：`git help commit` 的 DISCUSSION 一节，以及 git 项目的 [SubmittingPatches](https://git-scm.com/docs/SubmittingPatches)
- [Keep a Changelog 1.1.0](https://keepachangelog.com/zh-CN/1.1.0/)
- [语义化版本 2.0.0](https://semver.org/lang/zh-CN/)
- 发布用附注标签（`git help tag`）
- [Google 开发者文档风格指南：Timeless documentation](https://developers.google.com/style/timeless-documentation)

项目有明文规定时按规定：commitlint、changesets、release-please 等工具的配置，CONTRIBUTING.md 中写明的格式，发布 workflow 读取的文件。提交历史和已有文件中的惯用写法不视为明文规定。

## 许可证

[MIT](LICENSE)
