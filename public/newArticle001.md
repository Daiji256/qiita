---
title: newArticle001
tags:
  - ''
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# Swift/Objective-C から Kotlin へインスタンスを渡すと `deinit` が遅れる理由と対策

この記事では Swift を例に説明しますが、Objective-C でも同様の挙動となります。

Kotlin Multiplatform（KMP）で iOS アプリを開発する際、Swift と Kotlin の相互運用は避けて通れません。KMP は非常に便利ですが、トラブルなく使いこなすには Swift と Kotlin 詳しく理解している必要があります。

この記事では、両者のメモリ管理の違いを簡単に紹介した後、この違いを理解していないと Swift エンジニアにとって「不可解」に見える挙動について解説します。

具体的には「Swift から Kotlin の関数等にインスタンスやその参照を渡すと、Swift 側の解放が遅延する」という事象です。

## インスタンスの解放が遅れる（`deinit` が遅れる）

例えば SwiftUI で画面を閉じたら、その画面に関連するインスタンスは即座に解放されてほしいです。Swift エンジニアとしては、それが自然な挙動だと感じることが多いでしょう。

しかし、Kotlin への関数呼び出しなどでインスタンスを渡している場合、画面を閉じてもすぐにはインスタンスが解放されない場合があります。つまり、`deinit` が呼ばれるタイミングが遅れます。少し時間が経過したり、他の処理が進んだ後に、ようやくインスタンスが解放されるという挙動になります。

この現象を理解するには、Swift の ARC と Kotlin/Native の GC という、メモリ管理方式の違いを知る必要があります。

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

以下のようにクロージャ（ラムダ）の形で Kotlin 側に渡し、その中で `self` を参照している場合も同様です。

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
<summary>動作確認に利用した SwiftUI のコード</summary>

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

### 参照する期間が Kotlin の GC に依存する

Swift だけで完結する場合、インスタンスが参照されなくなった瞬間に解放されます。これは **ARC（Automatic Reference Counting）** という仕組みによるものです。ARC はインスタンスが何箇所から参照されているかをカウントし、参照されなくなった瞬間（カウントが 0 になったとき）にインスタンスを解放します。

一方、Kotlin/Native では **GC（Tracing Garbage Collector）** という仕組みが使われています[^kotlin-drc]。GC は、参照されなくなったオブジェクトを定期的に検出し、まとめて解放するアプローチをとります。

[^kotlin-drc]: Kotlin/Native 1.7.20 から、新しいメモリ管理方式（New Memory Manager）がデフォルトとなり GC に移行しました。それ以前は参照カウントベースの管理方式が採用されていました。

Swift の ARC は当然、Kotlin 側からの参照もカウントします。Kotlin 側にインスタンス（またはそれを参照したクロージャ）を渡すと、Kotlin/Native から Swift のインスタンスを参照している状態になります。

このとき、たとえ Kotlin 側の関数の実行が終了しても、Kotlin の GC サイクルが回り「このオブジェクトはもう不要だ」と判断するまでは、Swift 側のインスタンスへの参照が残り続けます。

結果として、Swift 側の ARC カウントが 0 にならず、`deinit` の呼び出しが Kotlin の GC タイミングまで遅れることになるのです。

## 予期せぬ不具合の可能性

この挙動は、単に「メモリの解放が少し遅れる」という性能面の話だけではなく、実害のある不具合につながる可能性があります。これは言語やフレームワークの欠陥ではなく、特性を理解していないことによる設計上のミスマッチが原因です。

例えば、`deinit` で「画面が閉じられたこと」を検出していたとします。そこで「アナリティクスイベントを送信」や「ユーザーへ何らかの通知を送る」といった処理をする場合、`deinit` が遅れることで「閉じたはずなのにイベントが送られない」や「通知が来ない」といった挙動を引き起こします。

また、`LoggerKt.log { String(describing: self) }` のようなロガーを利用している場合も注意が必要です。「バグ調査のためにログを追加したら、`self` の寿命が延びてしまい、動作が変わってしまった」という事態にもなり得ます。

`deinit` を純粋なメモリ解放のためだけに利用している場合は大きな問題になりませんが、それ以外の副作用（イベント通知など）を期待している場合には特に注意が必要です。

## 対策

### 不要な参照を渡さない

この問題を回避する最も確実な方法は、Kotlin 側に Swift のインスタンスを渡さないことです。

以下の実装例では、クロージャの中で `self` を参照するのではなく、Swift 側であらかじめ評価し、評価後の値を渡すように修正しています。

これであれば、Kotlin 側が受け取るのは評価結果であり、`Foo` インスタンスへの参照は含まれません。そのため、画面を閉じると同時に `Foo` のインスタンスが解放されます。`deinit` の呼び出しも遅れません。

```swift
func callKmpFunction() {
    // クロージャ作成前に self を評価する
    // これにより Kotlin 側から self への参照がなくなる
    let str = String(describing: self)
    FunctionKt.function { str }

    // または、評価結果を直接渡すように関数を定義してもよい
    // FunctionKt.function(str: String(describing: self))
}
```

### 弱参照を利用する

`[weak self]` を使用して強参照を防ぐ方法もあります。これにより、Kotlin 側が `self` を強参照しなくなるため、カウントが増えることはありません。

しかし、この方法は実行されるときには `self` が解放される可能性があるため、`self` が `nil` になる可能性を考慮した実装にする必要があります。

```swift
FunctionKt.function { [weak self] in
    String(describing: self)
}
```

## まとめ

- Swift と Kotlin/Native ではメモリ管理方式が異なり、Swift は ARC、Kotlin/Native は GC を採用している
- Swift のインスタンスを Kotlin 側に渡すと、そのインスタンスの寿命は Kotlin の GC に依存するようになる
- インスタンスの解放が遅れることは、`deinit` の呼び出しが遅れることを意味し、予期せぬ不具合につながる可能性がある
- インスタンスそのものではなく、必要最低限のデータ（値型など）だけを Kotlin 側に渡すことや、弱参照を利用することで、この問題を回避できる

## 参考文献

1. [Kotlin/Native memory management - Kotlin Documentation](https://kotlinlang.org/docs/native-memory-manager.html)
2. [Integration with Swift/Objective-C ARC - Kotlin Documentation](https://kotlinlang.org/docs/native-arc-integration.html)
