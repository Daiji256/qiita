---
title: Swift/Objective-CからKotlinへインスタンスを渡すとdeinitが遅れる理由と対策
tags:
  - iOS
  - Kotlin
  - Swift
  - KotlinNative
  - KotlinMultiplatform
private: false
updated_at: '2026-03-19T12:24:45+09:00'
id: f977ab6bdc2f10e36fda
organization_url_name: null
slide: false
ignorePublish: false
---

本記事は、[Kotlin Advent Calendar 2025](https://qiita.com/advent-calendar/2025/kotlin)の17日目の記事です。

## はじめに

Kotlin Multiplatform（KMP）でiOSアプリを開発する際、SwiftとKotlinの相互運用は避けて通れません。KMPは非常に便利ですが、トラブルなく使いこなすにはSwiftとKotlin/Native両方のメモリ管理の特性をある程度理解している必要があります。

ここでは、両者のメモリ管理の違いを簡単に紹介した後、この違いを理解していないと、Swift/iOS側のライフサイクル管理において直感に反するように見える挙動について解説します。

具体的には「SwiftからKotlinの関数等にインスタンスやその参照を渡すと、Swift側の解放が遅延する」という事象です。

なお、本記事ではSwiftを例に説明しますが、Objective-CからKotlinの実装を呼び出す場合でも同様の挙動となります。

## インスタンスの解放が遅れる（`deinit` が遅れる）

例えばSwiftUIでは、画面を閉じたタイミングで、その画面に関連するインスタンスが解放されると期待されがちです。Swiftエンジニアにとっては、そのような挙動が自然に感じられる場面も多いでしょう。

しかし、Kotlinへの関数呼び出しなどでインスタンスを渡している場合、画面を閉じてもすぐにはインスタンスが解放されない場合があります。つまり、`deinit` が呼ばれるタイミングが遅れます。少し時間が経過した後や、他の処理が進んだ後に、ようやくインスタンスが解放されるという挙動になります。

この現象を理解するには、SwiftのARCとKotlin/NativeのGCという、メモリ管理方式の違いを知る必要があります。

### 問題が発生する例

まずは、問題が発生するコードを見てみましょう。

以下の実装の場合、`callKmpFunction()` を実行した直後に画面を閉じても、`Foo` の `deinit` が呼ばれるまでに遅延が発生します。

```kotlin:Function.kt
fun function(obj: Any) {
    // ...
}
```

```swift:Foo.swift
class Foo {
    func callKmpFunction() {
        FunctionKt.function(obj: self)
    }

    deinit {
        print("deinit")
    }
}
```

以下のようにクロージャ（ラムダ）の形でKotlin側に渡し、その中で `self` を参照している場合も同様です。

```kotlin:Function.kt
fun function(provider: () -> String) {
    // ...
}
```

```swift:Foo.swift
class Foo {
    func callKmpFunction() {
        FunctionKt.function { String(describing: self) }
    }

    deinit {
        print("deinit")
    }
}
```

<details>
<summary>動作確認に利用したSwiftUIのコード</summary>

```swift
struct ContentView: View {
    @State private var isPresented = false

    var body: some View {
        Button("Show") {
            isPresented = true
        }
        .fullScreenCover(isPresented: $isPresented) {
            FooView(
                hide: {
                    isPresented = false
                }
            )
        }
    }
}

struct FooView: View {
    var hide: () -> Void
    @State private var foo = Foo()

    var body: some View {
        Button("Hide") {
            foo.callKmpFunction()
            hide()
        }
    }
}
```

</details>

### Swiftオブジェクトの生存期間がKotlinのGCに依存する

Swiftだけで完結する場合、インスタンスが参照されなくなった瞬間に解放されます。これは**ARC（Automatic Reference Counting）**という仕組みによるものです。ARCはインスタンスが何箇所から参照されているかをカウントし、参照されなくなった瞬間（カウントが0になったとき）にインスタンスを解放します。

一方、Kotlin/Nativeでは**GC（Tracing Garbage Collector）**という仕組みが使われています[^kotlin-new-memory-manager]。GCは、参照されなくなったオブジェクトを定期的に検出し、まとめて解放するアプローチをとります。

[^kotlin-new-memory-manager]: Kotlin/Native 1.7.20から、新しいメモリ管理方式（New Memory Manager）がデフォルトとなりGCに移行しました。それ以前は参照カウントベースの管理方式が採用されていました。

SwiftのARCは当然、Kotlin側からの参照もカウントします。Kotlin側にインスタンス（またはそれを参照したクロージャ）を渡すと、Kotlin/NativeからSwiftのインスタンスを参照している状態になります。

このとき、たとえKotlin側の関数の実行が終了しても、Kotlin側のGCが走り、そのラッパーオブジェクトが破棄されるまでは、Swift側のインスタンスへの参照（参照カウント）が残り続けます。

結果として、Swift側ではARCによる参照カウントが0にならず、`deinit` の呼び出しがKotlin/NativeのGCが実行されるタイミングまで遅れることになります。

## 予期せぬ不具合の可能性

この挙動は、単に「メモリの解放が少し遅れる」という性能面の話だけではなく、実害のある不具合につながる可能性があります。これは言語やフレームワークの欠陥ではなく、特性を理解していないことによる設計上のミスマッチが原因です。

例えば、`deinit` で「画面が閉じられたこと」を検出していたとします。そこで「アナリティクスイベントの送信」や「ユーザーへの何らかの通知」といった処理をする場合、`deinit` が遅れることで「閉じたはずなのにイベントが送られない」や「通知が来ない」といった挙動を引き起こします。

また、`LoggerKt.log { String(describing: self) }` のようなロガーを利用している場合も注意が必要です。「バグ調査のためにログを追加したら、`self` の寿命が延びてしまい、動作が変わってしまった」という事態にもなり得ます。

`deinit` を純粋なメモリ解放のためだけに利用している場合は大きな問題になりませんが、それ以外の副作用（イベント通知など）を期待している場合には特に注意が必要です。

## 対策

### 不要な参照を渡さない（SwiftのライフサイクルをKotlinに委ねない）

この問題を回避する最も確実な方法は、Kotlin側にSwiftのインスタンスを渡さないことです。

以下の実装例では、クロージャの中で `self` を参照するのではなく、Swift側であらかじめ評価し、評価後の値を渡すように修正しています。

これであれば、Kotlin側が受け取るのは評価結果であり、`Foo` インスタンスへの参照は含まれません。そのため、画面を閉じると同時に `Foo` のインスタンスが解放されます。`deinit` の呼び出しも遅れません。

```swift
func callKmpFunction() {
    // クロージャ作成前にselfを評価する
    // これによりKotlin側からselfへの参照がなくなる
    let str = String(describing: self)
    FunctionKt.function { str }

    // または、評価結果を直接渡すように関数を定義してもよい
    // FunctionKt.function(str: String(describing: self))
}
```

### 弱参照を利用する

`[weak self]` を使用して強参照を防ぐ方法もあります。これにより、Kotlin側が `self` を強参照しなくなるため、カウントが増えることはありません。

ただしこの方法では、Kotlin側からクロージャが実行されるタイミングによっては `self` がすでに解放されている可能性があります。そのため、`self` が `nil` になることを考慮した実装が必要です。

```swift
FunctionKt.function { [weak self] in
    String(describing: self)
}
```

## まとめ

- SwiftとKotlin/Nativeではメモリ管理方式が異なり、SwiftはARC、Kotlin/NativeはGCを採用している
- SwiftのインスタンスをKotlin側に渡すと、そのインスタンスの寿命はKotlinのGCに依存するようになる
- インスタンスの解放が遅れることは、`deinit` の呼び出しが遅れることを意味し、予期せぬ不具合につながる可能性がある
- インスタンスそのものではなく、必要最低限のデータ（値型など）だけをKotlin側に渡すことや、弱参照を利用することで、この問題を回避できる

## 参考文献

1. [Kotlin/Native memory management - Kotlin Documentation](https://kotlinlang.org/docs/native-memory-manager.html)
2. [Integration with Swift/Objective-C ARC - Kotlin Documentation](https://kotlinlang.org/docs/native-arc-integration.html)
