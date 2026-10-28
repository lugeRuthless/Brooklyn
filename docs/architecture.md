---
paths:
  - "Shared/**/*.swift"
  - "BrooklynApp/**/*.swift"
  - "BrooklynExtension/**/*"
  - "Canvas/**/*.swift"
  - "BrooklynTests/**/*.swift"
  - "project.yaml"
---

# 架构详情

本文是 CLAUDE.md「架构」「改动前必读」两节的补充。修改代码前先阅读对应部分。

## 播放逻辑 (`BrooklynManager.makePlayerItems`)

```
makePlayerItems
  ├── Customize OFF … 默认。使用全部 75 个动画
  │     → original 放在开头播放 1 次
  │     → 其余随机打乱
  └── Customize ON  … 只使用用户选中的动画
        → 循环次数和随机顺序都遵循 Database 的设置
```

两种模式都由 `LoopPlayer` 做无限循环。实现方式见[仍保留的防御](#仍保留的防御)。

## App Extension 的构成

屏幕保护程序本体是 `Brooklyn.app/Contents/PlugIns/BrooklynExtension.appex`。系统将 appex 作为独立进程启动，并为每个显示器生成 view controller。系统设置中的预览也由 appex 自己绘制。

```
选择与启动    System Settings / WallpaperAgent
                     │  用 pluginkit 的注册信息解析模块
                     ▼
进程          BrooklynExtension.appex (sandboxed)
                     │  Info.plist 的 ScreenSaverViewControllerClass
                     ▼
显示          BrooklynViewController → BrooklynView → LoopPlayer
```

注册信息由 pluginkit 持有，主机应用的 `ExtensionRegistrar` 在启动时更新。

```
启动   registerAndRefresh()
         │
         ├── 若已有注册指向其他路径，则用 pluginkit -r 卸下
         │     → 仅在卸下时 killall WallpaperAgent
         │        （丢弃已解析路径的缓存）
         ├── pluginkit -a <appex 路径>
         ├── pluginkit -e use -i <extension id>
         └── refresh()
               → 若尚未注册，等待 1 秒后再次确认
                  （pkd 有时会异步公开注册）
```

`make install` 也用同样的命令完成注册。开发中 appex 路径发生变化时，用 `make reset` 结束 WallpaperAgent。

cask 更新是对同一路径的替换，WallpaperAgent 会沿用旧的解析结果启动 extension，从而出现黑屏。tap 里的 cask 在 install / upgrade 的 postflight 中重启 WallpaperAgent，以避免该问题。

选择界面的磁贴不会每次都从 appex 读取。legacy 提供方会提取一次 thumbnail，并持续保存在 `$(getconf DARWIN_USER_CACHE_DIR)` 下 `com.apple.wallpaper.extension.legacy/com.apple.wallpaper.legacy.thumbnails/` 的 PNG，以及 `com.apple.wallpaper.agent/com.apple.wallpaper.view-model-cache/extension-com.apple.wallpaper.extension.legacy-screenSaver` 的 view-model 中。即使把 appex 替换到同一路径也不会失效（2026-08 实测）。更换缩略图后，用 `make reset` 清除。

## 生命周期（macOS 26 实测）

```
开始   viewDidAppear                          → startAnimation()
                                                  → 仅首次构建 player
停止   viewWillDisappear                      → stopAnimation()
       com.apple.screensaver.willstop 通知    → stopAnimation()
```

view controller 的生命周期在每次开始和停止时都会可靠送达，因此在这里驱动播放。framework 不会可靠调用 ScreenSaverView 的 startAnimation / stopAnimation，由 view controller 调用。

通知订阅留在 `BrooklynView`。这是 viewWillDisappear 没有到来时的第二条停止路径。两条停止路径都只做 pause。framework 会复用实例，若在这里销毁 player 做收尾，下次就无法开始。

framework 会在进程内多次创建并销毁 principal class（`BrooklynExtension`）的实例。principal 保持无状态。

在虚拟显示器（Duet 等）上，MediaToolbox 会在每次片段切换时输出 `VRP err=-12852`，该显示器的绘制有时会停止（2026-08 实测）。播放本身会继续，日志里只能看到错误持续出现。

### 仍保留的防御

| 实际行为 | 防御 |
| --- | --- |
| `isPreview` 不会传入 | 播放代码不按 isPreview 分支。view controller 传入 false |
| 有时会生成不显示的实例 | player 在首次 startAnimation 时延迟构建。若不显示，则不加载 75 个 AVPlayerItem；被销毁的 player 也能在下次开始时重新组装 |
| 有时会生成尺寸为 0 的实例 | `BrooklynView` 跳过 player 构建 |
| `AVQueuePlayer` 用完队列后会停止 | `LoopPlayer` 监听 `AVPlayerItemDidPlayToEndTimeNotification`，把已结束的条目重新追加到末尾 |
| 结束通知只能以 object: nil 订阅 | 用 `ownedItems` 判断是否为自己拥有的条目。进程内每个显示器的 player 共存，若不判断，彼此的队列会被无限撑大 |

## 设置面板

在系统设置的 Options… 中打开 `BrooklynConfigurationViewController`，通过 NSHostingController + addChild 承载 SwiftUI 的 `ConfigureSheet`。单独使用 NSHostingView 时，ViewBridge 的远程面板不会绘制。

关闭面板的路径全部由 extension 侧实现。在远程面板中，NSApp.keyWindow、sheetParent、presentingViewController 全部为 nil，调用 `configureSheetDidEnd`（private API，与 Apple 官方 saver 相同）是唯一的关闭方式。按 Escape 也不会关闭。

设置仍使用 `ScreenSaverDefaults`。写入位置是 appex 的 sandbox 容器（`~/Library/Containers/dev.nozomiishii.brooklyn.extension/Data/Library/Preferences/ByHost/`）。

## Swift 6 Strict Concurrency

### `@MainActor` 目标类

这些类直接接触 UI 和设置，因此用 MainActor 隔离。

- `BrooklynManager`
- `Database`
- `ConfigureSheetViewModel`
- `ExtensionRegistrar`

只有 `ExtensionRegistrar` 的 pluginkit 调用放在 nonisolated 的 async 函数中，移出 main。

### `NotificationCenter` 观察者

用 `nonisolated(unsafe)` 属性持有。`NSObjectProtocol` token 本身是 thread-safe 的，不要求 actor isolation，因此不必用 `@MainActor` 包起来。在 deinitializer 中释放时，actor 约束会碍事。

### 从通知回调调用 `@MainActor` 方法

使用 `MainActor.assumeIsolated { ... }`。`com.apple.screensaver.*` 通知约定在 MainActor 上分发，不必用 `Task { @MainActor in ... }` 创建 async 上下文。同步 assume MainActor 即可。
