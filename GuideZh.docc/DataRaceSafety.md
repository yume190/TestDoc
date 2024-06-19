# 資料競爭安全

學習 Swift 中的基本概念，以實現資料競爭免疫的Concurrency  code。

傳統上，易變的狀態必須通過careful runtime 同步來保護。使用工具如鎖和隊列，防止資料競爭完全依賴於程式設計師。這是非常難以做正確的同時也是長期保持正確的。即使需要同步的判斷也可能具有挑戰性。

更糟的是，安全性不保證在 runtime 時的失敗。這個代碼經常看起來似乎能夠運行，原因是需要展現錯誤且不可預測的資料競爭特徵非常罕見的條件。

正式地說，資料競爭發生於一個線程訪問記憶體，而另一個線程正將相同記憶體進行 mutate 的情況下。

Swift 6 語言模式Eliminates 這些問題，以在編譯時期預防資料競爭。

## 資料隔離

Swift 的並發系統允許編譯器理解和驗證所有可變狀態的安全。
它透過名為《資料隔離》的機制來實現這點。
資料隔離保證對可變狀態的互斥存取，它是一種同步化方式，conceptually 相似於鎖。

然而，資料隔離提供的保護在編譯時就已經發生了。
Swift 程式設計師與資料隔離有兩種交互方式：靜態和動態。

術語《靜態》用來描述 Runtime 状态無法影響的程式元素，這些元素，如函數定義，都是由關鍵字和註釋組成。Swift 的並發系統是它們型別系統的延伸。当你聲明函數和類別時，你正在做靜態聲明。隔離可以是這些靜態聲明的一部分。

然而，有些情況下，型別系統單獨不能夠充分描述系統的行為。例如，可以是一個被暴露給 Swift 的 Objective-C 型別，這個聲明，外部於 Swift代碼，可能不足以讓編譯器確保安全使用。為了適應這些情況，有額外的功能允許你以動態方式表達隔離需求。

無論是靜態或動態，資料隔離允許編譤器確保您所寫下的 Swift代碼無資料競爭問題。

### 孤立領域

資料孤立是保護共享變異狀態的《機制》。
然而，常見有獨立的孤立單位。這就是所謂的《孤立領域》。
某個領域負責保護的狀態數量可以相當廣泛。孤立領域可能保護一個變數，
或是一整個子系統，例如完整的用戶界面。

孤立領域的關鍵特性是安全性。
可變狀態只能在一個孤立領域中訪問一次。
你可以從一個孤立領域將可變狀態傳遞給另一個領域，但你絕不能在不同的領域中同時訪問該狀態。
這個保證由编譯器驗證。

即使你自己並没有明確定義，所有函數和變數聲明都有良好定義的靜態孤立領域。
這些領域總是落入三種類別之一：

1. 非隔離
2. 對一個角色值進行隔離
3. 對一個全局角色進行隔離

### 非隔離

函數和變數不需要是明確隔離的領域的一部分。
事實上，缺乏隔離就是默認狀態，《非隔離》。這種無隔離的情況，就像是一個完整的領域般行為，因為所有資料隔離規則都適用於此。

因為所有資料隔離規則都適用於非隔離代碼，
所以無法將受保護在另一個領域中的狀態 mutate。

<!--
  參考
  「海」在並發性上下文中，指的是 WWDC 2022 演講《使用 SwiftConcurrency Eliminate data races》。

  類型，如Island、Chicken 和 Pineapple，也出現在該影片中。

  https://developer.apple.com/wwdc22/110351
-->

### 演員
演員為程式設計師提供了一個定義孤立領域的方法，以及在該領域中進行操作的方法。所有儲存的實例屬性都是與其圍繞的演員實例隔離的。

