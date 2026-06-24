---
title: Fuwariでポートフォリオ用サイトを準備した
published: 2026-06-24
description: "fuwariテーマのAstroブログをGitHub Actions + Cloudflare Workersで自動デプロイするまでの全記録。躓いた点も含めて。"
tags: ["Node.js", "Astro", "Cloudflare", "GitHub"]
category: 技術
draft: false
---

## はじめに

こんにちは、ひょろです。6月中旬から転職活動をしています。
自前のウェブサイトを持つエンジニアは少なくないようなので、自分もそれに倣い準備してみることにしました。
とはいえ2026年6月時点では転職活動中でエンジニアとしては未経験なのであくまでも真似事にすぎません。

以前は静的なトップページとWordPressのブログを公開していましたが、今回のサイトは完全にサーバーレスになってしまいました。
インフラエンジニアを目指す身としては本末転倒な気がしますが、経緯から詳しい導入手順まで書いていきます。

## 経緯

最初期は自宅のサーバーにWordPress環境と簡単なHTMLのページを準備していましたが、管理面で手間がかかるのと脆弱性を管理しきれないという理由で他の方法を探していました。
そんな時にChatGPTに[Astro](https://astro.build/)というSSGを教えてもらいました。
簡単に説明すると、Webサイトのページを公開前にすべてHTMLファイルとして生成しておく技術らしい。

動的サイトの代表格であるWordPressはアクセス時にPHPが動作して、データベースから記事情報を取得してHTMLを生成します。
それに対してSSGは、アクセスが来る前に静的なHTMLサイトを事前に準備しておく。
アクセスしに来た人はすでに出来上がってるページを見て即終了って感じ。
これによってレスポンスの遅さや脆弱性を克服できるって仕組み。何でも知ってるAIってすごい。

最初はこのSSGすら自前のサーバーでビルドしてから公開していましたが、なんとGitHubとCloudflareの機能を使えば自前のサーバーを使わなくても実装できるらしい。しかも無料で。
というわけでGitHub Actionsという機能と、Cloudflare Workersという機能をふんだんに使っていきます。

そんな経緯で最終的に開発環境はサーバーのvimからMacに移りました。
gitでcloneできる環境ならWindowsとか他のPCでも良くなったのはすごい。

## 最終的な流れと構成

記事を書く → git push → GitHub Actionsで自動ビルド → Cloudflare Workersで自動デプロイ → nonrem.netで公開

任意の端末でMarkdown記事を執筆してGitHubにプッシュすれば、自動でサイトに反映される仕組みです。
ここまで手ぶらで自動化できるのにドメイン料以外が無料なのは破格でヤバい。

| 項目 | 技術 |
|------|------|
| ブログテーマ | fuwari（Astro製） |
| ホスティング | Cloudflare Workers Static Assets |
| CI/CD | GitHub Actions + wrangler-action |
| ドメイン | nonrem.net（Cloudflare管理） |

### 選定理由

**fuwari（Astro）**
WordPress（PHP）は動的にページを生成するぶん、サーバーの処理とセキュリティ管理が重くなります。対してAstroは事前にHTMLを吐き出すSSGなので、軽量で速く、攻撃される面も少なくなります。テーマの見た目が好みだったのも大きいです。

**Volta**
Node.jsのバージョン管理ツールです。Rust製で動作が速いのと、プロジェクトごとに使うバージョンを自動で切り替えてくれるのが楽です。正直この用途なら他のツール（nvm, fnm等）でも問題ないと思います。流行ってるので採用しました。

**GitHub**
言わずもがな。今回は特にソース管理より、GitHub Actions（自動ビルド・デプロイ）の働きが主役です。

**Cloudflare Workers**
当初はCloudflare Pagesを使う予定でしたが、Cloudflareが新規プロジェクトの推奨をPagesからWorkersに移しています。今から作るならWorkers一択です。

**wrangler**
CloudflareのCLIツールです。Workers/Pagesへのデプロイやログの確認などができます。GitHub Actionsからのデプロイにも内部的に使われています（wrangler-action）。

**GitHub CLI（gh）**
GitHubの操作をターミナルから完結させるCLIツールです。リポジトリの作成やPRの操作がブラウザを開かずにできます。今回はリポジトリ作成に使いました。

## 構築手順

### 1. fuwariのセットアップ

[fuwari](https://github.com/saicaca/fuwari) は[Astro](https://astro.build/)製のブログテーマで、Node.jsで動きます。自分の環境では、Node.jsのバージョン管理に[Volta](https://volta.sh/)を使用しました。

fuwariはNode.js 22、pnpm 9を指定しているので、それぞれインストールします。

```bash
volta install node@22
volta install pnpm@9
```

インストールできたら[create-fuwari](https://github.com/L4Ph/create-fuwari)を使って、ローカルにプロジェクトを初期化します。

```bash
pnpm create fuwari@latest
```

`src/config.ts` でサイト名・プロフィール・テーマカラーを設定します。

### 2. Cloudflare Workersの準備

Cloudflare Workersは、Cloudflareのネットワーク上でコードの実行や静的ファイルの配信ができるサービスです。今回はその中の「Static Assets」という静的ファイルを配信する機能だけを使います。Astroがビルドした成果物（HTML/CSS/JS）を世界中に高速配信する置き場として利用するイメージです。

まず、操作に使うCLIツール[wrangler](https://developers.cloudflare.com/workers/wrangler/)を導入します。グローバルに入れると環境ごとにバージョンがブレるので、プロジェクトのdevDependenciesに入れて`package.json`でバージョンを固定します。これでcloneした別の端末でも同じバージョンが使えます。

```bash
pnpm add -D wrangler
```

導入できたらログインします。`pnpm wrangler`でプロジェクトのwranglerを実行できます。

```bash
pnpm wrangler login
```

リポジトリのルートに`wrangler.toml`を置き、配信するフォルダを指定します。

```toml
name = "fuwari"
compatibility_date = "2025-01-01"
workers_dev = true

[assets]
directory = "./dist"   # Astroのビルド出力先
```

この時点で一度、手元から手動デプロイして動作確認します。

```bash
pnpm run build     # dist/ を生成
pnpm wrangler deploy  # Workersにアップロード
```

成功すると `https://プロジェクト名.アカウント名.workers.dev` でアクセスできます。

### 3. GitHubにpush

ローカルで動作確認ができたら、GitHubにリポジトリを作成してソースコードをpushします。後でGitHub Actionsに自動ビルドさせるための土台になります。

リポジトリの作成には[GitHub CLI](https://cli.github.com/)（`gh`コマンド）を採用しました。

```bash
gh repo create fuwari --public   # 公開リポジトリを作成
git add .
git commit -m "initial commit"
git push origin main
```

### 4. GitHub Actionsで自動デプロイ

ここがこの仕組みの核心です。GitHub Actionsは、リポジトリへのpushなどをきっかけに、ビルドやデプロイといった作業を自動実行してくれる機能です。

`main`ブランチにpushされたら「依存パッケージのインストール → Astroでビルド → Cloudflareへデプロイ」を自動で走らせるよう設定します。

リポジトリに`.github/workflows/deploy.yml`を作成し、以下を記述します。

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
      - uses: pnpm/action-setup@v4

      # Node.jsを準備（.node-versionファイルからバージョンを読む）
      - uses: actions/setup-node@v4
        with:
          node-version-file: .node-version
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

最後の`secrets.〜`は、APIトークンなどの秘密情報をyamlに直書きせず、GitHubに安全に保管して呼び出す仕組みです。リポジトリの **Settings → Secrets and variables → Actions** から登録します。

| Secret名 | 取得方法 |
|----------|---------|
| `CLOUDFLARE_API_TOKEN` | Cloudflare → My Profile → API Tokens →「Edit Cloudflare Workers」テンプレートで作成 |
| `CLOUDFLARE_ACCOUNT_ID` | ターミナルで `pnpm wrangler whoami` を実行して確認 |

### 5. カスタムドメインの設定

`nonrem.net` はすでにCloudflareで管理しているドメインなので、Workersのダッシュボードから追加するだけでDNS設定まで自動でやってくれます。

**Workers & Pages → fuwari → Settings → Domains & Routes → Add Custom Domain**

ここに `nonrem.net` を入力して保存するだけです。Cloudflareが管理しているドメインであれば、CNAMEレコードの追加などは不要で、数分後には `https://nonrem.net` でアクセスできるようになります。

他社のDNSサービスを使っている場合は、手動でCNAMEレコードを追加する必要があるので注意してください。

---

## ハマった点と解決策

### fuwariのビルドバグ（CSS処理順の問題）

`pnpm run build` を実行したら即エラーが出ました。

```
The `link` class does not exist.
If `link` is a custom class, make sure it is defined within a `@layer` directive.
```

最初はCSSの書き方を間違えたのかと思いましたが、調べてみるとfuwari自体の問題でした。

fuwariのCSSはこういう構造になっています。

- `main.css` … `@layer components` の中で `.link` クラスを定義
- `markdown.css` … `@apply link` でそのクラスを参照

Tailwind v3はCSSファイルを**個別にPostCSS処理**します。ビルド時のファイル処理順序がランダムになることがあり、`markdown.css` が `main.css` より先に処理されると `.link` クラスがまだ定義されていないためエラーになります。毎回起きるわけではなく、ローカルの開発サーバーでは気づきにくいですが、本番ビルドで顕在化しやすい不安定な挙動らしいです。

同じ問題はfuwariのGitHubにも複数報告されており、[PR#738](https://github.com/saicaca/fuwari/pull/738) で修正パッチが提案されていました。

修正は `src/styles/markdown.css` の先頭に1行追加するだけです。

```css
@import "./main.css";   /* ← これを追加 */

.custom-md {
  /* ... */
}
```

`postcss-import` が `@import` を先に解決するので、必ず `main.css` → `markdown.css` の順で処理されるようになります。

### Node.jsとpnpmのバージョン固定

GitHub Actionsでビルドが走ったとき、環境によってはNode.jsのバージョンがローカルと違うことがあります。バージョンを明示していないと「ローカルでは動くのにCI環境でだけ謎のエラーが出る」という事態になりやすいです。

そこで `.node-version` ファイルをリポジトリルートに置き、使うNode.jsのバージョンを1ファイルで管理します。

```
22
```

前述のワークフローでは `actions/setup-node` の `node-version-file` でこのファイルを参照させているので、バージョンを変えたいときはこの1ファイルを直すだけで済みます。

`package.json` の `engines` フィールドも合わせて設定しておくと、意図しないバージョンでのインストールを弾けて確実です。

```json
"engines": {
  "node": ">=22",
  "pnpm": ">=9"
}
```

### pnpmのバージョン二重指定に注意

GitHub Actionsで `pnpm/action-setup@v4` を使うとき、以下のように `version` を明記するとエラーになります。

```yaml
# NG
- uses: pnpm/action-setup@v4
  with:
    version: 9   # package.jsonのpackageManagerと競合してエラー
```

`package.json` の `packageManager` フィールドに `pnpm@9.14.4` のように書いてあれば、`version` は省略でOKです。action-setupがそこから自動で読んでくれます。

---

## まとめ

最終的にできあがった構成はこれです。

| 項目 | 内容 |
|------|------|
| フレームワーク | Astro（fuwariテーマ） |
| ホスティング | Cloudflare Workers Static Assets |
| CI/CD | GitHub Actions + wrangler-action |
| ドメイン | nonrem.net（Cloudflare管理） |

`main` ブランチにpushするだけで、ビルドからデプロイまで全自動で動きます。記事を書いて `git push` すれば公開まで完結します。ドメイン代以外は完全に無料です。

インフラエンジニアを目指してサーバーを自前管理しようとしていたのに、気づいたらフルサーバーレス構成になっていたのはちょっと面白いです。まあ、目的はポートフォリオを公開することだったので良しとします。

### 今後やりたいこと

- OGP画像の自動生成
- 記事をちゃんと書く
