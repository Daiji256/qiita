---
title: Kotlin 2.4.0のSwift ExportでFlowをSwiftUIのStateにバインドする
tags:
  - KMP
  - Kotlin
  - Swift
  - SwiftUI
  - SwiftExport
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

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

SwiftUIの `View` は、画面の再描画のたびに `init` が呼び出されるという特性があります。そのため、`View` の中で直接Kotlinのクラスをインスタンス化すると、不要な再生成が繰り返されてしまいます[^xcode-27]。

[^xcode-27]: `@Observable` と `@State` を使わずに、旧来の `ObservableObject` と `@StateObject` を使う理由はここにあります。ただし、Xcode 27からは `@State` 周辺の仕組みが変わるため、今後は実装方法が変わると思います。

これを防ぐ、Composeの `remember` のように「Viewのライフサイクルに合わせて一度だけ初期化・保持」させるようなものを実装します。具体的には、SwiftUIの `@StateObject` で扱えるようシンプルなラッパークラスを作成します。

```swift
final class Remember<T>: ObservableObject {
    let value: T

    init(_ initialValue: @autoclosure () -> T) {
        self.value = initialValue()
    }
}
```

## `Flow` の観測・値を用いた描画のための `View` を実装

SwiftUIとComposeではインスタンスの作成タイミングやライフサイクルの管理方法が異なります。SwiftUIではComposeと同じアプローチをとることはできません。

そこで、SwiftUIの `@State` で状態を保持し、`init` と `task` にて初期化と更新を行う専用の `View` を作成します。

SwiftではKotlinの `Flow` を直接 `collect` することはできませんが、Swift Exportによる `Flow.asAsyncSequence()` により `AsyncSequence` に変換し、`for try await` することで `collect` と同等の動作を実現できます。

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
        self._state = State(wrappedValue: initialValue)
        self.flow = flow
        self.content = content
    }

    init(
        stateFlow: any KotlinTypedStateFlow<T>,
        @ViewBuilder content: @escaping (T) -> Content
    ) {
        self._state = State(wrappedValue: stateFlow.value)
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

作成した `Remember` と `ObservingView` を使って、実際にSwiftUIの `View` を構築します。

`View` では直接的には `@StateObject` を用いて `Store` を保持することしかしません。状態の監視等は `ObservingView` に閉じて行います。`ObservingView` の `content` では `state` を受け取ることができるため、`Flow` の変更をUIに反映することができます。

```swift
struct ContentView: View {
    @StateObject private var storeHolder = Remember(Store())

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
- SwiftUIのライフサイクルの違い（`init` の反復呼び出しなど）を吸収するため、`ObservableObject` / `@StateObject` を活用してKotlin側のインスタンスを保持する
- `Flow` の購読処理を `ObservingView` のように専用の `View` へ切り出すことで、`Flow` を手軽に描画に利用できる。

## 参考文献

- [Interoperability with Swift using Swift export | Kotlin Documentation](https://kotlinlang.org/docs/native-swift-export.html)
- [Flows | Kotlin Documentation](https://kotlinlang.org/docs/coroutines-flow.html)
- [AsyncSequence | Apple Developer Documentation](https://developer.apple.com/documentation/swift/asyncsequence)
