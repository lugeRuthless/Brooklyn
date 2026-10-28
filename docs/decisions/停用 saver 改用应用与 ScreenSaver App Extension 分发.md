---
status: accepted
date: 2026-08-20
---

# 停用 saver，改用应用与 ScreenSaver App Extension 分发

## 背景与问题

经由 legacyScreenSaver 的 `.saver` 会随每次 macOS 更新而损坏，规避代码[不断增加](https://github.com/nozomiishii/Brooklyn/issues/161)。[无法从 mise 的 brew-cask 安装的问题](https://github.com/nozomiishii/homebrew-tap/issues/63)也源于 cask 的 screen_saver stanza。

## 考虑过的选项

- 维持现状。Aerial 作者[记录了](https://github.com/JohnCoates/Aerial/issues/1396) Tahoe 上 preview 双重启动没有规避办法，情况会继续恶化
- 应用 + ScreenSaver App Extension。与 Aerial 相同的构成。`com.apple.screensaver` extension point 是未公开头文件的 private API，但 Apple 全部官方 saver 都采用同一构成，从 Sonoma 到 Tahoe 一直稳定
- 后台进程 + 全屏应用。不会出现在系统设置列表中，失去屏幕保护程序的体验

## 决定

采用应用 + ScreenSaver App Extension。在 macOS 26 实机 PoC 中，[已验证](https://github.com/nozomiishii/Brooklyn/issues/161#issuecomment-5347557110)列表显示、播放、SwiftUI 设置面板、设置持久化、全屏实际运行全部通过。

private API 依赖仍然存在，但依赖对象变了：从会毫无预告损坏的 legacyScreenSaver，转到 Apple 全部官方 saver 所使用的基础。

## 结果

### 改善之处

- 结束时 viewWillDisappear 会可靠送达。最大的问题 stopAnimation 不触发，由此以正规方式解决
- 分发物变为 Brooklyn.app。cask 可以迁移到 app artifact，也能从 mise 的 brew-cask 安装

### 承担的成本

- 自行维护 PrivateHeaders/ScreenSaverPrivate.h 中的 private 声明。macOS 更新时需要做运行确认

### 暂缓的议题

无
