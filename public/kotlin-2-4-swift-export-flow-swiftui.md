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

以下のような状態を管理する `Store` クラスを定義します。

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

## SwiftUI向けの拡張関数を作成する

SwiftUIとComposeではインスタンスの作成タイミングやライフサイクルの管理方法が異なります。SwiftUIではComposeと同じアプローチをとることはできません。

そこで、SwiftUIの `@State` で状態を保持し、`init` と `task` にて初期化と更新を行います。再利用性を考え、`Flow` を `collect` した時に `@State` に値を格納する `View` の拡張関数として `bind` を作成します。

SwiftではKotlinの `Flow` を直接 `collect` することはできません。Swift Exportによる `Flow.asAsyncSequence()` により、`AsyncSequence` に変換し、`for try await` することで `collect` と同等の動作を実現できます。

以下の実装例では、`Flow` のインスタンスが変わったときに `Task` を再起動するようにしています。

```swift
extension View {
    func bind<T>(
        _ flow: any KotlinTypedFlow<T>,
        to binding: Binding<T>
    ) -> some View {
        let flowId = ObjectIdentifier(flow as AnyObject)
        return self.task(id: flowId) {
            do {
                for try await value in flow.asAsyncSequence() {
                    binding.wrappedValue = value
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

作成した `bind` を使って、実際にSwiftUIの `View` を構築します。

Composeの `remember` の代わりに `init` で初期状態を設定し、作成した `bind` でKotlinの `Flow` とSwiftUIの `@State` を紐付けます。

```swift
struct ContentView: View {
    @State private var store: Store
    @State private var state: Int32

    init() {
        let store = Store()
        self.store = store
        self.state = store.state.value
    }

    var body: some View {
        VStack {
            Text("\(state)")
            Button("increment") {
                store.increment()
            }
        }
        .bind(store.state, to: $state)
    }
}
```

## 結果

前節までで紹介したサンプルコードにより、`Flow` をUI上で表示できるようになります。

![SwiftUI上でFlowの表示ができている様子](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/5f398e72-0cfb-4e7f-92d6-b8fc191f9472.gif)

## まとめ

- Kotlin 2.4.0のSwift Exportと `Flow.asAsyncSequence()` を活用することで、Swiftからも `Flow` を扱うことができる
- `init` と `task` で初期化・更新を行うことで、SwiftUI上で `Flow` の値を描画に利用できる

## 参考文献

- [Interoperability with Swift using Swift export | Kotlin Documentation](https://kotlinlang.org/docs/native-swift-export.html)
- [Flows | Kotlin Documentation](https://kotlinlang.org/docs/coroutines-flow.html)
- [AsyncSequence | Apple Developer Documentation](https://developer.apple.com/documentation/swift/asyncsequence)
