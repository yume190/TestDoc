# 常見編譯器錯誤

識別、理解和解決在使用 Swift 並發性時常見的問題。

編譯器提供的數據隔離保證影響所有 Swift 代碼。這意味著，完整的並發檢查可能會暴露潛在的問題，即使是那些不直接使用任何並發語言特性的 Swift 5 代碼也不例外。並且，在 Swift 6 語言模式啟用時，這些潛在問題有可能變成錯誤。

啟用完整檢查後，許多項目可能會出現大量警告和錯誤。不要感到不知所措！大多數問題可以追溯到一小部分根本原因。而這些原因通常是由常見的模式引起的，這些模式不僅易於修復，還能非常有助於理解 Swift 的數據隔離模型。

## 不安全的全局和靜態變量

全局狀態，包括靜態變量，可以在程序的任何地方訪問。這種可見性使它們特別容易受到並發訪問的影響。在數據競爭安全性之前，全局變量模式依賴於程序員謹慎地訪問全局狀態，以避免數據競爭，而不依賴於編譯器的幫助。

> Experiment: 這些代碼示例以包的形式提供。請在 [Globals.swift][Globals] 中自行試用。

[Globals]: https://github.com/apple/swift-migration-guide/blob/main/Sources/Examples/Globals.swift

### 可發送的類型

```swift
var supportedStyleCount = 42
```

這裡，我們定義了一個全局變量。該全局變量既是非隔離的又是可變的，並且可以從任何隔離域訪問。在 Swift 6 模式中編譯上述代碼會產生錯誤消息：

```swift
1 | var supportedStyleCount = 42
  |              |- 錯誤：全局變量 'supportedStyleCount' 不是並發安全的，因為它是非隔離的全局共享可變狀態
  |              |- 註釋：將 'supportedStyleCount' 轉換為 'let' 常量以使共享狀態不可變
  |              |- 註釋：如果 'supportedStyleCount' 僅從主線程訪問，則將其限制為主 actor
  |              |- 註釋：如果所有訪問都受外部同步機制保護，則可以不安全地標記 'supportedStyleCount' 為並發安全
2 |
```

兩個具有不同隔離域的函數訪問此變量會導致數據競爭。在下面的代碼中，`printSupportedStyles()` 可以在主 actor 上運行，並且與從另一個隔離域調用的 `addNewStyle()` 並發運行：

```swift
@MainActor
func printSupportedStyles() {
    print("Supported styles: ", supportedStyleCount)
}

func addNewStyle() {
    let style = Style()

    supportedStyleCount += 1

    storeStyle(style)
}
```

解決這個問題的一種方法是改變變量的隔離。

```swift
@MainActor
var supportedStyleCount = 42
```

變量保持可變，但已隔離到全局 actor。所有訪問現在只能在一個隔離域中進行，並且在 `addNewStyle` 中的同步訪問在編譯時無效。

如果變量是常量且從未被修改，一個簡單的解決方案是將其表達為常量。通過將 `var` 更改為 `let`，編譯器可以靜態禁止修改，保證安全的只讀訪問。

```swift
let supportedStyleCount = 42
```
如果存在一個同步機制以保護該變量，並且這種機制對編譯器是不可見的，可以使用 `nonisolated(unsafe)` 關鍵字禁用所有隔離檢查：

```swift
/// 該值僅在持有 `styleLock` 時訪問。
nonisolated(unsafe) var supportedStyleCount = 42
```

僅當你仔細保護所有對變量的訪問並使用外部同步機制（例如鎖或調度隊列）時，才使用 `nonisolated(unsafe)`。

### 非可發送類型

在上述示例中，變量是 `Int`，一個本質上是 `Sendable` 的值類型。全局引用類型提出了額外的挑戰，因為它們通常不是 `Sendable`。

```swift
class WindowStyler {
    var background: ColorComponents

    static let defaultStyler = WindowStyler()
}
```

這個 `static let` 聲明的問題與變量的可變性無關。問題是 `WindowStyler` 是一個非 `Sendable` 類型，這使得其內部狀態在不同隔離域之間共享不安全。

```swift
func resetDefaultStyle() {
    WindowStyler.defaultStyler.background = ColorComponents(red: 1.0, green: 1.0, blue: 1.0)
}

@MainActor
class StyleStore {
    var stylers: [WindowStyler]

    func hasDefaultBackground() -> Bool {
        stylers.contains { $0.background == WindowStyler.defaultStyler.background }
    }
}
```

