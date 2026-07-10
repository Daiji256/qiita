---
title: Xcode 27で@Stateのマクロ化したことで、無駄なインスタンス生成がなくなる
tags:
  - SwiftUI
  - Xcode
  - Swift
  - Observable
  - iOS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

# Xcode 27で@Stateがマクロ化されたことで、無駄なインスタンス生成がなくなる

最近SwiftUIを学んでいたら、`View` が再評価されるたびに、状態を保持しているにもかかわらずインスタンスが無駄に生成されるという現象に違和感を覚えました。調べたところ、この問題は他のエンジニアも感じている既知の問題だったようです。

iOS 17で導入されたObservationフレームワーク（`@Observable` と `@State`）は、従来の `ObservableObject` に代わる新しい状態管理を提供しました。しかし、Xcode 26までの環境では、無駄なインスタンス生成というパフォーマンス上の課題が残されていました。

本記事では、Xcode 27で実施された `@State` のマクロ化によるパフォーマンス改善の内部メカニズムと、既存プロジェクトをXcode 27へ移行する際に必ず直面する「初期化ルールの厳格化」に伴うコンパイルエラーの解消方法について、深く掘り下げて解説します。

## Xcode 26までの課題：プロパティラッパーにおける遅延評価の限界

Xcode 26までの `@State` は単なるプロパティラッパーとして実装されているため、初期値の遅延評価をサポートしていません。これにより、`View` の再生成のたびに利用されることのないインスタンスが生成されます。2回目以降に生成されたインスタンスは不要なため、生成直後に破棄されます。

以下のサンプル実装では、`ContentView` が更新されると `CounterView` が再生成され、`Counter` の新しいインスタンスが生成されます。つまり `init` にある `print("Counter initialized")` も再実行されます。

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
        Button("increment") {
            counter.increment()
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

## Xcode 27の `@State` は遅延評価に対応した

Xcode 27では、この問題を解決するため `@State` がマクロ化され、初期値の遅延評価に対応しました。

マクロはコンパイル時にコードを評価し、データを保持するためのプロパティを生成します。宣言時に与えられた初期値が直接評価されるのではなく、自動的にクロージャの中にラップされます。

このクロージャは、SwiftUIがインスタンス用のメモリを確保する最初のタイミングでのみ評価・実行されます。以降の `View` 再生成時にはクロージャの実行がスキップされるため、インスタンス化やそれにまつわる処理は実行されません。

前述のサンプルコードをXcode 27でビルドすると、`Counter initialized` は初回描画時の1度しか出力されません。

このパフォーマンス改善は、実行環境のOSではなく、コンパイラによって実現されています。つまり、Xcode 27を使用してアプリをビルドすれば、古いOSバージョンでも、この遅延評価の恩恵を受けることができます。

## まとめ

- Xcode 26までは、`View` の再描画時に無駄なインスタンスの生成・破棄が実行されていた
- Xcode 27の `@State` マクロ化により、`View` の再描画に伴う `@Observable` インスタンスの無駄な生成が解消された
- `@State` マクロ化はコンパイラにより実現されるため、古いOSバージョンでも恩恵を受けることができる

## 参考文献

- [TN3211: Resolving SwiftUI source incompatibilities for State and ContentBuilder | Apple Developer Documentation](https://developer.apple.com/documentation/technotes/tn3211-resolving-swiftui-source-incompatibilities-for-state-and-contentbuilder)
- [Migrating from the Observable Object protocol to the Observable macro | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)
