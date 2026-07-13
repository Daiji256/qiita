---
title: chws-toolにより任意のフォントをchws対応にする
tags:
  - Android
  - font
  - typography
  - 約物
  - chws
private: false
updated_at: '2026-07-04T08:40:09+09:00'
id: 69cfafbd3b62e7d45319
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: 783b7a849caf11eefd91
agreed_posting_campaign_term: true
---

## はじめに

Android 12から標準搭載されているNoto Sans/Serif CJKは、chws（Contextual Half-width Spacing）機能に対応しました。これにより、括弧や句読点などの約物のアキが自動で調整されるようになり、より美しい文字組みが実現されています。最近では `chws` に対応したフォントやテキスト描画システムが増えつつあり、ユーザー側も自然な文字組みを期待するようになっています。

## `chws` に対応していないフォント

`chws` 対応フォントが増えてきたとはいえ、いまだに対応していないフォントは数多く存在します。例えば、Google FontsからダウンロードできるNoto Sans JPでも、ウエイトごとに個別で提供されている静的フォント（Static Font）は `chws` に対応していません[^google-fonts-noto-sans-jp-variable]。

バリアブルフォントが利用できない環境のため静的フォントを使わざるを得ない場面や、そもそも `chws` に対応していないフォントを使いたい場面は依然としてあります。このようなとき、約物のアキが不自然に残ってしまい、もどかしく感じる開発者は多いでしょう。

[^google-fonts-noto-sans-jp-variable]: バリアブルフォントの方は `chws` に対応しています。

## `chws-tool` による解決

任意のフォントを `chws` 対応させるには、Google Fontsチームが公開している [`chws-tool`](https://github.com/googlefonts/chws_tool) が非常に便利です。実際のNoto CJKフォントも `chws-tool` によって `chws` 対応が行われているようです。

`chws-tool` は、指定したフォントファイルを解析し、`chws`, `vchw`, `halt`, `vhal` といったOpenType機能を追加します。Pythonのパッケージマネージャである[uv](https://docs.astral.sh/uv/)を利用している環境であれば、`chws-tool` は `uvx` コマンドで簡単に実行できます：

```sh
uvx --from chws-tool add-chws font-file.ttf
```

## 実際に確認

実際に `chws` に対応していない「Zen Maru Gothic」を `chws` に対応させてみました。

![Zen Maru Gothicのchws未対応・対応の比較。対応後は約物間のアキが調整される。](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/699841/3b9ceb77-59b6-4e06-b156-2ee66448a82c.png)

## 参考文献

- [googlefonts/chws\_tool](https://github.com/googlefonts/chws_tool)
- [notofonts/noto-cjk](https://github.com/notofonts/noto-cjk)
- [googlefonts/zen-marugothic](https://github.com/googlefonts/zen-marugothic)
- [uv](https://docs.astral.sh/uv/)
