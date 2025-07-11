---
title: TODO
tags:
  - Android
  - Kotlin
  - DI
  - JetpackCompose
  - Hilt
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# Hilt で Composable に DI する

Composable 関数内で、Hilt で provides / binds したオブジェクトを扱いたいことがあります。Composable と Hilt を使っている場合、多くの場合は ViewModel を利用します。`@HiltViewModel` でアノテートされた ViewModel に依存関係を inject することで、Composable からも間接的にそのオブジェクトを利用できます。

しかし、設計の都合などにより、ViewModel を経由したくない場合もあります。また、`CompositionLocal` も使いたくない場合もあります。

そこで、本記事では Composable で直接的に Hilt で provides / binds されたオブジェクトを扱う方法を紹介します。

## エントリポイント

Hilt は Composable への直接的な依存関係の注入に対応していないため、inject の対象をエントリポイントとして定義する必要があります。

エントリポイントの作成には `@EntryPoint` アノテーションを使用します。エントリポイント内は Hilt の管理対象となり、オブジェクトのグラフに入ります。これにより Hilt の管理対象外の領域から、エントリポイントを通して依存関係にアクセスできます。

たとえば、`SingletonComponent` にインストールするエントリポイントは、このように定義できます：

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FooEntryPoint {
    val foo: Foo
}
```

## Composable 内でエントリポイントを取得

エントリポイントにアクセスするには、`EntryPointAccessors` を使用します。エントリポイントがどこにインストールされているかで、引数が変わります。`SingletonComponent` にインストールされている場合は、`ApplicationContext` を使用します。そして、エントリポイントから扱いたいオブジェクトを取得します。

Composable 内でこれを行いたい場合は、`LocalContext` により `ApplicationContext` を取得します：

```kotlin
@Composable
fun rememberFoo(): Foo {
    val context = LocalContext.current
    return remember {
        EntryPointAccessors.fromApplication<FooEntryPoint>(context.applicationContext).foo
    }
}
```

複数のエントリポイントを扱う場合は、Composable 内でエントリポイントを取得するための関数を定義すると良いでしょう：

```kotlin
@Composable
inline fun <reified T : Any> rememberSingletonEntryPoint(): T {
    val context = LocalContext.current
    return remember {
        EntryPointAccessors.fromApplication<T>(context.applicationContext)
    }
}
```

## Composable Scoped

前節では `SingletonComponent` を利用しました。しかし、Composable 用のスコープで扱いたい場合もあるでしょう。その場合は、専用のスコープを定義します：

```kotlin
@Scope
annotation class ComposableScoped

@ComposableScoped
@DefineComponent(parent = ActivityComponent::class)
interface ComposableComponent
```

Composable からアクセスする場合は、`SingletonComponent` の時と同様にエントリポイントを用います。`ComposableComponent` は `ActivityComponent` 下にあるため、`ActivityContext` が必要です。

Composable 内から `ComposableComponent` にインストールしたエントリポイントにアクセスするには、以下のように実装します：

```kotlin
@Composable
inline fun <reified T : Any> rememberComposableEntryPoint(): T {
    val activity = checkNotNull(LocalActivity.current)
    return remember {
        val builderEntryPoint =
            EntryPointAccessors.fromActivity<ComposableComponentBuilderEntryPoint>(activity)
        EntryPoints.get(builderEntryPoint.builder.build(), T::class.java)
    }
}

@DefineComponent.Builder
interface ComposableComponentBuilder {
    fun build(): ComposableComponent
}

@EntryPoint
@InstallIn(ActivityComponent::class)
interface ComposableComponentBuilderEntryPoint {
    val builder: ComposableComponentBuilder
}
```

## 注意点

- エントリポイントを使用すると、Hilt の自動的な依存関係管理の恩恵を一部失うことになります
- パフォーマンスの観点から、`remember` を使用してオブジェクトの再作成を避けることが重要です
- カスタムスコープを使用する場合は、適切なライフサイクル管理が必要です

## まとめ

- エントリポイントを使うことで Composable から Hilt で provides / binds したオブジェクトを取得できる
- エントリポイントは `EntryPointAccessors` とスコープにあった `Context` によりアクセスできる
- Composable 用のスコープを扱いたい場合は、独自で定義できる
- エントリポイントの使用はパフォーマンスや設計への影響を考慮して慎重に行う

## 参考

https://dagger.dev/hilt/entry-points.html

https://dagger.dev/api/latest/dagger/hilt/DefineComponent.html

https://developer.android.com/training/dependency-injection/hilt-android

https://github.com/Daiji256/android-showcase
