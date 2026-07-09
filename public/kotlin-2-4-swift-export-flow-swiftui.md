---
title: newArticle001
tags:
  - KMP
  - Kotlin
  - Swift
  - SwiftUI
  - Swift Export
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Kotlin 2.4.0のSwift ExportでFlowをSwiftUIのStateにバインドする

## はじめに

Kotlin 2.4.0のSwift Exportにより、Kotlinの `Flow` がSwift側から扱えるようになりました。`Flow.asAsyncSequence()` により、Swiftの `AsyncSequence` として処理できます。

この記事では、SwiftUIから気軽に `Flow` を参照し、描画に使える状態として扱えるようにする方法を紹介します。

## サンプルに利用するコード

本記事では以下のような状態を管理する `Store` クラスをKotlinで定義し、Swift ExportによりSwift側から参照することを前提に説明します。

```kotlin
class Store {
    private val _state = MutableStateFlow(0)
    val state = _state.asStateFlow()

    fun increment() {
        _state.update { it + 1 }
    }
}
```

## Composeではどのように扱っているか

SwiftUIでの扱いを考える前に、Composeではどうしていたかをおさらいします。Composeでは `collectAsState()` により、`Flow` / `StateFlow` を `State` に変換しています。

```kotlin
@Composable
fun ContentComposable() {
    val store = remember { Store() }
    val state by store.state.collectAsState()
    // ...
}
```

## Kotlinの実装を `ObservableObject` でラップする

TODO: 簡単に説明、remember に相当

```swift
final class ObservableWrapper<T>: ObservableObject {
    @Published var value: T

    init(_ value: T) {
        self.value = value
    }
}
```

## `Flow` の観測・値を用いた描画のための `View` を実装

SwiftUIとComposeではインスタンスの作成タイミングやライフサイクルの管理方法が異なります。SwiftUIではComposeと同じアプローチをとることはできません。

そこで、SwiftUIの `@State` で状態を保持し、`init` と `task` にて初期化と更新を行います。

SwiftではKotlinの `Flow` を直接 `collect` することはできません。Swift Exportによる `Flow.asAsyncSequence()` により、`AsyncSequence` に変換し、`for try await` することで `collect` と同等の動作を実現できます。

```swift
struct ObservingView<T, Content: View>: View {
    @State private var state: T
    private let flow: any KotlinTypedFlow<T>
    private let content: (T) -> Content

    init(
        flow: any KotlinTypedFlow<T>,
        initialValue: T,
        @ViewBuilder content: @escaping (T) -> Content
    ) {
        self.state = initialValue
        self.flow = flow
        self.content = content
    }

    init(
        stateFlow: any KotlinTypedStateFlow<T>,
        @ViewBuilder content: @escaping (T) -> Content
    ) {
        self.state = stateFlow.value
        self.flow = stateFlow
        self.content = content
    }

    var body: some View {
        content(state)
            .task {
                do {
                    for try await value in flow.asAsyncSequence() {
                        state = value
                    }
                } catch is CancellationError {
                    // cancelled
                } catch {
                    assertionFailure("Flow<\(T.self)> collection failed: \(error)")
                }
            }
    }
}
```

## SwiftUIでの利用例

作成した `ObservingView` を使って、実際にSwiftUIの `View` を構築します。

TODO: 補足

```swift
struct ContentView: View {
    @StateObject private var storeHolder = ObservableWrapper(Store())

    var body: some View {
        let store = storeHolder.value
        ObservingView(stateFlow: store.state) { state in
            VStack {
                Text("\(state)")
                Button("increment") {
                    store.increment()
                }
            }
        }
    }
}
```

## 結果

前節までで紹介したサンプルコードにより、`Flow` をUI上で表示できるようになります。

![SwiftUI上でFlowの表示ができている様子](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/5f398e72-0cfb-4e7f-92d6-b8fc191f9472.gif)

## まとめ

- Kotlin 2.4.0のSwift Exportと `Flow.asAsyncSequence()` を活用することで、Swiftからも `Flow` を扱うことができる
- TODO

## 参考文献

- [Interoperability with Swift using Swift export | Kotlin Documentation](https://kotlinlang.org/docs/native-swift-export.html)
- [Flows | Kotlin Documentation](https://kotlinlang.org/docs/coroutines-flow.html)
- [AsyncSequence | Apple Developer Documentation](https://developer.apple.com/documentation/swift/asyncsequence)