這裡，我們看到兩個函數可以並發地訪問 `WindowStyler.defaultStyler` 的內部狀態。編譯器僅允許使用 `Sendable` 類型進行這種跨隔離訪問。一種解決方案是將變量隔離到單個域使用全局 actor，但也可以考慮直接添加 `Sendable` 遵從性。

## 協議遵從性隔離不匹配

協議定義了實現類型必須滿足的要求。Swift 確保協議的客戶端可以以尊重數據隔離的方式與其方法和屬性交互。為此，協議本身及其要求必須指定靜態隔離。這可能會導致協議的聲明和遵從類型之間的隔離不匹配。

這類問題有很多可能的解決方案，但它們通常涉及權衡。選擇合適的方法首先需要理解為什麼會出現不匹配。

> Experiment: 這些代碼示例以包的形式提供。請在 [ConformanceMismatches.swift][ConformanceMismatches] 中自行試用。

[ConformanceMismatches]: https://github.com/apple/swift-migration-guide/blob/main/Sources/Examples/ConformanceMismatches.swift

### 未明確指定的協議

這種問題最常見的形式是協議沒有明確的隔離。在這種情況下，與所有其他聲明一樣，這意味著非隔離。非隔離協議要求可以從任何隔離域中的泛型代碼調用。如果要求是同步的，那麼實現類型的實現訪問 actor 隔離狀態是無效的：

```swift
protocol Styler {
    func applyStyle()
}

@MainActor
class WindowStyler: Styler {
    func applyStyle() {
        // 訪問主 actor 隔離狀態
    }
}
```

上述代碼在 Swift 6 模式下會產生以下錯誤：

```swift
 5 | @MainActor
 6 | class WindowStyler: Styler {
 7 |     func applyStyle() {
   |          |- 錯誤：主 actor 隔離實例方法 'applyStyle()' 無法用於滿足非隔離協議要求
   |          `- 註釋：將 'applyStyle()' 標記為 'nonisolated' 以使此實例方法不隔離到 actor
 8 |         // 訪問主 actor 隔離狀態
 9 |     }
```

協議可能應該隔離，但尚未更新以反映這一點。如果實現類型首先被遷移以添加正確的隔離，就會出現不匹配。

```swift
// 這實際上僅在主 actor 類型中有意義，但尚未更新以反映這一點。
protocol Styler {
    func applyStyle()
}

// 一個實現類型，現在正確隔離，暴露了不匹配。
@MainActor
class WindowStyler: Styler {
}
```

#### 添加隔離

如果協議要求總是從主 actor 調用，添加 `@MainActor` 是最好的解決方案。

有兩種方法可以將協議要求隔離到主 actor：

```swift
// 整個協議
@MainActor
protocol Styler {
    func applyStyle()
}

// 每個要求
protocol Styler {
    @MainActor
    func applyStyle()
}
```

使用全局 actor 屬性標記協議意味著全局 actor 隔離適用於所有協議要求和擴展方法。當遵從項未在擴展中聲明時，還會推斷全局 actor 遵從性。

每個要求的隔離對 actor 隔離推斷影響較小，因為推斷僅適用於該要求的實現。它不會影響協議擴展或實現類型中其他方法的推斷隔離。如果實現類型不需要綁定到相同的全局 actor，應優先使用這種方法。

無論哪種方式，更改協議的隔離會影響實現類型的隔離，並且可能對使用該協議的泛型代碼施加限制。你可以使用 `@preconcurrency` 分階段處理由於添加全局 actor 隔離引起的診斷：

```swift
@preconcurrency @MainActor
protocol Styler {
    func applyStyle()
}
```

#### 異步要求

對於實現同步協議要求的方法，方法的隔離必須完全匹配要求的隔離，或者方法必須是 `nonisolated`，這意味著它可以從任何隔離域調用而不會有數據競爭風險。使要求異步可以在實現類型的隔離上提供更多的靈活性。

```swift
protocol Styler {
    func applyStyle() async
}

@MainActor
class WindowStyler: Styler {
    func applyStyle() async {
        // 訪問主 actor 隔離狀態
    }
}
異步要求將允許從其他 actor 調用，並允許不同的實現類型在不同的隔離域中提供實現。

總之，識別並解決這些常見的編譯器錯誤需要理解 Swift 並發模型及其數據隔離保證。通過改變變量的隔離，確保協議和其實現的一致性，並正確地處理不同隔離域之間的數據共享，可以有效地解決大多數問題。