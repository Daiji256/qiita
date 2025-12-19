---
title: newArticle001
tags:
  - GitHubActions
  - GitHub
  - Git
  - CICD
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# GitHub Actions で orphan ブランチを使ってファイルを保存・取得する方法

本記事は、[GitHub Actions Advent Calendar 2025](https://qiita.com/advent-calendar/2025/github-actions) の 19 日目の記事です。

## はじめに

GitHub Actions で生成されたファイルを保存する代表的な手段として、GitHub Actions の Artifacts や、S3 などの外部ストレージサービスの利用が挙げられます。しかし、Artifacts や外部ストレージには、用途によってはいくつかの制約や不便さを感じる場面があります。

そこで本記事では、orphan ブランチを利用してファイルを保存・取得するというアプローチと、それを GitHub Actions から利用する方法を紹介します。

## 親を持たない orphan ブランチ

Git では `git switch --orphan <branch-name>` などのコマンドを使うことで、親を持たない空のブランチ（orphan ブランチ）を作成できます。

通常のブランチとは異なり、履歴が完全に分離されているのが特徴です。そのため、GitHub に orphan ブランチをプッシュしていたとしても、後でブランチを削除すれば、その履歴ごとリポジトリから取り除かれます。これにより、不要になった生成物をリポジトリから取り除き肥大化を防ぐことができます[^git-gc]。

[^git-gc]: ブランチ削除後すぐに完全に消えるわけではなく、`git gc` が実行されるまでは内部にデータが残ります。頻繁に大容量ファイルを追加・削除する運用では注意が必要です。

この性質を利用することで、orphan ブランチを生成物の保存場所として扱えます。

## ファイル管理に orphan ブランチを使う利点と欠点

Artifacts 機能は手軽で便利ですが、ワークフロー実行単位で管理されるため、ワークフローをまたいでファイルを参照することが難しいという側面があります。また、保存期間にも制限があります。さらに、プルリクエストの説明文などからファイルを直接参照することはできません。

外部ストレージサービスを利用する場合、費用面や認証情報の管理が課題になることがあります。利用する用途や環境によっては、やや大げさに感じられることもあります。また、GitHub だけで完結しない点もデメリットです。

これらと比べると、orphan ブランチは GitHub 上で完結し、ワークフローに依存せずにファイルを管理できます。また、保存期間に制限がない点も利点です。さらに、通常のブランチと同様に `raw.githubusercontent.com` や `raw=true` を利用してファイルを参照できるため、プルリクエストの説明文に直接画像を埋め込むことも可能です。

一方で、orphan ブランチを利用する場合、ブランチの管理は利用者側で行う必要があります。ブランチ数が増えるとリポジトリの管理が煩雑になり、不要なブランチを削除し忘れると、結果としてリポジトリが肥大化する恐れがあります。また、大容量ファイルの保存には適していません。

例えば、実装前と実装後の UI のスクリーンショットを orphan ブランチに保存し、プルリクエストの説明文で比較表示するといった使い方が考えられます。スクリーンショットに限らず、実装前後のカバレッジやパフォーマンスレポートなどを保存・比較する用途にも適しています。

## 作成した GitHub Actions の紹介

ファイルの保存・取得・削除を行うたびに、`git switch --orphan` や `git add`、`git commit`、`git push` といったコマンドをワークフローに記述するのは手間がかかります。

そこで、これらの処理をまとめて行う GitHub Actions を作成しました。

### [`daiji256/upload-to-orphan-branch`](https://github.com/Daiji256/upload-to-orphan-branch)

ワークフロー内で生成されたファイルを、指定した orphan ブランチにコミットしてプッシュする Action です。

以下は、`outputs-branch-name` という orphan ブランチに、`**/bar` を除いた `dir` 配下のファイルを保存する例です。

```yaml
- uses: daiji256/upload-to-orphan-branch@v1
  with:
    branch: outputs-branch-name
    path: |
      dir
      !**/bar
```

### [`daiji256/download-from-orphan-branch`](https://github.com/Daiji256/download-from-orphan-branch)

指定したブランチからファイルを取得する Action です。別のワークフローで事前に保存したファイルを再利用できます。

以下は、`outputs-branch-name` というブランチからファイルを取得し、`dir` に展開する例です。

```yaml
- uses: daiji256/download-from-orphan-branch@v1
  with:
    branch: outputs-branch-name
    path: dir
```

### [`daiji256/delete-orphan-branch`](https://github.com/Daiji256/delete-orphan-branch)

不要になった orphan ブランチを削除するための Action です。前述のリポジトリ肥大化を防ぐためにも、生成物ごとにブランチを作成する運用では、定期的なクリーンアップが重要になります。

以下は、`orphan-output-` で始まるブランチのうち、7 日以上前に作成されたものを削除する例です。

```yaml
- uses: daiji256/delete-orphan-branch@v1
  with:
    branch-regex: orphan-output-.*
    older-than-seconds: 604800 # 7 days
```

## 現状の課題と今後の改善

現状の実装にはいくつか課題があります。Linux 環境での動作確認に留まっている点や、git コマンド失敗時などのエラーハンドリングが十分ではありません。

今後、TypeScript での再実装や、macOS・Windows を含めた環境での動作確認、エラーハンドリングおよびログ出力の強化を進めていきたいと考えています[^yaruki]。

[^yaruki]: やる気と時間があれば、という意味です。

## おわりに

GitHub Actions の中だけで生成物を扱い、後から参照できる形で残したい場合には、orphan ブランチを利用は有効な選択肢です。

用途と制約を理解した上で、orphan ブランチを活用したファイル管理を検討してみてください。また、今回紹介した GitHub Actions も、その一助になれば幸いです。

## 参考文献

- [Git - git-switch Documentation](https://git-scm.com/docs/git-switch)
- [GitHub Actions documentation - GitHub Docs](https://docs.github.com/en/actions)
- [daiji256/upload-to-orphan-branch](https://github.com/Daiji256/upload-to-orphan-branch)
- [daiji256/download-from-orphan-branch](https://github.com/Daiji256/download-from-orphan-branch)
- [daiji256/delete-orphan-branch](https://github.com/Daiji256/delete-orphan-branch)
- [Orphan Branch Upload / Download / Delete Examples](https://github.com/Daiji256/orphan-branch-upload-download-delete-examples)
- [Daiji256/android-showcase/.github/workflows/unit-test.yml](https://github.com/Daiji256/android-showcase/blob/1010840b0eb7873f50a405d6199e904ae2f0a28d/.github/workflows/unit-test.yml)