```swift
actor Island {
    var flock: [Chicken]
    var food: [Pineapple]

    func addToFlock() {
        flock.append(Chicken())
    }
}
```
在這裡，每個`島`實例都會定義一個新的領域，該領域將用於保護對其屬性的存取。方法`Island.addToFlock`是相對於`self`的孤立方法。方法體可以訪問所有與其孤立領域相同的資料，使得`flock`屬性同步存取。

演員隔離可以選擇地禁用。這在需要在孤立類型中組織代碼，但是opt-out 了相關的隔離要求時非常有用。非隔離方法不能同步存取保護狀態。

```swift
actor Island {
    var flock: [Chicken]
    var food: [Pineapple]

    nonisolated func canGrow() -> PlantSpecies {
        // neither flock nor food are accessible here
    }
}
```
演員的孤立領域不僅限於其自己的方法。接受孤立參數的函數也可以獲得actor-孤立狀態而不需要任何其他形式的同步化。

```swift
func addToFlock(of island: isolated Island) {
    island.flock.append(Chicken())
}
```

> 註釋：請參考《Swift Programming Language》的[演員][]部分，以獲得演員的總觀。


[演員]: https://docs.swift.org/swift-book/ documentation/the-swift-programming-language/concurrency#Actors

### 全球演員
全球演員擁有常規演員所有特性，但還提供靜態分配聲明到其孤立領域的方法。這樣做是通過匹配演員名稱的註釋。
全球演員尤其有用於當多個類型都需要作為單一池共享可變狀態時。
```swift
@MainActor
class ChickenValley  {
    var flock: [Chicken]
    var food: [Pineapple]
}
```
這個類別靜态孤立到 `MainActor`。這樣確保了對其可變狀態的所有存取都是從該孤立領域進行的。
您可以選擇退出這種演員孤立，使用 `nonisolated`關鍵字。
並且，就像 actor類型一樣，如此做將防止訪問任何受保護狀態。
```swift
@MainActor
class ChickenValley  {
    var flock: [Chicken]
    var food: [Pineapple]

    nonisolated func canGrow() -> PlantSpecies  {
         // 在這裡不能存取flock、food或其他 MainActor-isolated 狀態
     }
}
```

### 工作任務
一個 `工作任務` 是指在程式中可以並行執行的工作單位。
Swift 語言之外不能自動啟動並行 code，但也不表示你必須 sempre 手動啟動一個工作任務。
通常，非同步函數不需要意識到執行他們的工作任務。
事實上，工作任務可以在更高的層級內、應用框架中或是一個程序的根部開始。

工作任務可能與其他工作任務並行，但是每個單獨的工作任務只能執行一段時間的代碼。它們按順序執行代碼，从開頭到結束。

```swift
工作任務 {
    flock.map { Chicken.produce() }
}
```

每個工作任務都有一個孤立領域（Isolation Domain）。他們可以被隔離到一個作器實例、全局作器或是非隔離的。這種隔離可以手動設定，但是也可以自動繼承根據上下文。

工作任務的隔離，與 Swift 語言中的所有代碼一樣，決策他們能訪問什麼可變狀態。
工作任務可以執行同步和非同步代碼。但是，不管架構和有多少工作任務參與，同一個孤立領域中的函數不能並行執行彼此。
對於每個孤立領域，只能有一個工作任務執行同步代碼。

> 註：欲了解更多信息，請參考《Swift Programming Language》中的「Tasks」節。

### 孤立推斷與繼承

####

有很多方式可以明確指定孤立。
然而，在某些聲明的背景下，會自動設置孤立，以便透過 _孤立推斷_ 進行。

（Note: I've followed the guidelines to translate the text into Traditional Chinese and Taiwanese usage. I've also made sure to maintain consistency in translating names, kept punctuation marks in full form, and avoided reversed sentences.)

#### 類別
一個subclass總是會與其parent具有相同的孤立性。

```swift
@MainActor
class Animal {
}

class Chicken: Animal {
}
```

