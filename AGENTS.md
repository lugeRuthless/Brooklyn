# Repository Guidelines

## 运行确认

| 手段 | 步骤 |
| --- | --- |
| Canvas 目标 | `make generate && open Brooklyn.xcodeproj`。无需安装，在窗口内播放 |
| 真机 | 用 `make install` 安装到 /Applications，再从系统设置的屏幕保护程序列表中选择 |

列表中不出现，或选中后不生效时，先执行 `make reset`。WallpaperAgent 缓存了已解析的路径。

## 架构

### 目标构成

在 project.yaml 中定义。

| 目标 | 作用 |
| --- | --- |
| Brooklyn | 主机应用 (`Brooklyn.app`)。把 extension 放在稳定路径下携带，启动时向 pluginkit 注册 |
| BrooklynExtension | 屏幕保护程序本体的 App Extension。以 `NSExtensionPointIdentifier: com.apple.screensaver` 注册到系统，在独立的 sandboxed 进程中运行 |
| Canvas | 调试用 macOS 应用。在窗口内显示屏幕保护程序 |
| BrooklynTests | 单元测试 |

播放相关源码放在 `Shared/`，BrooklynExtension、Canvas、BrooklynTests 三者构建同一份代码。只有主机应用使用 `BrooklynApp/` 的独立源码。

### 层级构成

```
BrooklynExtension.appex
  BrooklynViewController (ScreenSaverViewController 子类)
    └── BrooklynView (ScreenSaverView)
          └── LoopPlayer (AVQueuePlayer)    # 无限循环播放

  BrooklynConfigurationViewController (设置面板)
    └── ConfigureSheet (SwiftUI)
          └── ConfigureSheetViewModel

  用于组装播放条目。BrooklynView 不持有引用
    BrooklynManager                         # 播放逻辑与选择管理
      ├── Database                          # ScreenSaverDefaults 包装
      └── Animation (enum)                  # 75 种视频定义，rawValue 为文件名

Brooklyn.app
  BrooklynApp (SwiftUI App)
    ├── ContentView
    └── ExtensionRegistrar                  # pluginkit 注册
```

## 改动前必读

容易弄坏的地方。

### private API 依赖

`ScreenSaverExtension` / `ScreenSaverViewController` / `ScreenSaverConfigurationViewController` 是未公开头文件的 private 类。声明位于 BrooklynExtension/PrivateHeaders/ScreenSaverPrivate.h。

Apple 官方 saver 全部采用同一构成，因此比较稳定，但 macOS 更新时仍需做运行确认。选择这一构成的经过见 [停用 saver，改用应用与 ScreenSaver App Extension 分发](<docs/decisions/停用 saver 改用应用与 ScreenSaver App Extension 分发.md>)。

### appex 的实际行为（macOS 26 实测）

变更时用测试确认 regression。

| 实际行为 | 对应 |
| --- | --- |
| ScreenSaverView 的 startAnimation / stopAnimation 不会被 framework 可靠调用 | 在 BrooklynViewController 的 viewDidAppear / viewWillDisappear 中驱动播放 |
| `isPreview` 不会传入 | 播放代码不按 isPreview 分支。view controller 传入 false |
| 有时会生成不显示的实例 | player 在首次 startAnimation 时延迟构建。尺寸为 0 的实例跳过构建 |
| `AVQueuePlayer` 在只有 1 个条目时会停止 | `LoopPlayer` 自动复制 |
| 单独的 NSHostingView 在 ViewBridge 的远程面板中不会绘制 | 要在设置面板中承载 SwiftUI，使用 NSHostingController + addChild |
| 在远程面板中 NSApp.keyWindow 和 sheetParent 都为 nil | 调用 `configureSheetDidEnd` 是唯一的关闭方式。关闭操作全部由 extension 侧实现 |
| WallpaperAgent 会缓存已解析的模块路径和缩略图磁贴 | 要出现在列表中，需要 pluginkit 注册以及对 use 的 election。路径或缩略图变化后执行 `make reset`。cask 的 install / upgrade 中，tap 的 postflight 会重启 WallpaperAgent |

legacy 的 ScreenSaverEngine 不会启动 appex。

### Swift 6 Strict Concurrency

- `BrooklynManager`、`Database`、`ConfigureSheetViewModel`、`ExtensionRegistrar` 为 `@MainActor`
- NotificationCenter 观察者用 `nonisolated(unsafe)` 持有
- 从通知回调调用 `@MainActor` 方法时使用 `MainActor.assumeIsolated`

## Homebrew cask 的同名冲突

homebrew/cask 中已有 pedrommcarrasco/Brooklyn 的 `brooklyn`。裸 token 会解析到那边，因此 install / upgrade / info / uninstall 全部使用完整 token。

```bash
brew upgrade --cask nozomiishii/tap/brooklyn   # 正确
brew upgrade --cask brooklyn                   # 错误。会被替换成另一个项目的 2.1.0
```
