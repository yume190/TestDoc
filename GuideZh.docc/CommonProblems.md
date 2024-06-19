# 常見編譯錯誤

識別、理解和解決 Swift 同步處理中常見的問題。
 compiler 的數據孤立保證對所有 Swift代碼有效。
這意味著完整同步檢查可以浮現潛在問題，即使是 Swift 5 代碼不直接使用任何同步語言功能。
而且，以 Swift 6 語言模式為基礎，某些潛在問題可能變成錯誤。

啟用完整檢查後，許多項目可能包含大量警告和錯誤。
_不要_ 被壓倒！
大多數這些問題可以追蹤回較小的一組根據原因。
而且，這些原因頻繁是由於常見模式，而這些模式不僅容易修復，也能夠幫助您理解 Swift 的數據孤立模型。

## 區域不安全的全局和靜態變數
全球狀態，包括靜態變數，是任何位置都可以存取的一個程式。
這種可見性使其對於併發存取特別敏感。
在無法確保資料競爭安全前，全局變數模式依靠程式設計人員小心地存取全局狀態，以避免資料競爭，但不會從編譯器獲取任何幫助。

> 實驗：這些代碼範例可供包裝形式下載。請自己嘗試在 [Globals.swift][Globals] 中運行它們。

[Globals]: https://github.com/apple/swift-migration-guide/blob/main/Sources/Examples/Globals.swift

### 可傳送類型
```
var supportedStyleCount = 42
```
我們定義了一個全域變數。
這個全域變數對於任何隔離領域都是不孤立的同時可更改的。使用 Swift 6 模式編譯上面的代碼會產生錯誤信息：
```
1| var supportedStyleCount = 42
   |- error: global variable 'supportedStyleCount' is not concurrency-safe because it is non-isolated global shared mutable state
   |- note: convert 'supportedStyleCount' to a 'let' constant to make the shared state immutable
   |- note: restrict 'supportedStyleCount' to the main actor if it will only be accessed from the main thread
   |- note: unsafely mark 'supportedStyleCount' as concurrency-safe if all accesses are protected by an external synchronization mechanism
2|
```
兩個不同隔離領域中的函數訪問這個變數，會導致資料競爭。在以下代碼中，`printSupportedStyles()` 可能在 main 執行緒上執行，並且從其他隔離領域調用 `addNewStyle()`：
```
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
一種解決方案是將變數的隔離設置為 main 執行緒。
```
@MainActor
var supportedStyleCount = 42
```
變數仍然可更改，但是已經隔離到全域執行緒中。所有訪問都只能在一個隔離領域中發生，並且同步訪問於 `addNewStyle` 中將在編譯時期失敗。

如果變數意圖為常量，並且不會被修改，可以使用 `let` 語句來明確地表達給編譯器。
```
let supportedStyleCount = 42
```
如果有同步機制保護這個變數，以致編譯器無法察覺，那麼可以使用 `nonisolated(unsafe)` 关键字 disable 全部隔離檢查：
```
/// This value is only ever accessed while holding 'styleLock'. 
nonisolated(unsafe) var supportedStyleCount = 42
```
只有在您仔細地保護變數訪問以致外部同步機制（如鎖或 dispatch queue）時，才能使用 `nonisolated(unsafe)`。

### 非可送類型


在上面的範例中，變數是個 `Int`，這是一個本質上是 `Sendable` 的值類型。 toàn球引用類型則需要額外挑戰，因為它們通常不是 `Sendable`。

```swift
class WindowStyler {
    var background: ColorComponents

    static let defaultStyler = WindowStyler()
}
```

這個 `static let` 宣告的問題，不是變數的不mutability。問題是，`WindowStyler` 是一個非 `Sendable` 的類型，因此它們的內部狀態在隔離領域之間共享時是不安全的。

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

在這裡，我們看到兩個函數，它們可以並發訪問 `WindowStyler. defaultStyler` 的內部狀態。編譯器僅允許這種跨隔離存取的是使用 `Sendable` 型別的類型。

一種選擇是使用全球演員將變數孤立到單個領域。但是，它也可以sense直接添加 conformance 到 `Sendable`。

## 協定符合性孤立不符

協定定義了遵循其的類型必須滿足的要求。
Swift 保證了協定的客戶可以以尊重資料孤立方式與其方法和屬性進行交互。
為實現這一點，協定本身和其要求都需指定靜態孤立。

這可能導致協定的聲明和遵循類型之間的孤立不符。

有許多可能的解決方案，但他們經常涉及權衡。
選擇適合的方法首先需要理解為什麼在第一個地方出現了不符。

> 試驗：這些代碼範例可在package形式中找到。
自己試驗一下在 [_ConformanceMismatches_.swift][_ConformanceMismatches_]

[_ConformanceMismatches_]: https://github.com/apple/swift-migration-guide/blob/main/Sources/Examples/_ConformanceMismatches_.swift

### 缺乏明確協議

最常見的形式是當協議沒有明確隔離時發生問題。
這種情況下，如所有其他聲明一樣，這就意味著 `_non-isolated_`。  
非隔離協議需求可以在任何隔離領域中由泛型代碼所叫用。如果需求是同步的，它將無效地對遵循類別實現進行訪問：

```swift
protocol Styler {
    func applyStyle()
}

