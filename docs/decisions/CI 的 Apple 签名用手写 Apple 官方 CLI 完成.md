---
status: accepted
date: 2026-08-12
---

# CI 的 Apple 签名用手写 Apple 官方 CLI 完成

## 背景与问题

在发布 CI 中[加入](https://github.com/nozomiishii/Brooklyn/pull/141) Developer ID 签名和公证时，Apple 没有提供面向 GitHub Actions 的官方工具。官方抽象只有 Xcode GUI 和 Xcode Cloud。

## 考虑过的选项

| 选项 | 评价 |
| --- | --- |
| 手写 `security` + `codesign` + `notarytool` | 内容与 GitHub 官方步骤一致。UTM、alt-tab-macos 等主要 macOS 开源项目也是同一构成 |
| 社区 Action（Apple-Actions/import-codesign-certs 等） | 名称相反，实为 Apple 非官方的 unverified org |
| 开源再实现 rcodesign | 不需要钥匙串，Linux 上也能运行，但自 2024-11 后再无发布 |

## 决定

采用手写的 Apple 官方 CLI。处理签名密钥的环节最敏感，向这个用 SHA pin 运营的仓库再增加第三方依赖，与既定方针相悖。

## 结果

### 改善之处

- 签名环节的依赖降为零，审计范围收束在 workflow 内约 30 行 bash

### 承担的成本

- 自行维护钥匙串流程的样板代码。说明放在[发布流程详情](../release.md)

### 暂缓的议题

- Apple 推出官方 CI 工具后再替换。通过[跟踪 issue](https://github.com/nozomiishii/Brooklyn/issues/140) 关注