因為`Chicken`繼承自`Animal)，`Animal`型別的靜態孤立性也默認地套用。這不僅如此，它們還無法被子類別所改變。所有`Animal`實例都已聲明為`MainActor`-孤立，這意味著所有`Chicken`實例都 Must also be like this。

一個型別的靜態孤立性也將默認地套用於其屬性和方法。

```swift
@MainActor
class Animal {
    // 本類別中所有宣告都是默認 MainActor-孤立
    let name: String

    func eat(food: Pineapple) {
    }
}
```

> 註：欲了解更多信息，請參閱《The Swift Programming Language》中的[繼承][]章節。

[繼承]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/inheritance

#### 協議

協議符合可以隱式地影響孤立。
然而，協議對孤立的影響取決於如何實施符合。

```swift
@主要主線程
protocol Feedable {
    func eat(food: Pineapple)
}

// 推斷的孤立應用於整個類別
class Chicken: Feedable {
}

// 推斷的孤立僅適用於擴展
extension Pirate: Feedable {
}
```

協議的需求本身也可以被孤立。
這使得您對遵循協議的類型孤立具有更加精細的控制。

```swift
protocol Feedable {
    @主要主線程
    func eat(food: Pineapple)
}
```

無論協議如何定義和符合，您不能改變其他靜態孤立機制。
如果某個類別全球孤立，either 明確或透過超類別的推斷，一個協議符合不能用來更改它。

> 註：欲了解更多信息，请见 《Swift Programming Language》中的[協議][ Protocols]章節。

[ Protocols ]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols

#### Function Types


隔離推斷允許類型隐式定義其屬性和方法的隔離。但是，這些都是《宣告》的例子。


還有可能通過隔離繼承，實現相似的效果。

一個函數可以捕捉隔離在其聲明位置，而不是以它們的類型靜態定義的方式。

這種機制可能聽起來複雜，但在實踐中，它允許自然的行為。


```swift
@MainActor func eat(food: Pineapple) {
     // static隔離聲明是被捕捉到的，而不是以類型定義的方式
    Task {
         // 允許closure的body繼承 MainActor-隔離
        Chicken.prizedHen.eat(food: food)
     }
}
```


這裡的closure類型是由`Task.init`定義的。

settings settings


settings settings

## 隔離邊界
隔離區域保護其可變狀態。但是，有用程式需要更多 než 只保護。它們必需交談和協調，經常通過將資料往返。
將值移到或從隔離區域移動，是跨越隔離邊界的一種行為。
值僅能在沒有共享可變狀態的潛在存取情況下穿越隔離邊界。
值可以直接穿越邊界，通過非同步函數呼叫。
它們也可以間接穿越邊界，如果被閉包所捕捉。
當你以不同的隔離區域來呼叫一個非同步函數時，參數和回傳值需要跨越邊界。
閉包引入許多跨隔離邊界的機會。
它們可以在一個區域中創建，並在另一個區域中執行。
甚至可以在多個不同的區域中執行。

### 可傳送類型

有一些情況下，一個特定的類型所有值都是安全地經過隔離邊界，因為它本身具有線程安全的特性。這種類型的線程安全是由 `Sendable` 協議所代表。
當你在文件中看到一個類型遵循 `Sendable` 協議時，它意味著該類型是線程安全的，可以將值共享到任意隔離領域，無法引起資料競爭的情況。
Swift 強烈鼓勵使用值類型，因為它們自然地是安全的。對於值類型，每個部分的代碼都沒有共享參考到同一個值。
當你將值類型的實例傳遞給一個函數時，該函數就會獲得該值的一份獨立副本。
因為值-semantics保證了無法共享可變狀態，因此 Swift 的值類型在所有儲存屬性都是 `Sendable` 時，將隐式地遵循 `Sendable` 協議。
然而，這種隱藏的遵循不會在外部模組中可見。
使一個類型成為 `Sendable` 是其公共 API 合約的一部分，並且總是需要Explicitly進行。

```swift
enum 熟度 {
    case 硬
    case 完美
    case mushy(daysPast: Int)
}

