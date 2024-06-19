# Incremental 採用

了解如何將 Swift 同步功能逐步引入您的專案中。
遷移到 Swift 6 語言模式通常是階段性的進行。事實上，許多專案在 Swift 6 還未出現時就已經開始了過程。你可以繼續引入同步特性,_逐步_ 地解決可能出現的問題。
這樣便能夠順暢進步，而不會干擾整個專案。
Swift 包含許多語言功能和標準庫 API，以幫助您逐步採用。

Note: I've kept the formatting and capitalization of the original text, but translated it into Traditional Chinese as requested.

## 使用回呼函數包裝 callback 函數

Swift 中接受並呼叫單一完成函數的 API 是非常常見的模式。

可以將這種函數轉換為直接從異步上下文中使用。

```swift
func updateStyle(backgroundColor: ColorComponents, completionHandler: @escaping () -> Void) {
    // ...
}
```

這是一個使用回呼函數通知客戶其工作完成的範例。沒有方法讓叫用者知道回呼將在什麼時間或執行緒上被invoke，除非查看文檔。

你可以將這個函數包裝為一個异步版本使用 `_continuations`。

```swift
func updateStyle(backgroundColor: ColorComponents) async {
    withCheckedContinuation { continuation in
        updateStyle(backgroundColor: backgroundColor) {
            continuation.resume()
        }
    }
}
```

在异步版本中，沒有更多模糊性。在函數完成後，執行總是會恢復到最初啟動它的上下文。

```swift
await updateStyle(backgroundColor: color)
// style has been updated
```

`withCheckedContinuation` 函數是一個標準庫 API，存在於以便使非 async 和 async代碼之間進行交互。

注意：引入异步代碼到專案中可能會 surface 數據隔離檢查違反。要了解和處理這些問題，请查看 [跨隔離邊界][]

[crossing-isolation-boundaries]: commonproblems#Crossing-Isolation-Boundaries
[continuation-apis]: https://developer.apple.com/documentation/swift/concurrency/continuations

## 動態孤立化


使用注釋和其他語言構造表達程式的靜態孤立化，實現了強大的同時性。然而，這也可能導致更新所有依賴项同時需要努力。

動態孤立化提供了runtime機制，您可以將其作為 fallback 使用，以描述數據孤立化。
這是一種在與其他未更新的Swift 6元件進行交互時，非常重要的工具，即便這些元件位於相同模組中。

### 內部隔離

假設您已經確定了專案中的一個參考型別最好用`MainActor`靜態隔離來描述。
```swift
@MainActor
class WindowStyler  {
    private var backgroundColor: ColorComponents

    func applyStyle()  {
        // ...
    }
}
```
這個`MainActor`隔離可能是邏輯正確的。但如果這個型別在其他未遷移的位置中被使用，添加靜態隔離將需要進行許多額外的變更。另一個選擇是使用動態隔離以控制作用域。
```swift
class WindowStyler  {
    @MainActor
    private var backgroundColor: ColorComponents

    func applyStyle()  {
        MainActor.assumeIsolated  {
            // 使用和互動其他`MainActor`狀態
        }
    }
}
```
在這裡，隔離已經內部化到類別中。這樣就能夠將變更局限於型別，不影響任何類別的客戶。
然而，這種技術的一大缺點是型別的真實隔離需求仍然不明顯。在公用 API 中，您無法了解是否或如何應該根據此進行變更。因此，僅當您已經考慮過其他選項時才應該使用這種方法。

### 使用僅隔離


如果不能將隔離僅限於某種類型，你可以將隔離擴展到僅在該類型的 API 使用中。

要進行這個步驟，先對類型apply靜態隔離，然后在使用位置使用動態隔離：
```
@主要演員
class WindowStyler  {
     // ...
}

class UIStyler  {
     @主要演員
    private let windowStyler: WindowStyler
    
    func applyStyle()  {
        MainActor.assumeIsolated  {
            windowStyler.applyStyle()
        }
    }
}
```
將靜態和動態隔離結合可以是一個強大的工具，保持變化的範圍漸進。

## 缺漏註釋

動態孤立可以在執行時提供表達孤立的工具。
但是，您可能也需要描述從未遷移模組中缺乏的其他並發性屬性。

