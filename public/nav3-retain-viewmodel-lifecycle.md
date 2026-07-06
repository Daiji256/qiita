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
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# Nav3においてViewModelとretainの生存期間は一部異なる（いいタイトルが思いつかない）

## はじめに

Composeには `remember`, `retain`, `rememberSaveable` によって状態を保持できる。これらは生存期間や利用目的が異なる。この3つで比較している間は違いを理解しやすく、`retain` は構成変更によるアクティビティの再作成では値が破棄されない特徴がある。

Androidアプリを開発する上で `androidx.lifecycle.ViewModel` が登場する。これも `retain` と似た特性がありアクティビティの再作成によって値が破棄されない。`ViewModel` はComposeの登場前から利用されていて、現在も利用されることが多い。

この記事では `retain` と `ViewModel` の画面遷移の文脈における生存期間の違いを整理する。また、`retain` の生存期間を調整するための方法を紹介する。

## 動作の確認

## `retain` の生存期間を確認する

### 構成変更で破棄されない

以下のコードを用いて動作を確認する。画面回転などの構成変更が発生したとき、`remember` で保持される `Foo` は破棄され再作成する。一方、`retain` や `viewModel` で保持される `Bar` と `Baz` は破棄されない。

```kotlin
@Composable
fun FirstScreen() {
    val foo: Foo = remember { Foo() }
    val bar: Bar = retain { Bar() }
    val baz: Baz = viewModel<Baz>()

    // ...
}
```

### 画面遷移で破棄される

Nav3により画面遷移した場合に、`retain` と `ViewModel` の生存期間は異なる。バックスタックが `[FirstNavKey, SecondNavKey]` のようして `SecondScreen` に遷移した場合、`FirstScreen` の `retain` で保持されていた `Bar` は破棄される。一方で、`ViewModel` の `Baz` は破棄されず、`FirstScreen` に戻ってきた後も同じ `Baz` インスタンスを参照できる。

なぜ、`ViewModel` はバックスタックにあれば保持され続けるかというと、`ViewModelStoreNavEntryDecorator` により生存期間がバックスタックと紐づくようになっているからである。以下に示すのは `ViewModelStoreNavEntryDecorator` の一部抜粋である。バックスタック全体で共通の `ViewModelStoreProvider` を持ち、`ViewModel` はそこに格納されることになる。バックスタックの破棄に合わせて（`onPop` で）不要になった `ViewModel` を破棄することで、この生存期間を実現している。

```kotlin
@Composable
public fun <T : Any> rememberViewModelStoreNavEntryDecorator(
    viewModelStoreProvider: ViewModelStoreProvider
): ViewModelStoreNavEntryDecorator<T> {
    return remember(viewModelStoreProvider) {
        ViewModelStoreNavEntryDecorator(viewModelStoreProvider)
    }
}

public class ViewModelStoreNavEntryDecorator<T : Any>(
    viewModelStoreProvider: ViewModelStoreProvider,
) : NavEntryDecorator<T>(
    onPop = { key -> viewModelStoreProvider.clearKey(key) },
    decorate = { entry ->
        val owner = rememberViewModelStoreOwner(
            entry.contentKey,
            viewModelStoreProvider,
            // ...
        )
        CompositionLocalProvider(LocalViewModelStoreOwner provides owner) { 
            entry.Content()
        }
    },
)
```

## バックスタックにある間は `retain` でも保持し続けるようにする

ここで紹介する実装は[nav3-recipes の Retain Recipe](https://github.com/android/nav3-recipes/tree/0f978fac1e1f6501205528112a452256c9b2013b/app/src/main/java/com/example/nav3recipes/retain)にて紹介されている。

`retain` でも `ViewModel` と同じようにバックスタックにある間保持するには、同じように `NavEntryDecorator` を実装すればよい。`retain` には `RetainedValuesStore` / `RetainedValuesStoreRegistry` という保持するための機能がある。これは `ViewModel` における `ViewModelStore`/`ViewModelStoreOwner` と似たようなものと認識してよい。

`RetainedValuesStoreRegistry` をバックスタック全体で共有し、各画面の `retain` で参照される `RetainedValuesStore` の指定やバックスタックの変化に合わせた値の破棄を対応すれば実現できる。具体的には、以下の実装になる。`ViewModelStoreNavEntryDecorator` と概ね同じような実装である。

```kotlin
@Composable
fun <T : Any> rememberRetainedValuesStoreNavEntryDecorator(
    registry: RetainedValuesStoreRegistry = retainRetainedValuesStoreRegistry(),
): RetainedValuesStoreNavEntryDecorator<T> =
    remember(registry) {
        RetainedValuesStoreNavEntryDecorator(registry)
    }

class RetainedValuesStoreNavEntryDecorator<T : Any>(
    registry: RetainedValuesStoreRegistry,
) : NavEntryDecorator<T>(
    onPop = { key ->
        registry.clearChild(key)
    },
    decorate = { entry ->
        registry.LocalRetainedValuesStoreProvider(entry.contentKey) {
            entry.Content()
        }
    },
)
```

## まとめ

- `retain` と `ViewModel` は画面遷移の文脈においては生存期間が異なる
- 一般的には `retain` は画面遷移で破棄されるが、`ViewModel` はバックスタックに含まれる間は保持される
- バックスタックにある間は値を保持したい場合、`RetainedValuesStoreRegistry` を使うとより

## 参考文献

- [Retain Recipe | android/nav3-recipes | GitHub](https://github.com/android/nav3-recipes/tree/0f978fac1e1f6501205528112a452256c9b2013b/app/src/main/java/com/example/nav3recipes/retain)
- [retain | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/retain/retain.composable)
- [ViewModelStoreNavEntryDecorator | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/lifecycle/viewmodel/navigation3/ViewModelStoreNavEntryDecorator)
