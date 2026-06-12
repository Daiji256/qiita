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

# Google Play Consoleの「デバッグシンボルがありません」を解決する

## はじめに

Androidアプリをリリース・アップデートしようとしたとき、Google Play Console上に以下のワーニングが表示されたことはないでしょうか？

本記事では、Android App Bundle（`.aab` ファイル）にデバッグシンボルを含めるようにすることで、この問題を解決する方法を紹介します。

> This App Bundle contains native code, and you've not uploaded debug symbols. We recommend you upload a symbol file to make your crashes and ANRs easier to analyze and debug.

> この App Bundle にはネイティブ コードが含まれ、デバッグ シンボルがアップロードされていません。クラッシュや ANR を簡単に分析、デバッグできるよう、シンボル ファイルをアップロードすることをおすすめします。

![TODO](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/7247fe64-8ede-45c0-abcb-e0403dd466c9.png)

## デバッグシンボルとは何か？

TODO: 簡単に説明

## 必要な設定

`app/build.gradle.kts` に `ndkVersion` と `debugSymbolLevel` を設定することで、ビルドされる `.aab` ファイルにデバッグシンボルが含まれるようになります。

`ndkVersion` を設定しなかったところ、`app/build/intermediates/merged_native_libs/.../out/lib` にデバッグシンボルは生成されますが、`.aab` には含まれませんでした。`.aab` にデバッグシンボルを含める処理がスキップされないためには、`ndkVersion` の指定が必要です。

`ndkVersion` の `27.3.13750724` は執筆時におけるLTSバージョンの最新です。

```kotlin
android {
    ndkVersion = "27.3.13750724"

    buildTypes {
        release {
            ndk.debugSymbolLevel = "FULL"
        }
    }
}
```

## NDKをインストールする方法

1. Tools > SDK Managerを選択
2. SDK ToolsからインストールするNDKのバージョンにチェックを入れる
3. Apply

TODO: ここにスクショを載せる

## 成功の確認方法

ビルドして得られた `.aab` ファイルをZIP解凍すると、`BUNDLE-METADATA/com.android.tools.build.debugsymbols` が含まれているはずです。

それでは、`.aab` ファイルをPlay Consoleにアップロードしてみましょう。警告が消えるだけでなく、App BundleのDownloadsタブを確認すると「Native debug symbols」に `native-debug-symbols.zip` が追加されています。

![TODO](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/e41a31f2-209d-43eb-a968-8b4fbcdafbc0.png)

## まとめ

- デバッグシンボルとは... TODO
- `.aab` にデバッグシンボルを同梱するには `app/build.gradle.kts` に `ndkVersion` と `ndk.debugSymbolLevel = "FULL"` を設定する

## 参考文献

- https://developer.android.com/topic/performance/app-optimization/enable-app-optimization#native-crash-support
- https://developer.android.com/ndk/downloads/
- TODO: 他にも参考にした文献があればここに書く
