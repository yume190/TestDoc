# 將 Swift 遷移到 6

@Metadata {
   @TechnologyRoot
}

@Options(scope: global) {
   @AutomaticSeeAlso(disabled)
   @AutomaticTitleHeading(disabled)
   @AutomaticArticleSubheading(disabled)
}

## 簡介

Swift 的並發系統，於 [Swift 5.5](https://www.swift.org/blog/swift-5.5-released/) 中引入，
使得非同步和平行的代碼更易寫作和理解。
在 Swift 6 語言模式下，編譯器現在能夠保證並發程序免於數據競爭的問題。
當啟用時，編譯器安全檢查從前optional變為required。

將 Swift 遷移到 6 語言模式是您控制的，無論是在每個目標上抑或是在其他語言中曝露給 Swift。

可能您已經逐步 adoptConcurrency特性，因為他們被引入。
或者，您也許一直等待 Swift 6 的發布以後使用這些功能。
不管您的專案在什麼階段，該指南將提供概念和實際幫助，以簡化遷移。

以下您將找到文章和代碼範例，將：

* 解釋Swift 的數據競爭安全模型所需的概念。
* 顯示如何啟用 Swift 6 語言模式。
* 顯示如何啟用完整 concurrency檢查 для Swift 5 專案。
* 提供 Incremental Adoption 技術。
* 提供解決常見問題的策略。

> 重要提示：Swift 6 語言模式是_選擇性的_。
現有的專案將不會自動遷移到這個模式，需要進行設定變更。

>
> 這有個重要的區別之間compiler 版本和語言mode。
Swift 6 的 compiler 支持四種不同的語言mode：「6」、「5」、「4.2」和「4」。

> 註：該指南仍在活躍開發中。您可以查看原始碼、查看完整的代碼範例，並學習如何參與 GitHub 存儲庫中。[repository] 。

[repository]: https://github.com/apple/swift-migration-guide