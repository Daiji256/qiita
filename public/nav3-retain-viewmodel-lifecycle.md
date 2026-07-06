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

# Nav3における `retain` と `ViewModel` の生存期間の違いと調整方法

## はじめに

Composeでは、`remember`、`retain`、`rememberSaveable` などの関数を用いて状態を保持できます。これらはそれぞれ生存期間や利用目的が異なりますが、中でも `retain` は、画面回転などの構成変更によるアクティビティの再作成時にも値が破棄されないという特徴を持っています。

一方、Androidアプリ開発で従来から広く使われている `androidx.lifecycle.ViewModel` も、アクティビティの再作成時に値が破棄されないという `retain` と似た特性を持っています。`ViewModel` はComposeの登場前から利用されており、現在でもアーキテクチャの要として採用されることが多いコンポーネントです。

この記事では、Navigation 3（Nav3）を用いた画面遷移の文脈において、`retain` と `ViewModel` の生存期間がどのように異なるのかを整理します。また、`retain` の生存期間を調整し、`ViewModel` のようにバックスタックにある間も値を保持し続ける方法を紹介します。

## `retain` と `ViewModel` と生存期間の比較

### 構成変更で破棄されない（共通の挙動）

まずは以下のコードを用いて、基本的な動作を確認します。
画面回転などの構成変更が発生したとき、`remember` で保持される `Foo` は破棄され、再作成されます。一方、`retain` や `viewModel` で保持される `Bar` と `Baz` は破棄されません。

```kotlin
@Composable
fun FirstScreen() {
    val foo: Foo = remember { Foo() }
    val bar: Bar = retain { Bar() }
    val baz: Baz = viewModel<Baz>()

    // ...
}
```

### `retain` は画面遷移で破棄される

Nav3を用いて画面遷移を行った場合、`retain` と `ViewModel` の生存期間には違いがあります。

たとえば、`FirstScreen` から `SecondScreen` へ遷移し、バックスタックが `[FirstNavKey, SecondNavKey]` の状態になったとします。このとき、`FirstScreen` 内の `retain` で保持されていた `Bar` は破棄されます。

一方で、`ViewModel` の `Baz` は破棄されません。そのため、`SecondScreen` からバック操作で `FirstScreen` へ戻ってきた後も、同じ `Baz` インスタンスを参照し続けることができます。

なぜ、`ViewModel` はバックスタックにある間保持され続けるかというと、Nav3の `ViewModelStoreNavEntryDecorator` によって、ViewModelの生存期間がバックスタックと紐づくように設計されているためです。

以下のコードは、`ViewModelStoreNavEntryDecorator` の実装の一部抜粋です。これを見ると、バックスタック全体で共通の `ViewModelStoreProvider` を保持しており、各画面の `ViewModel` はそこに格納されます。そして、バックスタックからエントリが破棄されるタイミング（`onPop`）に合わせて、不要になった `ViewModel` を破棄することで、画面遷移に合わせた生存期間を実現しています。

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

ここで紹介する実装アプローチは、公式の[nav3-recipes にある Retain Recipe](https://github.com/android/nav3-recipes/tree/0f978fac1e1f6501205528112a452256c9b2013b/app/src/main/java/com/example/nav3recipes/retain)にて紹介されている手法です。

`retain` で保持する値も `ViewModel` と同じようにバックスタックにある間維持したい場合は、先ほどの `ViewModel` と同じように `NavEntryDecorator` を独自に実装すれば解決します。

`retain` の裏側には、値を保持するための `RetainedValuesStore` や `RetainedValuesStoreRegistry` という仕組みが存在します。これは `ViewModel` における `ViewModelStore` や `ViewModelStoreOwner` と同じような役割を持つと考えてよいでしょう。

この `RetainedValuesStoreRegistry` をバックスタック全体で共有し、各画面の `retain` で参照される `RetainedValuesStore` の指定や、バックスタックのポップ（破棄）に合わせた値のクリアを行えば実現できます。

具体的な実装は以下のようになります。`ViewModelStoreNavEntryDecorator` と概ね同じような構造になっていることがわかります。

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

- `retain` と `ViewModel` は、画面遷移の文脈においては生存期間が異なる
- 一般的に `retain` は画面を離れる（遷移する）と破棄されるが、`ViewModel` はバックスタックに含まれている間はインスタンスが保持され続ける
- `retain` を用いたバックスタックにある間も値を保持は、`RetainedValuesStoreRegistry` と独自の `NavEntryDecorator` により実現できる

## 参考文献

- [Retain Recipe | android/nav3-recipes | GitHub](https://github.com/android/nav3-recipes/tree/0f978fac1e1f6501205528112a452256c9b2013b/app/src/main/java/com/example/nav3recipes/retain)
- [retain | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/compose/runtime/retain/retain.composable)
- [ViewModelStoreNavEntryDecorator | API reference | Android Developers](https://developer.android.com/reference/kotlin/androidx/lifecycle/viewmodel/navigation3/ViewModelStoreNavEntryDecorator)
