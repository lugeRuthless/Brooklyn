---
paths:
  - ".github/workflows/release.yaml"
  - "Makefile"
  - "release-please-config.json"
  - ".release-please-manifest.json"
---

# 发布流程详情

本文是 CLAUDE.md「CI / 发布」一节的补充。修改 `release.yaml`、`.github/workflows/*` 或 Homebrew Cask 相关内容前必须先阅读。

## 作业构成

`release.yaml` 在 push 到 `main` 或 `workflow_dispatch` 时启动。

```
create-draft-release (ubuntu)
  │  outputs: release_created / tag_name
  ▼
upload-assets (macos-26)          仅当 release_created == 'true'
  ├── homebrew-update (macos-26)  同上
  └── release-pr (ubuntu)         除 upload-assets 失败外始终运行
```

- `create-draft-release`：release-please 汇总 conventional commits；若 release PR 已合并，则创建 draft release 和 tag
- `upload-assets`：用 XcodeGen + `make build` + `make test` 构建内置 screen saver extension 的 `Brooklyn.app`，经[签名与公证](#签名与公证)后，把 `Brooklyn.app.zip` 上传到 GitHub Release 并 publish
- `homebrew-update`：用 `brew bump-cask-pr` 为 `nozomiishii/homebrew-tap` 的 `Casks/brooklyn.rb` 创建更新 PR，并用 `gh pr merge --auto --squash` 启用 auto-merge
- `release-pr`：用 release-please 创建或更新下一个 release PR

## 签名与公证

`upload-assets` 在构建后完成分发物。本地构建（`make build` / `make install`）保持 ad-hoc 签名，此处理只在发布 CI 中执行。

```
security import      把 .p12 导入临时钥匙串。在 job 末尾的 cleanup step 中删除
      ▼
codesign             由内向外签名
      │                先签 BrooklynExtension.appex
      │                再签 Brooklyn.app
      ▼
codesign --verify    用 --deep 验证包含嵌套内容。若 appex 签名中
      │                app-sandbox 缺失则 fail
      ▼
notarytool submit    提交用 ditto 打好的 zip。结果不是 Accepted 时
      │                输出 notarytool log 并 fail
      ▼
stapler staple       把票据钉到 bundle 上
      ▼
ditto                把已 staple 的 bundle 重新打成用于分发的 zip
```

`codesign` 使用 `--force --options runtime --timestamp` 调用。启用 hardened runtime 的 `--options runtime` 和 `--timestamp` 都是公证的必要要求。

`--force` 会丢掉已有的 entitlements，因此 appex 要用 `--entitlements` 重新传入 BrooklynExtension/BrooklynExtension.entitlements。app 侧不带 entitlements。

签名 identity 从临时钥匙串自动检测。1Password 中不保存 identity 字符串。

提交用的 zip 是一次性的，分发物由 Package step 从已 staple 的 bundle 重新制作。

钥匙串 import 遵循 [GitHub 官方步骤](https://docs.github.com/zh/actions/how-tos/deploy/deploy-to-third-party-platforms/sign-xcode-applications)。改动了以下 4 点。

- 钥匙串密码不放入 secret，每次执行用 `openssl rand` 生成。该值只在 job 内使用，因此不增加长期 secret
- 没有 provisioning profile 步骤。Developer ID 直接分发不使用它
- `.p12` 文件在 import 后立即删除
- 官方步骤只要求 self-hosted runner 做 cleanup，这里出于防御始终执行

### 1Password vault `apple-developer`

Apple 的密钥放在专用 vault，不放在共享的 `nozomiishii-ci` vault。划分 vault 的标准是「发到同一处的密钥放在同一个 vault」。

- 每个仓库一个 service account（Brooklyn 用的是「Brooklyn」）。只有 Brooklyn 的 service account 能读取 `apple-developer` vault，这样其他仓库的 CI 被入侵时也接触不到签名密钥
- 1Password 的 service account 创建后不能追加 vault 或更改权限（[官方说明](https://www.1password.dev/service-accounts/manage-service-accounts/)）。需要增加可访问的 vault 时，重新创建 service account，并把 `OP_SERVICE_ACCOUNT_TOKEN` 换成新 token
- 密钥泄露或疑似泄露时，轮换该 vault 中的全部 item

| item | 字段 | 内容 |
| --- | --- | --- |
| `developer-id` | `certificate-p12` | Developer ID Application 证书与私钥的 .p12，经 base64 编码的字符串 |
| `developer-id` | `p12-password` | 导出 .p12 时设置的密码 |
| `app-store-connect` | `key-id` | App Store Connect API 密钥的 Key ID |
| `app-store-connect` | `issuer-id` | App Store Connect API 的 Issuer ID |
| `app-store-connect` | `private-key` | API 密钥 .p8 经 base64 编码的字符串。字段中保留含换行的 PEM，避免被破坏 |

### 密钥更新

- Developer ID Application 证书有效期为 5 年。过期后在 Xcode → Settings → Accounts → Manage Certificates 中重新创建，从 Keychain Access 导出 .p12，并更新 `certificate-p12` / `p12-password`。签名 identity 由 workflow 从钥匙串自动检测，无需登记 identity 字符串
- 已公证的分发物在证书失效后仍然有效，由已 staple 的票据验证
- App Store Connect API 密钥没有期限，但 revoke 后要替换 `app-store-connect` item 的 `key-id` / `private-key`

## Homebrew Cask 更新的实现方针

`homebrew-update` 直接调用 Homebrew 官方 CLI `brew bump-cask-pr`。

| 手段 | 取舍 |
| --- | --- |
| `brew bump-cask-pr` | 采用。官方 [Autobump](https://docs.brew.sh/Autobump) 也使用这条主流路径 |
| 交给 Renovate | `homebrew` manager 只支持 Formula，不支持 Cask。[相关讨论](https://github.com/renovatebot/renovate/discussions/32965) 仍为 open，评论数为 0。用 `postUpgradeTasks` 重算 sha256 的做法在 Mend Cloud 上不可用 |
| `mislav/bump-homebrew-formula-action` | 会静默失败。详见下文 |
| 手写 `git push` + `gh pr create` | 能工作，但不符合 Homebrew 生态的常规做法 |

`brew bump-cask-pr` 代为完成以下 4 项。

- 获取新版 tarball 并自动重算 `sha256`
- 按字段更新 `Casks/brooklyn.rb`
- 通过 API 创建 commit 和 PR，并内置重复 PR 检测
- 用 `brew style` 做语法校验

### `mislav/bump-homebrew-formula-action` 的静默失败

`homebrew-tap` 的 `main` 受 GitHub Rulesets 保护。该 action 看到 `branchRes.data.protected === true` 后，会走到在 `update-<file>-<timestamp>` 这个其他分支上创建 commit 的路径。若保持 `create-pullrequest: false`，则既不会变成 PR 也不会合并，只留下孤立分支。

作业以 success 结束，因此不易察觉。实例是被搁置的 `update-git-harvest.rb-1777372050`。

## runner 选择

`homebrew-update` 在 macos-26 上运行，与 `upload-assets` 使用同一 runner。macOS runner 已预装 `brew`，因此不需要 setup 步骤。

也曾试过 Linux runner 加 `Homebrew/actions/setup-homebrew`。`Homebrew/actions` 是 monorepo，没有单独的 tag，会触发 zizmor 的 `stale-action-refs`；即使 pin 到 commit，警告也会继续出现。

## 周边前提

- GitHub App `nozomiishii-ci` 已获得 `homebrew-tap` 的 `contents: write` 与 `pull-requests: write` 权限
- `homebrew-tap` 已启用 `allow_auto_merge: true`
- `homebrew-tap` 的 `main` 由 GitHub Rulesets 要求 4 个 required status checks（`pull-request / validate`、`github-actions / required`、`secret-scan / secretlint`、`GitGuardian Security Checks`）
- 这些检查通过后，GitHub auto-merge 会自动 squash merge 由 `brew bump-cask-pr` 创建的 PR
