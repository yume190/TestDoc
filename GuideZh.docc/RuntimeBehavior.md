# Concurrency Runtime 行為

了解Swift concurrency runtime semantics與其他執行環境的差異，並學習實現類似的執行 semantics 的通用模式。

Swift Concurrency模型強調async/await、actors 和 tasks，這意味著來自其他library 或 concurrency runtimes 的某些模式不直接轉化為這個新模型。在這裡，讓我們探討common patterns和runtime behavior的差異，並了解如何處理這些問題，而不是在您的代碼從SwiftConcurrency中遷移到。

## 使用 Task Groups 限制Concurrency

有時候，你可能會遇到需要處理大量工作的情況。

即使可以將所有這些工作項目 enqueue 到一個 task group 中，但是一開始enqueue thousands of tasks是否適合？創建一個task（在 `addTask`中）需要分配一些記憶體以便 suspend 和 execute 這個task。這個amount of memory 不是太大，如果創建 thousands of tasks 只是等待 executor 執行，而不是 immediately execure 這可能會導致significant memory usage。

當你面對這種情況時，手動 throttle  Concurrently added tasks 到 task group 中可能是有益處的。

```swift
let lotsOfWork: [Work] = ... 
let maxConcurrentWorkTasks = min(lotsOfWork.count, 10)
assert(maxConcurrentWorkTasks > 0)

await withTaskGroup(of: Something.self) { group in
    var submittedWork = 0
    for _ in 0..<maxConcurrentWorkTasks {
        group.addTask { // or 'addTaskUnlessCancelled'
            await lotsOfWork[submittedWork].work() 
        }
        submittedWork += 1

    }

    for await result in group {
        process(result) // process the result somehow, depends on your needs

        // Every time we get a result back, check if there's more work we should submit and do so
        if submittedWork < lotsOfWork.count, let remainingWorkItem = lotsOfWork[submittedWork] {
            group.addTask { // or 'addTaskUnlessCancelled'
                await remainingWorkItem.work() 
            }
            submittedWork += 1
        }
    }
}
```