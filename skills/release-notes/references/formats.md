# CHANGELOG 与 Release 正文模板

依据 Keep a Changelog 1.1.0 与 SemVer 2.0.0。`[Unreleased]`、类型名与 `**BREAKING:**` 前缀（本 skill 的约定）保留英文。其余文字的语言：已有 CHANGELOG.md 时跟随其中已有的条目，新建时跟随 README。

## 新建 CHANGELOG.md

README 为中文时：

```markdown
# Changelog

本项目的所有重要变更都记录在这个文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循[语义化版本](https://semver.org/lang/zh-CN/)。

## [Unreleased]

## [1.0.0] - 2026-01-15

### Added

- …

[unreleased]: https://github.com/<owner>/<repo>/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/<owner>/<repo>/releases/tag/v1.0.0
```

README 为英文时，说明段使用 Keep a Changelog 的原文：

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
```

## 版本段

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added

- <用户能做的新事情>。

### Changed

- **BREAKING:** <改了什么>。<影响谁>。迁移：<怎么做>。

### Deprecated

- <功能> 已弃用，改用 <替代方式>，将在 <版本> 移除。

### Removed

- <移除的功能>。

### Fixed

- 修复<现象>的问题。

### Security

- <影响范围>（<CVE 编号>）。
```

没有条目的类型整节省略。

## Release 正文

```markdown
### Added

- …

### Fixed

- …

### 升级步骤

1. …
2. …

**Full Changelog**: https://github.com/<owner>/<repo>/compare/<基线 tag>...<目标 tag>
```

“升级步骤”只在用户需要手动操作时出现；正文为英文时写作 “Upgrade steps”。

## 完整示例

syncd 是一个文件同步工具，README 为中文。准备发布 2.0.0，基线 v1.4.2。`git log v1.4.2..HEAD --no-merges` 的标题：

```
feat(sync)!: 默认跳过隐藏文件
feat(sync): 支持用 .syncignore 按目录忽略文件
feat(cli): 用 --interval 取代 --watch-interval
fix(sync): 修复文件名含空格时同步失败
fix(sync): 不跟随指向同步目录外的符号链接
refactor(sync): 拆分上传队列
test(sync): 增加断点续传用例
build: 升级 TypeScript 到 5.6
chore(release): 2.0.0
```

有不兼容变更，所需递增为 major，所需版本 2.0.0，目标版本符合。refactor、test、build、chore 的提交不写入。

CHANGELOG.md 中新增的段落与链接引用：

```markdown
## [Unreleased]

## [2.0.0] - 2026-09-24

### Added

- 可以在任意目录放置 `.syncignore`，按目录忽略文件。
- 新增 `--interval` 参数，设置轮询间隔。

### Changed

- **BREAKING:** 默认不再同步隐藏文件（名称以 `.` 开头的文件和目录）。需要同步隐藏文件的用户受影响。迁移：在 `config.toml` 中设置 `include_hidden = true`。

### Deprecated

- `--watch-interval` 已弃用，改用 `--interval`，将在 3.0.0 移除。

### Fixed

- 修复文件名含空格时同步失败的问题。

### Security

- 修复指向同步目录之外的符号链接会让目录外的文件被上传的问题。

## [1.4.2] - 2026-08-30

…

[unreleased]: https://github.com/example/syncd/compare/v2.0.0...HEAD
[2.0.0]: https://github.com/example/syncd/compare/v1.4.2...v2.0.0
[1.4.2]: https://github.com/example/syncd/compare/v1.4.1...v1.4.2
```

Release 正文先列出与上面相同的各类型，然后：

```markdown
### 升级步骤

1. 需要继续同步隐藏文件时，在 `config.toml` 中设置 `include_hidden = true`。
2. 把脚本中的 `--watch-interval` 改为 `--interval`；`--watch-interval` 在 2.x 中仍可用。

**Full Changelog**: https://github.com/example/syncd/compare/v1.4.2...v2.0.0
```
