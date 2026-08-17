# AGENT.md

本文件面向在这个仓库中工作的 AI 代理，目标是帮助你快速理解项目职责、关键约束和安全修改方式。

## 项目目标

这个仓库用于自动维护 AUR 软件包。

核心流程如下：

1. 读取 `packages/<pkgname>/package.json` 中定义的元数据。
2. 查询对应上游 GitHub Release。
3. 如果发现新版本，更新 `PKGBUILD` 中的 `pkgver`、source 字段和 checksum 字段。
4. 重新生成 `.SRCINFO`。
5. 将更新提交回当前仓库。
6. 使用 GitHub Actions 将更新后的包发布到 AUR。

主工作流文件是 `.github/workflows/update.yml`。

## 关键文件

- `.github/workflows/update.yml`
  GitHub Actions 自动更新、发布到 AUR 和生成包 README 的完整流程。

- `scripts/validate-packages.sh`
  校验所有包的元数据和 `PKGBUILD` 约束。

- `scripts/detect-updates.sh`
  查询 Release、比较版本、更新 `PKGBUILD`、计算 checksum。

- `scripts/commit-updates.sh`
  重新生成 `.SRCINFO`，提交变更并推送。

- `scripts/generate-package-readme.sh`
  为每个包目录生成 README.md，显示包来源、版本等信息。

- `packages/<pkgname>/package.json`
  每个包的更新规则定义。

- `packages/<pkgname>/PKGBUILD`
  AUR 包定义文件。

- `packages/<pkgname>/.SRCINFO`
  由 `makepkg --printsrcinfo` 生成，不应手工维护。

- `README.md`
  面向维护者的中文项目说明。

## 仓库约束

这个仓库当前依赖“结构化元数据 + 轻量文本替换”的方式维护 `PKGBUILD`，因此必须遵守以下约束。

### 目录和命名

- 包目录路径必须是 `packages/<pkgname>`。
- 目录名必须与 `package.json` 中的 `pkgname` 一致。
- `package.json` 中的 `pkgname` 必须与 `PKGBUILD` 中的 `pkgname=` 一致。

### PKGBUILD 约束

- `pkgver` 必须是单行赋值。
- 所有会被自动更新的 source 字段必须是单行赋值。
- 所有会被自动更新的 checksum 字段必须是单行赋值。
- 不要把这些字段改写成多行数组、复杂 shell 拼接或难以匹配的格式，除非你同时更新自动化脚本。
- `.SRCINFO` 是生成产物。如果修改了 `PKGBUILD`，通常也要重新生成 `.SRCINFO`。

### package.json 约束

每个包都必须提供：

- `pkgname`
- `repo`
- `release_strategy`
- `allow_prerelease`
- `tag_to_pkgver`

下载源配置必须二选一：

1. 单 source 模式
2. `sources[]` 多 source 模式

可选字段：

- `tag_pattern`

### 当前支持的策略值

`release_strategy` 仅支持：

- `github_latest`
- `github_beta`

`tag_to_pkgver` 仅支持：

- `identity`
- `strip_v`
- `animeko_beta`
- `surf_beta`

如果你需要新增策略，必须同时更新：

1. `scripts/detect-updates.sh`
2. `scripts/validate-packages.sh`
3. `README.md`
4. 本文件 `AGENT.md`

## 已有自动化能力

### validate-packages.sh

当前会校验：

- `package.json` 是合法 JSON
- 必填字段存在
- `release_strategy` 是支持值
- `tag_to_pkgver` 是支持值
- 单 source 或多 source 配置格式合法
- `sources[].field` 不重复
- `sources[].checksum_field` 不重复
- 所有配置的 source/checksum 字段都真实存在于 `PKGBUILD`
- 包目录名与 `pkgname` 一致
- `PKGBUILD` 中的 `pkgname` 与元数据一致
- `tag_pattern` 如果存在，必须是合法正则
- `makepkg --printsrcinfo` 可成功执行

### detect-updates.sh

当前支持：

- 查询 GitHub latest release
- 查询 GitHub beta release
- `tag_pattern` 过滤 tag
- tag 到 `pkgver` 的归一化
- 多 source 更新
- checksum 自动下载并计算 `sha512`
- 下载重试（404 等永久错误不重试）
- 更清晰的日志输出
- `DRY_RUN=true` 预演模式
- 单包失败隔离：每个包在独立子进程中处理，失败（如资产 404）只跳过该包并回滚其修改，其余包照常更新；失败清单通过 `failed_packages` 输出
- AUR 版本对账：上游无更新时，若 AUR 上的版本落后于仓库（如发布时 AUR 维护导致推送失败），会重新加入发布队列，避免漂移永久卡死

### DRY_RUN

如果你只是想确认会更新什么，而不修改任何文件，可以运行：

```bash
DRY_RUN=true GITHUB_OUTPUT=/tmp/aur-action-output ./scripts/detect-updates.sh
```

注意：dry-run 仍然会访问 GitHub API，并下载实际资源以计算 checksum。

## 修改建议

### 新增软件包时

1. 新建 `packages/<pkgname>/PKGBUILD`
2. 新建 `packages/<pkgname>/package.json`
3. 确保 source 和 checksum 字段名对得上
4. 运行 `./scripts/validate-packages.sh`
5. 如果需要，生成 `.SRCINFO`

### 修改脚本时

优先保持最小改动。

适合当前仓库的修改方式：

- 扩展已有 case 分支
- 增加元数据字段校验
- 增加日志和错误提示
- 增强 dry-run 或验证逻辑

不适合直接做的大改：

- 无充分理由时把 shell 全量重写成别的语言
- 在未同步更新校验逻辑的前提下增加新的元数据字段
- 修改 `PKGBUILD` 模板风格但不更新自动替换逻辑

## 修改后应做的验证

至少执行：

```bash
bash -n scripts/detect-updates.sh
bash -n scripts/validate-packages.sh
bash -n scripts/commit-updates.sh
bash -n scripts/generate-package-readme.sh
./scripts/validate-packages.sh
```

如果改动涉及更新逻辑，建议额外执行：

```bash
DRY_RUN=true GITHUB_OUTPUT=/tmp/aur-action-output ./scripts/detect-updates.sh
```

## 提交前注意事项

- 不要手工改写 `.SRCINFO` 内容，应该通过 `makepkg --printsrcinfo` 生成。
- 不要随意重命名 `packages/<pkgname>` 目录。
- 不要引入当前脚本无法识别的 `PKGBUILD` 字段格式。
- 如果你新增了策略、字段或约束，记得同步更新文档。

## 面向 AI 的工作原则

- 先读工作流和脚本，再修改。
- 默认做最小正确改动。
- 如果只是验证更新逻辑，优先使用 dry-run。
- 如果发现某个包的 `PKGBUILD` 写法与自动更新逻辑冲突，优先修正为仓库统一风格。
- 如果需要引入新的上游版本规则，优先在元数据层表达，无法表达时再扩展脚本。

## 一句话总结

这是一个通过 GitHub Actions 自动跟踪上游 Release、更新 `PKGBUILD` 和 `.SRCINFO` 并发布到 AUR 的轻量自动化仓库；修改时请优先保持模板化、可验证、最小改动。