### 非標記化可发送閉包

一個閉包的sendability直影響編譯器對其身體的孤立性假設。
有一個callback閉包它實際上跨越了孤立界限，但卻缺乏`Sendable`註釋，違反了並發系統的一個關鍵不變式。

```swift
// 在 pre-Swift 6 模組中定義
extension JPKJetPack {
    // 注意沒有 `@Sendable` 註釋
    static func jetPackConfiguration(_ callback: @escaping () -> Void) {
        // 可能跨越孤立界限
    }
}

@MainActor
class PersonalTransportation {
    func configure() {
        JPKJetPack.jetPackConfiguration { 
            // 在 MainActor 孤立中 inferred 這裡
            self.applyConfiguration()
        }
    }

    func applyConfiguration() {
    }
}
```

如果 `jetPackConfiguration` 可以在另一個孤立界限中呼叫其閉包，那麼它就必須標記為 `@Sendable`。
當一個未遷移的模組還沒有進行這樣的修改時，它將導致錯誤的.actor 瞩觀。
這個代碼將在編譯時無問題，但是在 runtime 時會 Crash。

為了 workaround 這個問題，你可以手動將閉包註釋為 `@Sendable`。這樣就會阻止編譯器對 MainActor 孤立的假設。
因為編譯器現在知道 Actor 孤立可能會改變，所以它需要在_callsite_ 中使用 await。

```swift
@MainActor
class PersonalTransportation {
    func configure() {
        JPKJetPack.jetPackConfiguration { @Sendable in 
            // Sendable 閉包不會對 actor 孤立進行假設，將這個上下文標記為非孤立
            await self.applyConfiguration()
        }
    }

    func applyConfiguration() {
    }
}
```

## 後向相容性


static 隔離，因為它是型別系統的一部分，會影響你的 public API。
但是，你可以將自己的模組遷移到 Swift 6，以改善其 APIs，而不破壞任何現有的客戶端。
假設 `WindowStyler` 是 public API。
你已經确定了它真的應該被 `MainActor` 隔離，但想確保後向相容性以便客戶。

```swift
@preconcurrency @MainActor
public class WindowStyler {
    // ...
}
```
使用 `@preconcurrency` 這樣標記隔離，標誌著隔離條件是_CLIENT_MODULE_ 也已經啟用完整檢查。這樣保留了源相容性，以便客戶還沒有開始採用 Swift 6。

## Dependency relaciones

有時候，你無法控制需要 import 的模組。
如果這些模組仍未遵循 Swift 6，則可能發生難以解決或無法解決的錯誤。
使用未遷移到的代碼會導致多種不同的問題。
`@preconcurrency` 標籤可以幫助解決許多這樣的情況：

- 非可傳送的類型[]
- 協議承認隔離[] mismatch

[非可傳送的類型]: commonproblems#跨界限限制
[協議承認隔離]: commonproblems#跨界限限制

## C/Objective-C


您可以透過注釋，對 C 和 Objective-C API 透露 Swift 并發支持。這是由 Clang 的 [並發特定的注釋][clang-annotations] 所實現：

[clang-annotations]: https://clang.llvm.org/docs/AttributeReference.html#customizing-swift-import

``` 
__attribute__((swift_attr(“@Sendable”)))
__attribute__((swift_attr(“@_nonSendable”)))
__attribute__((swift_attr("nonisolated")))
__attribute__((swift_attr("@UIActor")))

__attribute__((swift_async(none)))
__attribute__((swift_async(not_swift_private, COMPLETION_BLOCK_INDEX)))
__attribute__((swift_async(swift_private, COMPLETION_BLOCK_INDEX)))
__attribute__((__swift_async_name__(NAME)))
__attribute__((swift_async_error(none)))
__attribute__((__swift_attr__("@_unavailableFromAsync(message: \"msg\")")))
```


當您在可以匯入 Foundation 的項目中工作時，以下注釋 macro 皆可在 `NSObjCRuntime.h` 中找到：

