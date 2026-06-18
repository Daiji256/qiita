---
title: newArticle001
tags:
  - Android
  - Typography
  - Font
  - chws
  - 約物
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# chws-toolにより任意のフォントをchws対応にする

Android 12から標準搭載されているNoto Sans/Serif CJKは、`chws`（Contextual Half-width Spacing）機能に対応しました。これにより、括弧や句読点などの約物のアキが調整されるようになり、より美しい文字組みが実現されています。最近では `chws` に対応したフォント・テキスト描画システムが増えつつあり、ユーザー側も自然な文字組みを期待するようになっています。

## `chws` に対応していないフォント

`chws` に対応するフォントが増えてきたといっても、いまだに対応していないフォントはたくさんあります。また、Google FontsからダウンロードできるNoto Sans JPもstaticに格納されたウエイトごとのフォントの方は、`chws` に対応していません[^google-fonts-noto-sans-jp-variable]。

まだ `chws` に対応していないフォントを使いたい場面、Noto Sans JPのウエイトがstaticなものを使いたい場面はあります。このとき、約物のアキが不自然に残ってしまい、もどかしい思いになります。

[^google-fonts-noto-sans-jp-variable]: バリアブルフォントは `chws` に対応しています。

## `chws-tool` による解決

任意のフォントを `chws` 対応させるには、Google Fontsチームが公開している [`chws-tool`](https://github.com/googlefonts/chws_tool) が非常に便利です。実際にNoto CJKも `chws-tool` によって `chws` 対応が行われているようです。

`chws-tool` により、指定したフォントファイルを解析し `chws`, `vchw`, `halt`, `vhal` を追加してくれます。

Pythonの環境に[uv](https://docs.astral.sh/uv/)を利用している場合、`chws-tool` は `uvx` で簡単に実行できます：

```sh
uvx --from chws-tool add-chws font-file.ttf
```

## 実際に確認

実際に `chws` に対応していない「Zen Maru Gothic」を `chws` に対応させてみました。

![Zen Maru Gothicにchws対応する前のした後の比較。対応後は約物間のアキが調整される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/3b9ceb77-59b6-4e06-b156-2ee66448a82c.png)

## 参考文献

- [googlefonts/chws_tool](https://github.com/googlefonts/chws_tool)
- [notofonts/noto-cjk](https://github.com/notofonts/noto-cjk)
- [googlefonts/zen-marugothic](https://github.com/googlefonts/zen-marugothic)
- [uv](https://docs.astral.sh/uv/)
