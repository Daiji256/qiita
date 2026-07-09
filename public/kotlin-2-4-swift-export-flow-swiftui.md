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

## 色々

- Kotlin 2.4.0のSwift ExportでFlowに対応
- Composableでやっている `collectAsState` をSwiftUIでもやりたい
- SwiftUIでは全く同じようにはできないので、`@State` に値を格納するための拡張関数を作る
- Composableのようなライフサイクルやインスタンスの作成にはならないから、`init` と `task` で実現する
- `flow.asAsyncSequence()` で `Sequence` になるので、Swiftで扱いやすくなる

## 記事に載せる予定のソースコード

### このようなよくあるコードをSwiftUI側からも扱いたい

```kotlin
class Store {
    private val _state = MutableStateFlow(0)
    val state = _state.asStateFlow()

    fun increment() {
        _state.update { it + 1 }
    }
}
```

### Composableでいうこんな感じ

```kotlin
@Composable
fun ContentComposable() {
    val store = remember { Store() }
    val state by store.state.collectAsState()
    Column {
        Text(text = state.toString())
        Button(onClick = { store.increment() }) {
            Text("increment")
        }
    }
}
```

### Stateに格納する拡張関数を作った

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

### SwiftUIではこんなふうに利用できる

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