``` 
NS_SWIFT_SENDABLE
NS_SWIFT_NONSENDABLE
NS_SWIFT_NONISOLATED
NS_SWIFT_UI_ACTOR

NS_DISABLE_ASYNC
NS_ASYNC(COMPLETION_BLOCK_INDEX)
NS_REFINED_FOR_SWIFT_ASYNC(COMPLETION_BLOCK_INDEX)
NS_SWIFT_ASYNC_NAME
NS_SWIFT_ASYNC_NOTHROW
NS_UNAVAILABLE_FROM_ASYNC(msg)
```

### Objective-C 庫中的缺少孤立註釋


Objective-C 庫和其他 Objective-C 庫會逐漸採用 SwiftConcurrency，並將文檔中解釋的合約轉換為編譯器和runtime 強制執行的孤立檢查。這些檢查可以在您使用這些API 時被 Swift 遮護。

例如，前Swift Concurrency 時候，API 通常需要在註釋中解釋它們的執行緒行為，像這樣：

"This will always be called on the main thread。"

SwiftConcurrency 讓我們將這些註釋轉換為编译器和runtime 強制執行的孤立檢查。

例如，虛構的 NSJetPack 協議通常在主要執行緒上呼叫所有委派方法，因此現在已經變得是 MainActor 孤立的。

庫作者可以使用 `NS_ SWIFT_UI_ACTOR` 屬性標記為 MainActor 孤立，這等於在 Swift 中使用 `@MainActor` 選項：

```swift
@protocol NSJetPack  // 虛構協議
   // ...
@end
```

由於這樣，協議中的所有成員方法都會繼承 MainActor 孤立，並且大多數方法都是正確的。

然而，在這個例子中，讓我們考慮一個曾經以以下方式文檔記錄的方法：

```objc
@protocol NSJetPack  // 虛構協議
/* Return YES if this jetpack supports flying at really high altitude!

 JetPackKit invokes this method at a variety of times, and not always on the main thread. For example,... */

@property(readonly) BOOL supportsHighAltitude;

@end
```

這個方法的孤立性被誤解為 MainActor，因為包含該型別的註釋。但是，它實際上具有不同的執行緒策略 - 它可能不會在主要執行緒上呼叫 - 但它缺乏在方法上標記這些 semantics 的註釋。

正確的長期解決方案是庫作者修正該方法的註釋，將其標記為 `nonisolated`：

```objc
// 解決方案在提供API 的庫中：
@property(readonly) BOOL supportsHighAltitude NS_ SWIFT_NONISOLATED;
```

直到庫作者修正了它們的注釋問題，您可以使用正確的 `nonisolated` 方法，例如這樣：

```swift
// 解決方案在採用客戶代碼中：
@MainActor
final class MyJetPack: NSJetPack  {
   // 正確
  override nonisolated class var readyForTakeoff: Bool  {
    true
   }
}
```

這樣，Swift 就知道不應該檢查方法是否需要主要執行緒孤立。

## 分派

有些您可能從分派或其他Concurrency library 中習慣的模式，在Swift的結構化Concurrency模型中需要重新塑造。

### 使用 Task Group 限制並發的Concurrency


有時候，你可能會遇到需要處理大量工作的 situation。

雖然可以將所有工作項目 enqueue 到一個 task group 裡面，但是這樣可能會導致創建數千個task，直到系統核心數目和執行器設定允許執行，這樣可能不是很明智的做法。

如果你懷疑有百或數千個items，那麼在創建tasks 時可以 manual throttle 來限制並發的Concurrency，例如：

```swift
let lotsOfWork: [Work] = ... 
let maxConcurrentWorkTasks = min(lotsOfWork.count, 10)
assert(maxConcurrentWorkTasks > 0)

await withTaskGroup(of: Something.self) { group in
    var submittedWork = 0
    for _ in 0..<maxConcurrentWorkTasks {
        group.addTask { // 或 'addTaskUnlessCancelled'
            await lotsOfWork[submittedWork].work() 
        }
        submittedWork += 1
    }

    for await result in group {
        process(result) // 處理結果，根據需要

        //每次取得結果後，檢查是否還有更多的工作要提交，並進行提交
        if submittedWork < lotsOfWork.count, 
           let remainingWorkItem = lotsOfWork[submittedWork] {
            group.addTask { // 或 'addTaskUnlessCancelled'
                await remainingWorkItem.work() 
            }
            submittedWork += 1
        }
    }
}
```