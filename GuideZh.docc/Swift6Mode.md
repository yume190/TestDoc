# 在 Swift 6 語言模式中啟用 concurrency 訪問

確保您的代碼免於資料競爭問題，透過啟用 Swift 6 語言模式。

## 使用 Swift 編譯器

### 在命令行中啟用

使用 `swift` 或 `swiftc` 命令直接在命令行中啟用完整Concurrency檢查，可以傳遞 `-swift-version 6` 參數：
```
~ swift  -swift-version 6 main.swift
```

## 使用 SwiftPM

### 命令行 Invocation

可以在 Swift package manager 命令行中傳遞 `-Xswiftc` flag 和 `-swift-version 6` 參數：
```
~ swift build  -Xswiftc  -swift-version 6
~ swift test  -Xswiftc  -swift-version 6
```

### Package manifesto

在 `Package.swift` 文件中使用 `swift-tools-version` 為 `6.0`，就能夠啟用 Swift 6 語言模式，對於整個package 的語言模式可以通過 `swiftLanguageVersions` 屬性設定。

然而，您現在還是可以根據需要在 per-target 基本上設定語言模式，使用新的 `swiftLanguageVersion` 建立设置：
```swift
// swift-tools-version: 6.0

let package = Package(
    name: "MyPackage",
    products: [...],
    targets: [
        // 使用預設工具語言模式
        .target(name: "FullyMigrated"),
        // 尚未準備好
        .target(name: "NotQuiteReadyYet",
            swiftSettings: [.swiftLanguageVersion(.v5)]
        )
    ]
)
```

## 使用 Xcode

### Build Settings

可以控制 Xcode 項目或標准的語言模式，通過設定“Swift Language Version” build 设置為“6”。

### XCConfig

也可以在 xcconfig 文件中设置 `SWIFT_VERSION` 设置為“6”：

```
// 在 Settings.xcconfig 中
SWIFT_VERSION = 6;
```