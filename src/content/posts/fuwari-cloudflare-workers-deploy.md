---
title: Fuwariでポートフォリオ用サイトを準備した
published: 2026-06-24
description: "fuwariテーマのAstroブログをGitHub Actions + Cloudflare Workersで自動デプロイするまでの全記録。躓いた点も含めて。"
tags: ["Node.js", "Astro", "Cloudflare", "GitHub"]
category: 技術
draft: true
---

# Fuwariでポートフォリオ用サイトを準備した

## はじめに

こんにちは、ひょろです。6月中旬から転職活動をしています。
エンジニアはポートフォリオとして自前のウェブサイトを用意することが一般的なようなので、自分もそれに倣い準備してみることにしました。
最初は静的なトップページと備忘録的な技術ブログをWordPressで構築していましたが、最終的に自分で運用する環境は完全にサーバーレスになってしまいました。
インフラエンジニアを目指す身としては本末転倒な気はしますが、経緯から詳しい導入手順まで書いていきます。

## 経緯

最初期は自宅のサーバーにWordPress環境と簡単なHTMLのページを準備していましたが、管理面で手間がかかるのと脆弱性を管理しきれないという理由で他の方法を探していました。
そんな時にChatGPTに[Astro](https://astro.build/)というSSGを教えてもらいました。
簡単に説明すると、Webサイトのページを公開前にすべてHTMLファイルとして生成しておく技術らしい。流れは以下の通り。

動的サイトの代表格であるWordPressはアクセス時にPHPが動作して、データベースから記事情報を取得してHTMLを生成します。
それに対してSSGは、アクセスが来る前に静的なTHMLサイトを事前に準備しておく。
アクセスしに来た人はすでに出来上がってるページを見て即終了って感じ。
これによってレスポンスの遅さや脆弱性を克服できるって仕組み。何でも知ってるAIってすごい。

最初はこのSSGすら自前のサーバーでビルドしてから公開していましたが、なんとGitHubとCloudflareの機能を使えば自前のサーバーを使わなくても実装できるらしい。しかも無料で。
というわけでGitHub Actionsという機能と、Cloudflare Workersという機能をふんだんに使っていきます。

そんな経緯で最終的に開発環境はサーバーのvimからMacに移りました。
gitでcloneできる環境ならwindowsとか他のpcでも良くなったのはすごい。

## 最終的な流れと構成

記事を書く
↓
git push,commit
↓
GitHub Actionsで自動ビルド
↓
Cloudflare Workersで自動デプロイ
↓
そのままCloudflareが[nonrem.net](https://nonrem.net)にプロキシして表示される

任意の端末でMarkdown記事を執筆してGitHubにプッシュすれば自動でサイトに反映される仕組み。
ここまで手ぶらで自動化できるのにドメイン料以外が無料なのは破格でヤバい。

| 項目 | 技術 |
|------|------|
| ブログテーマ | fuwari（Astro製） |
| ホスティング | Cloudflare Workers Static Assets |
| CI/CD | GitHub Actions + wrangler-action |
| ドメイン | nonrem.net（Cloudflare管理） |

### 選定理由

**fuwari（Astro）**
WordPress（PHP）は動的にページを生成するぶん、サーバーの処理とセキュリティ管理が重い。対してAstroは事前にHTMLを吐き出すSSGなので、軽量で速く、攻撃される面も少ない。テーマの見た目が好みだったのも大きい。

**Volta**
Node.jsのバージョン管理ツール。Rust製で動作が速いのと、プロジェクトごとに使うバージョンを自動で切り替えてくれるのが楽。正直この用途なら他のツール（nvm, fnm等）でも問題ないと思う。流行ってるので採用した。

**GitHub**
言わずもがな。今回は特にソース管理よりGitHub Actions（自動ビルド・デプロイ）の働きが主役。

**Cloudflare Workers**
当初はCloudflare Pagesを使う予定だったが、Cloudflareが新規プロジェクトの推奨をPagesからWorkersに移している。今から作るならWorkers一択。

**wrangler**
CloudflareのCLIツール。Workers/Pagesへのデプロイやログの確認などができる。GitHub Actionsからのデプロイにも内部的に使われている（wrangler-action）。

**GitHub CLI（gh）**
GitHubの操作をターミナルから完結させるCLIツール。リポジトリの作成やPRの操作がブラウザを開かずにできる。今回はリポジトリ作成に使った。

## 構築手順

### 1. fuwariのセットアップ

[fuwari](https://github.com/saicaca/fuwari) は[Astro](https://astro.build/)製のブログテーマ。Node.jsで動く。自分の環境ではNode.jsのバージョン管理に[Volta](https://volta.sh/)を使用した。

fuwariはNode.js 22、pnpm 9を指定しているので、それぞれインストールする。

```bash
volta install node@22
volta install pnpm@9
```

インストールできたら[create-fuwari](https://github.com/L4Ph/create-fuwari)を使って、ローカルにプロジェクトを初期化する。

```bash
pnpm create fuwari@latest
```

`src/config.ts` でサイト名・プロフィール・テーマカラーを設定する。

### 2. Cloudflare Workersの準備

Cloudflare Workersは、Cloudflareのネットワーク上でコードの実行や静的ファイルの配信ができるサービス。今回はその中の「Static Assets」という静的ファイルを配信する機能だけを使う。Astroがビルドした成果物（HTML/CSS/JS）を世界中に高速配信する置き場として利用するイメージ。

まず操作に使うCLIツール[wrangler](https://developers.cloudflare.com/workers/wrangler/)をインストールしてログインする。

```bash
npm install -g wrangler
wrangler login
```

リポジトリのルートに`wrangler.toml`を置き、配信するフォルダを指定する。

```toml
name = "fuwari"
compatibility_date = "2025-01-01"
workers_dev = true

[assets]
directory = "./dist"   # Astroのビルド出力先
```

この時点で一度、手元から手動デプロイして動作確認する。

```bash
pnpm run build   # dist/ を生成
wrangler deploy  # Workersにアップロード
```

成功すると `https://プロジェクト名.アカウント名.workers.dev` でアクセスできる。

### 3. GitHubにpush

ローカルで動作確認ができたら、GitHubにリポジトリを作成してソースコードをpushする。後でGitHub Actionsに自動ビルドさせるための土台になる。

リポジトリの作成には[GitHub CLI](https://cli.github.com/)（`gh`コマンド）を採用した。

```bash
gh repo create fuwari --public   # 公開リポジトリを作成
git add .
git commit -m "initial commit"
git push origin main
```

### 4. GitHub Actionsで自動デプロイ

ここがこの仕組みの核心。GitHub Actionsは、リポジトリへのpushなどをきっかけに、ビルドやデプロイといった作業を自動実行してくれる機能。

`main`ブランチにpushされたら「依存パッケージのインストール → Astroでビルド → Cloudflareへデプロイ」を自動で走らせるよう設定する。

リポジトリに`.github/workflows/deploy.yml`を作成し、以下を記述する。

```yaml
name: Build & Deploy

# mainへのpushで起動
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # リポジトリを取得
      - uses: actions/checkout@v4

      # pnpmを準備
      # versionは書かない（package.jsonのpackageManagerから自動取得。
      # 明記するとそちらと競合してエラーになる）
      - uses: pnpm/action-setup@v4

      # Node.jsを準備
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      # 依存パッケージをインストール
      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      # Astroでビルド（dist/を生成）
      - name: Build
        run: pnpm run build

      # wranglerでCloudflareへデプロイ
      - name: Deploy to Cloudflare Workers
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

最後の`secrets.〜`は、APIトークンなどの秘密情報をyamlに直書きせず、GitHubに安全に保管して呼び出す仕組み。リポジトリの **Settings → Secrets and variables → Actions** から登録する。

| Secret名 | 取得方法 |
|----------|---------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare → My Profile → API Tokens →「Edit Cloudflare Workers」テンプレートで作成 |
| `CLOUDFLARE_ACCOUNT_ID` | ターミナルで `wrangler whoami` を実行して確認 |

### 5. カスタムドメインの設定

`nonrem.net` はすでにCloudflareで管理しているドメインなので、Workersのダッシュボードから追加するだけでDNS設定まで自動でやってくれる。

**Workers & Pages → fuwari → Settings → Domains & Routes → Add Custom Domain**

ここに `nonrem.net` を入力して保存するだけ。Cloudflareが管理しているドメインであれば、CNAMEレコードの追加などは不要で、数分後には `https://nonrem.net` でアクセスできるようになる。

他社のDNSサービスを使っている場合は手動でCNAMEレコードを追加する必要があるので注意。

---

## ハマった点と解決策

### fuwariのビルドバグ（CSS処理順の問題）

`pnpm run build` を実行したら即エラーが出た。

```
The `link` class does not exist.
If `link` is a custom class, make sure it is defined within a `@layer` directive.
```

最初はCSSの書き方を間違えたのかと思ったけど、調べてみるとfuwari自体の問題だった。

fuwariのCSSはこういう構造になってる。

- `main.css` … `@layer components` の中で `.link` クラスを定義
- `markdown.css` … `@apply link` でそのクラスを参照

Tailwind v3はCSSファイルを**個別にPostCSS処理**する。ビルド時のファイル処理順序がランダムになることがあり、`markdown.css` が `main.css` より先に処理されると `.link` クラスがまだ定義されていないためエラーになる。毎回起きるわけではなく、ローカルの開発サーバーでは気づきにくいけど本番ビルドで顕在化しやすい不安定な挙動らしい。

同じ問題はfuwariのGitHubにも複数報告されており、[PR#738](https://github.com/saicaca/fuwari/pull/738) で修正パッチが提案されてた。

修正は `src/styles/markdown.css` の先頭に1行追加するだけ。

```css
@import "./main.css";   /* ← これを追加 */

.custom-md {
  /* ... */
}
```

`postcss-import` が `@import` を先に解決するので、必ず `main.css` → `markdown.css` の順で処理されるようになる。

### Node.jsとpnpmのバージョン固定

GitHub Actionsでビルドが走ったとき、環境によってはNode.jsのバージョンがローカルと違うことがある。バージョンを明示していないと「ローカルでは動くのにCI環境でだけ謎のエラーが出る」という事態になりやすい。

`.node-version` ファイルをリポジトリルートに置いておくと、GitHub ActionsやCloudflareがそれを読んでバージョンを合わせてくれる。

```
22
```

`package.json` の `engines` フィールドも合わせて設定しておくと確実。

```json
"engines": {
  "node": ">=22",
  "pnpm": ">=9"
}
```

### pnpmのバージョン二重指定に注意

GitHub Actionsで `pnpm/action-setup@v4` を使うとき、以下のように `version` を明記するとエラーになる。

```yaml
# NG
- uses: pnpm/action-setup@v4
  with:
    version: 9   # package.jsonのpackageManagerと競合してエラー
```

`package.json` の `packageManager` フィールドに `pnpm@9.14.4` と書いてあれば、`version` は省略でOK。自動で読んでくれる。

---

## まとめ

最終的にできあがった構成はこれ。

| 項目 | 内容 |
|------|------|
| フレームワーク | Astro（fuwariテーマ） |
| ホスティング | Cloudflare Workers Static Assets |
| CI/CD | GitHub Actions + wrangler-action |
| ドメイン | nonrem.net（Cloudflare管理） |

`main` ブランチにpushするだけでビルドからデプロイまで全自動で動く。記事を書いて `git push` すれば公開まで完結する。ドメイン代以外は完全に無料。

インフラエンジニアを目指してサーバーを自前管理しようとしていたのに、気づいたらフルサーバーレス構成になっていたのはちょっと面白い。まあ、目的はポートフォリオを公開することだったので良しとする。

### 今後やりたいこと

- OGP画像の自動生成
- RSSの整備
- 記事をちゃんと書く