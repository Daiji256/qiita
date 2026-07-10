---
title: Xcode 27で@Stateマクロにより@Observableの無駄なインスタンス生成がなくなった
tags:
  - SwiftUI
  - Xcode
  - Swift
  - Observable
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Xcode 27で@Stateマクロにより@Observableの無駄なインスタンス生成がなくなった

KMPを学んでいると、自然とiOSやSwiftに興味が向くようになりました。AndroidやComposeに慣れているとSwiftUIの挙動に違和感を持つことがあります。

違和感の1つに、`@State` で保持しているにもかかわらず、インスタンスが何度も生成されることです。ただし、2回目以降は生成されたインスタンスは即時に不要となり、破棄されます。

調べていると、Xcode 27でこの問題が解決するということがわかりました。本記事では、執筆時最新のXcode 27 Beta 3を用いて確認したことを紹介します。

## Xcode 26までは再描画のたびにインスタンスが生成・破棄されていた

本節では、Xcode 26での動作を説明します。

従来の `ObservableObject` + `@StateObject` に代わる形で、iOS 17で `@Observable` + `@State` が登場しよりモダンな状態管理が可能になりました。しかし、`@StateObject` が持っていた初期値の遅延評価やインスタンスは1度しか生成しないという特性を、`@State` は持っていません。

そのため、親 `View` の更新による子 `View` の再生成時に、SwiftUIの状態自体は維持されるにもかかわらず、新しいインスタンスが生成されます。つまり、インスタンス作成時の `init` 処理も実行されます。

以下の実装例では、`ContentView` の更新により `CounterView` が再生成されると、`Counter` の `init` が再実行されます。

```swift
struct ContentView: View {
    var body: some View {
        // ...
        CounterView()
    }
}

struct CounterView: View {
    @State private var counter = Counter()

    var body: some View {
        VStack {
            Text("\(counter.count)")
            Button("increment") {
                counter.increment()
            }
        }
    }
}

@Observable
class Counter {
    private(set) var count: Int = 0
    
    init() {
        print("Counter initialized")
    }
    
    func increment() {
        count += 1
    }
}
```

## Xcode 27では `@State` がSwiftマクロになった

Xcode 27では、`@State` がプロパティラッパーからSwiftマクロに置き換わりました。

このマクロ化により、初期値の遅延評価に対応しました。また、それにより無駄にインスタンス生成することがなくなります。

前節のサンプルコードをXcode 27でビルドすると、`Counter initialized` は1度しか実行されません。このことから、インスタンスの生成が1度であることがわかります。


## まとめ

- TODO: 箇条書きで3つ程度

## 参考文献

- [TN3211: Resolving SwiftUI source incompatibilities for State and ContentBuilder | Apple Developer Documentation](https://developer.apple.com/documentation/technotes/tn3211-resolving-swiftui-source-incompatibilities-for-state-and-contentbuilder)
- [Migrating from the Observable Object protocol to the Observable macro | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)
