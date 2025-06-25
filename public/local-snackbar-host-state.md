---
title: "LocalSnackbarHostState: CompositionLocal による SnackbarHostState の管理方法"
tags:
  - ""
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

# LocalSnackbarHostState: CompositionLocal による SnackbarHostState の管理方法

## Snackbar とは

- Snackbar について 3 つ程度の箇条書きで簡単に説明
- Toast と比較したい

## Compose における Snackbar

`SnackbarHost` と `SnackbarHostState` について軽く説明したい

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
```

```kotlin
Scaffold(
    snackbarHost = {
        SnackbarHost(
            hostState = snackbarHostState,
        )
    },
    // ...
)
```

```kotlin
val coroutineScope = rememberCoroutineScope()
Button(
    onClick = {
        coroutineScope.launch {
            snackbarHostState.showSnackbar(message = "message")
        }
    },
) {
    Text(text = "show snackbar")
}
```

## CompositionLocal とは

- CompositionLocal の説明を簡単にする

## CompositionLocal を使う例

### Scaffold に対して

```kotlin
@Composable
fun MyScaffold(
    snackbarHostState: SnackbarHostState = remember { SnackbarHostState() },
    // ...
) {
    CompositionLocalProvider(
        LocalSnackbarHostState provides snackbarHostState,
    ) {
        Scaffold(
            snackbarHost = {
                SnackbarHost(hostState = snackbarHostState)
            },
        ),
        // ...
    }
}
```

### composable に対して

```kotlin
inline fun <reified T : Any> NavGraphBuilder.myComposable(/* ... */) =
    composable<T> { entry ->
        val snackbarHostState = remember { SnackbarHostState() }
        CompositionLocalProvider(
            LocalSnackbarHostState provides snackbarHostState,
        ) {
            content(entry)
        }
    }
```

## メリット

## まとめ

## 参考文献

https://m3.material.io/components/snackbar/overview

https://developer.android.com/develop/ui/compose/components/snackbar

https://developer.android.com/develop/ui/compose/compositionlocal