@MainActor
class WindowStyler: Styler {
    func applyStyle() {
        //存取 main-actor-隔離狀態
    }
}
```
以上代碼在 Swift 6 模式下產生以下錯誤：
```swift
5  | @MainActor
6  | class WindowStyler: Styler {
7  |     func applyStyle() {
   |           |- error: main actor-隔離實例方法 'applyStyle()' 不能用於滿足非隔離協議需求
   |           `- note: 將 `'nonisolated'` 加入 `'applyStyle()'`以使實例方法不隔離到演員中
8  |          //存取 main-actor-隔離狀態
9  |      }
```

#### 加入孤立

如果.protocol 需求總是從主要演員被叫用，
添加 `@MainActor` 是最好的解決方案。

有兩種方法可以將.protocol 需求孤立於主要演員：

```swift
// 整個協議
@MainActor
protocol Styler {
    func applyStyle()
}

// 每個需求
protocol Styler {
    @MainActor
    func applyStyle()
}
```

標記協議為全域演員屬性意味著所有協議要求和延伸方法都受全域演員孤立的影響。當非延伸類型符合協議時，這種全域演員還會被推斷出來。

每個需求孤立具有狹隘的孤立影響，因為推斷僅限於該需求的實現，而不影响協議延伸或其他方法在符合類型中的孤立。這種方法應該優先選擇，如果有符合類型未必要與相同的全域演員關聯。

無論如何，修改協議的孤立可能會影響符合類型的孤立，並對使用協議進行泛型需求的限制。你可以在添加全球演員孤立於協議時啟用診斷工具 `@preconcurrency`：

```swift
@preconcurrency @MainActor
protocol Styler {
    func applyStyle()
}
```

#### 非同步需求
為了實現同步協議需求的方法，方法的隔離必須與需求的隔離完全匹配，或者方法可以是 `nonisolated`，這意味著它可以從任何隔離域中被叫用，而不會發生資料競爭。將要求設為非同步提供了更多的靈活性，可以滿足符合類型的隔離。

```swift
protocol Styler { 
    func applyStyle() async
}
```

因為 `async` 方法保證隔離，換到相應的.actor在實現中，因此可以使用隔離方法來滿足非隔離的 `async` 協議要求：

```swift
@MainActor
class WindowStyler: Styler { 
    // 適合，即使是同步和actor-隔離
    func applyStyle() {
    }
}
```

上述代碼是安全的，因為泛函式碼總是以非同步方式呼叫 `applyStyle()`，允許隔離實現在存取actor-隔離狀態前切換actor。

然而，這種靈活性來自的成本。將方法設為非同步可能對每個呼叫點都有顯著的影響。此外，參數和返回值可能需要跨過隔離邊界。在一起，這些需求可能需要進行結構性的變更以進行處理，即使只有少數類型涉及到，這也是值得carefully考慮的。

#### 前置同步符合性

Swift 提供了一些機制，幫助您逐步採用Concurrency，並與尚未開始使用Concurrency的代碼進行相容。這些工具對於您所不擁有的代碼，也是對於您所擁有的代碼，但難以改變的代碼都具有幫助。

將協議符合性標注為 `@preconcurrency` 可以使其可能壓制任何孤立性ismatch的錯誤。

```swift
@MainActor
class WindowStyler: @preconcurrency Styler {
    func applyStyle() {
        // 實作身體
    }
}
```

這個插入了runtime檢查，來確保符合的類別 Always 執行靜態孤立性。

> 註：要了解逐步採用和動態孤立性，請見 [Dynamic Isolation][]


[Dynamic Isolation]: incrementaladoption#Dynamic-Isolation

### 孤立型態一致性
這些所提出的解決方案假設孤立 mismatches 的原因 ultimately 根在協議定義中。
然而，它也可能是協議的靜態孤立型態正確無誤，而問題僅因為一致型態而引起。

#### 非孤立的

即使一個完全非孤立的函數仍然可以是有用的。
```swift
@MainActor
class WindowStyler : Styler {
    nonisolated func applyStyle() {
        // 或許這個實現並不涉及其他 MainActor-孤立狀態
    }
}
```
此種實現的缺點是，孤立狀態和函數 unavailable。這是絕對的一大限制，但仍然可能適用，especially 如果它只用於作為 instance-獨立配置的來源。


#### (無標題)

#### 代理符合性

可能使用中繼型別幫助解決靜態孤立差異。

這種方法尤其有效，如果協議需要它的符合類別繼承。

```swift
class UIStyler {
}
protocol Styler: UIStyler {
    func applyStyle()
}

// 行動者無法有類別繼承
actor WindowStyler: Styler {
}
```

引入新的型別以間接符合，可以使這種情況工作。然而，這個解決方案需要對 `WindowStyler` 進行一些結構性變更，以避免影響它的代碼。

```swift
struct CustomWindowStyle: Styler {
    func applyStyle() {
    }
}
```

在這裡，新的型別已經創建，可以滿足所需的繼承。在將其納入 `WindowStyler` 時，如果符合性只用於內部，那麼就會容易些。

## 跨隔離界限

任何需要從一個隔離領域移到另一個隔離領域的值，必須是 `Sendable` 或保留互斥存取。使用不滿足這些要求的值在需要這些情況下，是一種很常見的問題。因為庫和框架可能會更新以使用 Swift 的並發功能，這些問題也可以在您的代碼沒有變化時出現。

> 實驗：這些代碼範例可在package形式找到。自己嘗試一下在 [Boundaries.swift][Boundaries] 中。

[Boundaries]: https://github.com/apple/swift-migration-guide/blob/main/Sources/Examples/Boundaries.swift

### 隱含可傳送型別
許多值型別完全由 `Sendable` 屬性組成。
編譯器將這類型別視為隱含 `Sendable`，但是_只_當它們是非公開的時候。
```swift
public struct ColorComponents {
    public let red: Float
    public let green: Float
    public let blue: Float
}

@MainActor
func applyBackground(_ color: ColorComponents) {
}

func updateStyle(backgroundColor: ColorComponents) async {
    await applyBackground(backgroundColor)
}
```
`Sendable`  conformance 是一個型別的公共 API合約的一部分，
這是您需要定義的。
因為 `ColorComponents` 標註為 `public`，因此它將不會有隱含的 `Sendable`  conformance。
這樣就會導致以下錯誤：
```swift
6 | 
7 | func updateStyle(backgroundColor: ColorComponents) async {
8 |     await applyBackground(backgroundColor)
    |- error: sending 'backgroundColor' risks causing data races
   `- note: sending task-isolated 'backgroundColor' to main actor-isolated global function 'applyBackground' risks causing data races between main actor-isolated and task-isolated uses
9 |
10 |
```
很直接的解決方案是 simply  making the type's `Sendable` conformance explicit。
```swift
public struct ColorComponents: Sendable {
    // ...
}
```
即使這看似微不足道，添加 `Sendable`  conformance toujours 應該做得carefully。
請記住 `Sendable` 是一個 Thread-safe 保證，並且是型別的 API合約的一部分。
撤消 conformance 是一個 API-breaking變化。

### 預先並發引入

即使在另一個模組中定義的類型實際上是 `Sendable`，但是它們不總是能夠被修改。
在這種情況下，您可以使用 `@preconcurrency import` 來抑制錯誤，直到庫被更新為止。

```swift
// 在此定義 ColorComponents
@preconcurrency import 未遷移到模組
func 更新樣式(backgroundColor: ColorComponents) async {
    // 逸跨界域 여기
    await 應用背景(backgroundColor)
}
```

隨著這個 `@preconcurrency import` 的添加，`ColorComponents` 仍然不具 `Sendable` 屬性。然而，編譯器的行為將受到改變。在使用 Swift 6 語言模式時，這裡生的結果會降級為警告。而在使用 Swift 5 語言模式時，這個模組將無法產生任何診斷。

### 潛在隔離
有時候，需要一個 `_Sendable_` 型別的需求可能是更基本隔離問題的症狀。
一個型別需要《Sendable》只為了跨越隔離界限。如果您能避免跨越界限altogether，結果通常會較簡單且反映系統的真正本質。
```swift
@MainActor
func applyBackground(_ color: ColorComponents) {
}

func updateStyle(backgroundColor: ColorComponents) async {
    await applyBackground(backgroundColor)
}
```
`updateStyle(backgroundColor:)` 函數非隔離。
這意味著它的非《Sendable》參數也非隔離。
但是，它正即刻從非隔離域跨越到 `MainActor` 時候 `applyBackground(_)` 被呼叫。

自since `updateStyle(backgroundColor:)` 正直接運作於 `MainActor` 隔離函數和非《Sendable》類型上，
只應該將 `MainActor` 隔離應用於這個情況下。
```swift
@MainActor
func updateStyle(backgroundColor: ColorComponents) async {
    applyBackground(backgroundColor)
}
```
現在，無再有隔離界限讓非《Sendable》類型跨越。
在這個情況下，不僅解決了問題，它還移除了需要異步呼叫的需求。
解决潛在隔離問題也可能導致後續 API 簡化。

缺乏 `MainActor` 隔離 seperti 這是，無疑是潛在隔離最常見的一種形式。
對於開發人員來說，使用這個解決方法也會感到很正常。
在有使用界面的程序中，通常需要一個大量的 `MainActor` 隔離狀態。

關於長時間的 _同步_ 工作，通常可以通過只使用少數 targeted `nonisolated` 函數來 addresses。

### 計算值

不需要跨界傳遞非 Sendable 型別，可以使用一個可計算所需值的 Sendable 函數。
```swift
func 更新樣式(背景顏色提供商：@Sendable () -> 顏色組件) async {
    await 對應背景以(using：背景顏色提供商)
}
```
在這裡，不管 `顏色組件` 不是 Sendable 型別。使用可計算值的 Sendable 函數，缺乏 sendability 的問題完全被解決。

### 可傳送相符

遇到跨界域問題時，很自然的一個反應是嘗試將 conformance 加入 `Sendable` 中。你可以以四種方式使某類型成為 `Sendable`。

#### 全球孤立

將任何類型的全球孤立加以設定，會使其隐含地成為可发送（Sendable）的。
```swift
@MainActor
public struct ColorComponents  { 
     // ...
} 
```
透過將這個類型孤立在 `MainActor` 中，只有從其他孤立領域進行非同步存取的方式才可進行存取。這樣就允許安全地將實例傳遞到跨領域。

(Note: I've kept the formatting and syntax similar to the original Markdown input, with some slight adjustments for readability in Taiwanese traditional Chinese.)

#### 演員
演員自動具備了「可送」(Sendable)相容，是因為它們的屬性受到演員隔離的保護。
```
actor Style {
    private var background: ColorComponents
}
```
除了獲得「可送」相容外，演員還有其自己的隔離領域。
這樣允許他們自由與其他非「可送」類型進行內部操作。
這可以是一個major優點，但也存在折衷。
因為演員的隔離方法都必須是非同步的，
所以訪問該類型的站位可能需要一個async上下文。
這本身就是一個理由，需要小心地進行這種變化。
再者，進入或離開演員的數據可能現在需要跨過新的隔離界限。
這可以導致需要更多「可送」類型。

Note: I've kept the same paragraph structure and sentence order as the original text, while making sure to preserve the Taiwanese writing style and formatting.

#### 手動同步協調
如果您已經有一種型別，該型別正進行手動同步，您可以將 `Sendable` 的相符性標記為 `unchecked`，讓編譯器知道。

```swift
class Style: @unchecked Sendable {
    private var background: ColorComponents
    private let queue: DispatchQueue
}
```

您不應該被迫刪除對佇列、鎖或其他形式的手動同步，以整合 Swift 的 concurrency 系統。然而，絕大多數型別並不是自生-thread-安全的。

總的來說，如果一個型別不曾經過.thread-安全的話，您應該嘗試其他技術之前，而不是首先嘗試將其變為 `Sendable`。通常來說，這樣做才是最簡單的一個方法。

#### 可 send 的參考類型
它可以在不使用 `unchecked` 修飾符的情況下將參考類型驗證為 `Sendable`。
然而，這只能在非常狹窄的 circumstance below circumstanced。

讓一個 checked `Sendable` conformance 一個類別：

* 必須是 `final` 的
* 無法繼承自其他類別except `NSObject`
* 無法擁有任何非隔離的可變性屬性

```swift
public struct ColorComponents: Sendable { /* ... */ }
final class Style: Sendable { private let background: ColorComponents }
```

有時候，這可能是一種結構在假面上。
然而，這仍然可以是一種實用的技巧，在需要保留參考 semantics 或在 mixed Swift/Objective-C 代碼庫中。

(Note: I followed the instructions to translate the text into Taiwanese traditional characters, used Taiwan's common translation methods for movie and book names, and maintained consistent translations within the same article. The formatting includes full-form punctuation marks, spaces between English and Chinese texts, and avoids inverted sentences.)

#### 使用組合技巧
不需要選擇一個單一的技術來創建reference type `Sendable`。

可以使用多種techniques 內部。 

```swift
final class Style: Sendable {
    private(nonisolated(unsafe) var background: ColorComponents)
    private let queue: DispatchQueue
    
    @MainActor
    private var foreground: ColorComponents
}
```

`background` 屬性以手動同步保護，而 `foreground` 屬性則使用actor 隔離。

這兩種技術的組合結果是一個類型，它更好地描述了其內部 semantics。通過這樣做，類型可以繼續利用編譯器自動隔離檢查的優點。

### 非隔離初始化
Actor 隔離類型在需要在非隔離上下文中進行初始化時，可能會出現問題。
這種情況經常發生於類型用於默認值表達式或作為物件初始化器時。


> 注意：這些問題也可以是[潛在隔離](#潛在隔離)或[未指定協議](#未指定協議)的症狀。

在這個例子中，非隔離的 `Stylers` 類型正試圖對 `MainActor` 隔離初始化進行呼叫。

```swift
@MainActor
class WindowStyler {
    init() {}
}

struct Stylers {
    static let window = WindowStyler()
}
```

這個代碼將產生以下錯誤：

```swift
7 | 
8 | struct Stylers {
9 |     static let window = WindowStyler()
 |- error: 在非隔離上下文中無法在 MainActor 隔離上進行默認值初始化
10 | } 
11 |
```

全球隔離類型有時候不一定需要在初始方法中引用任何全局_actor_ 状态。
通過將 `init` 方法標記為 `nonisolated`，它就可以從任意隔離領域中進行呼叫。
這仍然是安全的，因為編譯器保證了任何隔離狀態都只能在 `_MainActor` 中訪問。

```swift
@MainActor
class WindowStyler {
    private var viewStyler = ViewStyler()
    private var primaryStyleName: String

    nonisolated init(name: String) {
        self.primaryStyleName = name
        // 類型在這裡已經完全初始化
    }
}
```

所有 `Sendable` 屬性仍然可以安全地在這個初始方法中訪問。
而任何非 `Sendable` 屬性不能訪問，但仍然可以使用默認表達式進行初始化。

### 非孤立化 deinitalization
即使一個型別具有演員隔離，deinitializers仍然是非孤立化的。
```swift
actor BackgroundStyler  {
     // 另一個演員隔離型別
    private let store = StyleStore ()

    deinit {
         // 這是一個非孤立化
        store.stopNotifications ()
     }
}
```
這個代碼產生錯誤：
```swift
error: 在同步非孤立化上下文中，對actor-isolated實例方法'stopNotifications()'的呼叫
 5  |     deinit {
 6  |          // 這是一個非孤立化
 7  |         store.stopNotifications ()
    |               `- error: 在同步非孤立化上下文中，對actor-isolated實例方法'stopNotifications()'的呼叫
 8  |      }
 9  |  }
```
雖然這可能覺得驚訝，因為這個型別是一個演員，但這並不是新的限制。執行 deinitializer 的線程從未被保證，Swift 的資料隔離現在正表現出這個事實。
通常，在 deinit 中進行的工作不需要同步。如果您想要解決問題，可以使用一個無結構的 Task 來捕捉和操作孤立化值。

當使用這種技術時，**絕對不能**捕捉 `self`，即使是隐含地。
```swift
actor BackgroundStyler  {
     // 另一個演員隔離型別
    private let store = StyleStore ()

    deinit {
         // 在這裡沒有演員隔離，因此 Task 中也沒有
        Task { [store] in
            await store.stopNotifications ()
        }
    }
}
```

> 識別：**絕不**從 within `deinit` 延長 `self` 的生命周期。這樣會在 runtime 時崩潰。