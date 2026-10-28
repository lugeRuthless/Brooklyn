---
status: accepted
date: 2026-08-21
---

# 应用图标切换使用 NSWorkspace.setIcon

## 背景与问题

希望用户能在新旧两个应用图标之间选择。选择要反映到 Finder、Launchpad、Dock，并且在应用退出后仍然保留。主机应用关闭窗口后即退出。

macOS 没有与 iOS `setAlternateIconName` 对应的 API。

## 考虑过的选项

### 切换手段

| 选项 | 评价 |
| --- | --- |
| 切换到 Assets.car 中的另一个 appiconset | 没有手段。AppKit SDK 中没有任何 alternate icon 相关声明，系统遮罩后绘制的只有 `CFBundleIconName` 指向的那一个。改写 Info.plist 会破坏签名 |
| `NSApplication.applicationIconImage` | 只有运行中的 Dock 会变化，退出后恢复。[AppIconKit](https://github.com/danielsaidi/AppIconKit) 的 macOS 实现就是这样做的，每次启动都重新应用 |
| `NSWorkspace.setIcon` | 向 bundle 写入自定义 Finder 图标。[Loop](https://github.com/mrkai77/Loop) 和 [Karabiner-Elements](https://github.com/pqrs-org/Karabiner-Elements) 采用同一方式 |

### 图标形状的持有方式

系统只对 bundle 图标做遮罩，自定义图标会按传入的图像原样绘制。Brooklyn 的素材按 macOS 26 的做法是满版出血，因此要在某处补上圆角和阴影。

| 选项 | 评价 |
| --- | --- |
| 烘焙进素材 | Loop 和 Karabiner 的做法。bundle 图标用和自定义图标用的素材会变成两套 |
| 传入前再绘制 | 素材保持一份满版出血即可。形状比例由自己持有 |

## 决定

使用 `NSWorkspace.setIcon`，圆角和阴影在传入前绘制。比例从 Calculator.app 实测得到。

默认一侧不传入图像，用解除自定义图标来表达。这样只需处理一张图像。

## 结果

### 改善之处

- 图标选择会保留在 Finder、Launchpad、Dock。即使 cask upgrade 整包替换了 bundle，下次启动应用时也会恢复

### 承担的成本

- 应用自定义图标期间，bundle 会带上 `Icon\r` 和 `com.apple.FinderInfo`。对已安装的应用，`codesign --verify --strict` 会失败。普通的 `--verify` 可以通过；恢复默认后两者都会消失
- 同样的内容若附在构建产物上，下一次构建的签名步骤会停住。project.yaml 里放了一个在签名前去掉它们的构建阶段
- 图标形状的比例由自己持有。macOS 改变形状后需要重新实测

### 暂缓的议题

- macOS 出现 alternate app icon 的官方 API 后再替换
