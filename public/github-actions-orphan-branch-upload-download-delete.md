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

# GitHub Actions で orphan ブランチを使ってファイルを保存・取得する方法

本記事は、[GitHub Actions Advent Calendar 2025](https://qiita.com/advent-calendar/2025/github-actions) の 19 日目の記事です。

## はじめに

GitHub Actions で生成されたファイルを保存する代表的な手段として、GitHub Actions の Artifacts や S3 などの外部のストレージサービスの利用が挙げられます。しかし、Artifacts や外部ストレージでは、いくつかの制約や不便さを感じることがあります。

そこで本記事では、orphan ブランチを利用してファイルを保存・取得するというアプローチと、それを GitHub Actions から利用する方法を紹介します。

## 親を持たない orphan ブランチ

Git では orphan ブランチという、親を持たない空のブランチを作成できます：

```bash
git switch --orphan <branch-name>
```

通常のブランチとは異なり、履歴が完全に分離されています。このため、GitHub に orphan ブランチをプッシュしていたとしても、後で削除すればその履歴ごとリポジトリから取り除かれ、リポジトリの肥大化を防ぐことができます。この性質により、orphan ブランチを生成物の保存場所として利用できます。

## ファイル管理に orphan ブランチを使う利点と欠点

Artifacts 機能は手軽で便利ですが、ワークフロー実行単位で管理されるため、ワークフローを跨いでファイルを参照が難しいです。また、保存期間にも限りがあります。また、プルリクエストの説明文などからファイルを直接参照することはできません。

外部のストレージサービスは費用面や認証情報の管理が課題となることがあります。利用用途や環境によってはやや大げさに感じられることもあります。また、GitHub だけで完結しないのも欠点です。

これらと比べると、orphan ブランチは GitHub 上で完結し、ワークフローに依存せずにファイルを管理でき、保存期間に制限がない点が利点です。また、通常のブランチと同様に、`raw.githubusercontent.com` 経由でファイルを参照できます。そのため、プルリクエストの説明文などに直接画像を埋め込むことも可能です。

一方で、orphan ブランチを使う場合、ブランチの管理は手動で行う必要があります。ブランチが多数あるとリポジトリの管理が煩雑になったり、不要なブランチを削除し忘れるとリポジトリが肥大化する恐れがあります。また、大きなファイルの保存には適しません。

例えば、実装前の UI と実装後の UI のスクリーンショットを orphan ブランチに保存し、プルリクエストの説明文で比較表示することができます。スクリーンショットに限らず、実装前後のカバレッジやパフォーマンスなどのレポートを保存・比較する用途にも適しています。

## 作成した GitHub Actions の紹介

ファイルの保存・取得・削除のたびに、いくつかのコマンドを扱うのは大変です。そこで、orphan ブランチを用いたファイル管理を簡単に行うための GitHub Actions を作成しました。

詳しくは、それぞれの Actions の実装や[サンプル](https://github.com/Daiji256/orphan-branch-upload-download-delete-examples)を参照してください。

### [`daiji256/upload-to-orphan-branch`](https://github.com/Daiji256/upload-to-orphan-branch)

ワークフロー内で生成されたファイルを、指定した orphan ブランチにコミットしてプッシュする Action です。生成物を通常の開発履歴から分離したまま、リポジトリ内に保存できます。

`outputs-branch-name` という orphan ブランチに `**/bar` を除いた `dir` を保存する例は以下の通りです：

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

`outputs-branch-name` というブランチからファイルを取得し、`dir` に展開する例は以下の通りです：

```yaml
- uses: daiji256/download-from-orphan-branch@v1
  with:
    branch: outputs-branch-name
    path: dir
```

### [`daiji256/delete-orphan-branch`](https://github.com/Daiji256/delete-orphan-branch)

不要になった orphan ブランチを削除するための Action です。生成物ごとにブランチを作成する運用では、定期的なクリーンアップが重要になります。

`orphan-output-` で始まるブランチのうち、7 日以上前に作成されたものを削除する例は以下の通りです：

```yaml
- uses: daiji256/delete-orphan-branch@v1
  with:
    branch-regex: orphan-output-.*
    older-than-seconds: 604800 # 7 days
```

## 現状の課題と今後の改善

現状の実装にはいくつかの課題があります。Linux 環境での動作確認に留まっている点や、git コマンド失敗時などのエラーハンドリングが十分ではありません。

もしやる気が出たら、TypeScript での再実装や、macOS・Windows を含めた環境での動作確認、エラーハンドリングおよびログ出力の強化を進めていきたいと考えています。

## おわりに

ファイルの管理に orphan ブランチを使う方法は、すべてのケースに適したわけではありませんが、GitHub Actions の中だけで生成物を扱い、後から参照できる形で残したい場合には有効な選択肢です。

ぜひ orphan ブランチを活用したファイル管理を試してみてください。また、今回紹介した GitHub Actions も活用してもらえれば幸いです。

## 参考文献

- [Git - git-switch Documentation](https://git-scm.com/docs/git-switch)
- [GitHub Actions documentation - GitHub Docs](https://docs.github.com/en/actions)
- [daiji256/upload-to-orphan-branch](https://github.com/Daiji256/upload-to-orphan-branch)
- [daiji256/download-from-orphan-branch](https://github.com/Daiji256/download-from-orphan-branch)
- [daiji256/delete-orphan-branch](https://github.com/Daiji256/delete-orphan-branch)
- [Orphan Branch Upload / Download / Delete Examples](https://github.com/Daiji256/orphan-branch-upload-download-delete-examples)
- [Daiji256/android-showcase/.github/workflows/unit-test.yml](https://github.com/Daiji256/android-showcase/blob/1010840b0eb7873f50a405d6199e904ae2f0a28d/.github/workflows/unit-test.yml)