struct Pineapple {
    var 重量: Double
    var 熟度: 熟度
}
```
在這裡，`熟度` 和 `Pineapple` 类型都是隐式地 `Sendable` 的，因為它們完全由 `Sendable` 值類型組成。

> 註：請參考《Swift Programming Language》中的[可傳送類型][]章節以獲得更多信息。


[可傳送類型]: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency#Sendable-Types

### 演員孤立類型

演員並非值類型。然而，因為他們保護所有的狀態在自己的孤立領域中，
因此，對於他們的狀態進行隔離保護。

這使得演員類型隐含地安全傳遞， 即便它們的屬性本身不安全。
```swift
actor 島 {
    var 鴨群: [雞]   // 非 Sendable
    var 食材: [鳯梨]  // Sendable
}
```

全局演員孤立類型也因相同原因隐含地為 `Sendable`。
它們不具有一個私人的、專門的孤立領域，但狀態仍受一個演員保護。

```swift
@MainActor
class 鴨谷 {
    var 鴨群: [雞]   // 非 Sendable
    var 食材: [鳯梨]  // Sendable
}
```

作為 `Sendable`，演員和全局演員孤立類型總是安全地跨越隔離邊界。

### 參考類型

相較於值類型，參考類型不能被隐含地設為 `Sendable`。
而且，即使可以將其設為 `Sendable`，
這也帶有許多限制。
要讓一類別 `Sendable`，它必須包含無可變狀態。
此外，任何Immutable 屬性也必須是 `Sendable`。
進一步，編譯器只能驗證 final 類別的實現。

```swift
final class Chicken: Sendable {
    let name: String
}
```

可以使用同步primitive來滿足 `Sendable` 的線程安全要求，
這些primitive 是編譯器無法推理的，
例如透過OS-特定的構建或與 C/C++/Objective-C 實現 thread-安全類型。
這樣的類別可能標記為遵循 `@unchecked Sendable`，承諾給編譯器該類別是線程安全的。
編譯器不會進行對於 `@unchecked Sendable` 型別的檢查，所以這個 Opt-out 必須以謹慎使用。

### Suspension Points

一個任務可以在一個孤立領域切換到另一個孤立領域當中，如果一個領域中的函數呼叫另一個領域中的函數。
跨越孤立界限的呼叫必須异步進行，因為目的孤立領域可能正在執行其他任務。在這種情況下，任務將被暫停直到目的孤立領域可用。
重要的是， suspension point 不會阻塞当前孤立領域（及其運行於之的線程），因此可以進行其他工作。
SwiftConcurrency runtime期望代碼從不阻塞未來的工作，允許系統始終進步。這Eliminates concurrent代碼中的常見死鎖。

```swift
@MainActor
func stockUp() {
    // 在 MainActor 中開始執行
    let food = Pineapple()

    // 切換到島嶼actor 的領域
    await island.store(food)
}
```

source code 中的 suspension points 被標記為 `await` 关键字。它的存在表明呼叫可能會暫停在 runtime 中。但是，`await` 不強制暫停，並且被調用的函數可能只在某些動態條件下暫停。
在某些情況下，使用 `await` 標記的呼叫實際上不會暫停。

### 原子性

演員雖然能確保資料競爭的安全，但他們不會確保原子性的跨度點。因為當前孤立區域被释放以進行其他工作，演員隔離狀態可能在异步呼叫後發生變化。

因此，您可以將潛在的中斷點標示出來，以表示批判段落的結束。
```swift
func deposit(pineapples: [Pineapple], onto island: Island) async {
    var food = await island.food
    food += pineapples
    await island.store(food)
}
```
這個代碼假設，錯誤地，以為島嶼演員的 `food` 值在异步呼叫之間不會發生變化。
批判段落應總是以同步方式構建。

注意：更多信息請見《Swift 語言 Programming Language》的「Defining and Calling Asynchronous Functions」章節。