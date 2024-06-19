# 完整並發檢查

Incrementally address 資料競爭安全問題，將診斷結果設為警告。

Swift 6 語言模式旨於.incremental遷移，您可以逐步地在您的專案中module-by-module 地址資料競爭安全問題，並启用 compiler 的 actor 隔離和Sendable 檢查警告，以便評估您對於消除資料競爭的進度。

完整資料競爭檢查可以在 Swift 5 語言模式下启用警告，使用-strict-concurrencycompiler flag。

## 使用 Swift编譯器

要在直接在命令行中運行swift或swiftc時启用完整並發檢查，請傳入-strict-concurrency=complete：

```
~ swift -strict-concurrency=complete main.swift
```

## 使用 SwiftPM

### 在 SwiftPM 命令行 Invocation 中

可以在 Swift package manager 命令行 Invocation 中传入-strict-concurrency=complete，使用-Xswiftc flag：

```
~ swift build -Xswiftc -strict-concurrency=complete
~ swift test -Xswiftc -strict-concurrency=complete
```

這樣可以讓您對於資料競爭警告的數量進行評估，然后將flag永久地添加到package manifest中，正如下一節所述。

### 在 SwiftPM package manifesto 中

要启用完整並發檢查 untuk a target in a Swift package 使用 Swift 5.9 或Swift 5.10 工具，請在Swift settings 中使用[`SwiftSetting.enableExperimentalFeature`](https://developer.apple.com/documentation/(packagedescription/swiftsetting/enableexperimentalfeature(_::_:))：

```swift
.target(
    name: "MyTarget",
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency")
    ]
)
```

當使用 Swift 6.0 工具或更晚時，請使用[`SwiftSetting.enableUpcomingFeature`](https://developer.apple.com/documentation/packagedescription/swiftsetting/enableupcomingfeature(_::_:))：

```swift
.target(
    name: "MyTarget",
    swiftSettings: [
        .enableUpcomingFeature("StrictConcurrency")
    ]
)
```

使用 Swift 6 語言模式的目標則無需進行設定變更，完全檢查將自動启用。

## 使用 Xcode

要在 Xcode 項中启用完整並發檢查，請將 "Strict Concurrency Checking" 設定為 "Complete"：

```
// In a Settings.xcconfig
SWIFT_STRICT_CONCURRENCY = complete;
```