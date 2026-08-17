# PaloMiku AUR Action

这个仓库用于根据上游 GitHub Release 自动更新选定的 AUR 软件包。

## 工作流

`.github/workflows/update.yml` 中的 GitHub Actions 工作流会按日定时运行，也支持手动触发。

每次执行会完成以下步骤：

1. 校验包元数据和 `PKGBUILD` 文件。
2. 查询上游 GitHub Release。
3. 更新发生变化的 `PKGBUILD` 中的版本号、下载源和校验和字段。单个包处理失败（如上游资产缺失）只会跳过该包并在日志中告警，不影响其余包。
4. 与 AUR 实际版本对账：仓库已更新但 AUR 尚未发布成功（如发布时 AUR 维护）的包会重新进入发布队列。
5. 为更新过的包重新生成 `.SRCINFO`。
6. 将更新后的文件提交并推送回当前仓库。
7. 将每个更新过的包发布到 AUR。
8. 更新每个包目录下的 README.md 页面。

## 仓库结构

- `packages/<pkgname>/PKGBUILD`：发布到 AUR 的包定义文件。
- `packages/<pkgname>/.SRCINFO`：生成出来的 AUR 元数据文件。
- `packages/<pkgname>/package.json`：仓库内部使用的更新规则配置。
- `packages/<pkgname>/README.md`：自动生成的包信息展示页面。
- `scripts/validate-packages.sh`：元数据和 `PKGBUILD` 校验脚本。
- `scripts/detect-updates.sh`：Release 检测与 `PKGBUILD` 更新脚本。
- `scripts/commit-updates.sh`：`.SRCINFO` 生成、提交和推送脚本。
- `scripts/generate-package-readme.sh`：为每个包目录生成 README.md 脚本。

## 包元数据格式

每个包目录都必须包含一个 `package.json`，其中以下字段为必填：

- `pkgname`：必须与 `PKGBUILD` 中的 `pkgname` 赋值一致。
- `repo`：上游 GitHub 仓库，格式为 `owner/repo`。
- `release_strategy`：Release 选择策略。
- `allow_prerelease`：是否允许预发布版本参与更新。
- `tag_to_pkgver`：如何将上游 tag 转换为 Arch 使用的 `pkgver`。

下载源配置支持两种形式。

单一 source：

```json
{
  "source_field": "source_x86_64",
  "source_template": "Example-{pkgver}.AppImage::https://github.com/org/repo/releases/download/{tag}/Example-{pkgver}.AppImage",
  "checksum_field": "sha512sums_x86_64"
}
```

多 source：

```json
{
  "sources": [
    {
      "field": "source_x86_64",
      "template": "Example-{pkgver}-x86_64.AppImage::https://github.com/org/repo/releases/download/{tag}/Example-{pkgver}-x86_64.AppImage",
      "checksum_field": "sha512sums_x86_64"
    },
    {
      "field": "source_aarch64",
      "template": "Example-{pkgver}-aarch64.AppImage::https://github.com/org/repo/releases/download/{tag}/Example-{pkgver}-aarch64.AppImage",
      "checksum_field": "sha512sums_aarch64"
    }
  ]
}
```

可选字段：

- `tag_pattern`：用于筛选 tag 名称的正则表达式。

## 支持的 Release 策略

- `github_latest`：使用上游仓库的 latest release 接口。
- `github_beta`：搜索近期 GitHub Release，选取第一个匹配 `tag_pattern` 的 tag；如果未配置 `tag_pattern`，默认匹配 `beta`。

## 支持的 Tag 转换策略

- `identity`：保持 tag 原样不变。
- `strip_v`：将 `v1.2.3` 转换为 `1.2.3`。
- `animeko_beta`：将 Animeko 的 beta tag 规范化为 Arch 兼容的 `pkgver`。
- `surf_beta`：将 Surf 的 beta 和 rc tag 规范化为 Arch 兼容的 `pkgver`。

## PKGBUILD 约束

这个仓库会通过定向的逐行替换方式更新 `PKGBUILD`。为了保证自动化稳定运行，请遵守以下约束：

- `pkgver`、source 字段和 checksum 字段都应保持单行定义。
- 字段名应保持稳定，并与 `package.json` 中的配置一致。
- 不要将包目录重命名为 `packages/<pkgname>` 之外的名称。
- 将 `.SRCINFO` 视为生成产物，不要手工维护。

校验脚本会检查每个配置的 source 字段和 checksum 字段是否都存在于对应包的 `PKGBUILD` 中。
它还会拒绝不支持的 `release_strategy` 和 `tag_to_pkgver` 值，以及重复的 `sources[].field` 或 `sources[].checksum_field` 条目。
同时，它要求每个包目录名与 `pkgname` 一致，并验证 `tag_pattern` 是否为可用的正则表达式。

## 本地校验

本地校验依赖 Arch 打包工具，尤其是 `makepkg`，另外还需要 `jq`。

运行：

```bash
./scripts/validate-packages.sh
```

如果你想在接近 CI 的环境中测试更新检测，可以运行：

```bash
GITHUB_OUTPUT=/tmp/aur-action-output ./scripts/detect-updates.sh
```

如果你想预览更新内容而不修改任何 `PKGBUILD`，可以运行：

```bash
DRY_RUN=true GITHUB_OUTPUT=/tmp/aur-action-output ./scripts/detect-updates.sh
```

Dry-run 仍然会查询 Release，并下载资源来计算 checksum，但只会输出将要发生的修改，不会真正写入文件。

如果遇到 GitHub API 限流，可以提供 `GITHUB_TOKEN`。

## 新增软件包

1. 创建 `packages/<pkgname>/PKGBUILD`。
2. 创建 `packages/<pkgname>/package.json`，并按支持的 source 格式填写。
3. 确保 `package.json` 中的 source 字段名和 checksum 字段名与 `PKGBUILD` 中的定义完全一致。
4. 运行 `./scripts/validate-packages.sh`。
5. 提交 `PKGBUILD`、`.SRCINFO` 和 `package.json`。

## 说明

- checksum 是通过下载 `package.json` 中指向的实际 Release 资源计算得到的。
- Release 选择和 tag 规范化规则位于 `scripts/detect-updates.sh`。
- 发布到 AUR 的动作使用 `KSXGitHub/github-actions-deploy-aur`。
