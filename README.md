# My Blog

自分のブログ兼ポートフォリオサイトです。
自宅サーバーの構築記録や学習メモを中心に発信しています。

- **サイト**: 自己紹介と備忘録を兼ねた個人ブログ
- **フレームワーク**: [Astro](https://astro.build)（テンプレートに [Fuwari](https://github.com/saicaca/fuwari) を使用）
- **デプロイ**: `main` への push をトリガーに GitHub Actions がビルドし、Cloudflare Workers へ自動デプロイ

## 📝 記事の書き方

1. 新しい記事のひな形を作成する（`--` は付けない）:
   ```sh
   pnpm new-post <slug>
   ```
2. 生成された `src/content/posts/<slug>.md` を **フォルダ形式** に変える（画像を同梱するため）:
   ```sh
   mkdir src/content/posts/<slug>
   mv src/content/posts/<slug>.md src/content/posts/<slug>/index.md
   ```
3. 本文を書き、同じフォルダに画像を置いて `![alt](./image.png)` で参照する。
4. 完成したら frontmatter の `draft: false` にして commit & push（＝公開）。

### Frontmatter

```yaml
---
title: "記事タイトル"   # 「#」や「:」を含む場合は必ずダブルクォートで囲む（YAMLのコメント扱い回避）
published: 2026-07-02
description: "一覧・検索・SNSシェアで表示される説明文（120〜160文字が目安）"
image: ''
tags: ["AWS", "Docker"]
category: 技術
draft: false           # true にすると下書き（サイトに出ない）
lang: ''
---
```

## ⚙️ 設定ファイル

| 編集したいもの | ファイル |
|:---|:---|
| 名前・bio・アイコン・SNSリンク | `src/config.ts`（`profileConfig`） |
| サイトタイトル・テーマカラー等 | `src/config.ts`（`siteConfig`） |
| ナビゲーションメニュー | `src/config.ts`（`navBarConfig`） |
| Aboutページの本文 | `src/content/spec/about.md` |
| 記事 | `src/content/posts/<slug>/index.md` |

## ⚡ コマンド

プロジェクトルートで実行する:

| Command | Action |
|:---|:---|
| `pnpm install` | 依存関係をインストール |
| `pnpm dev` | 開発サーバーを起動（`localhost:4321`） |
| `pnpm build` | 本番ビルドを `./dist/` に生成（＋Pagefind検索インデックス） |
| `pnpm preview` | ビルド結果をローカルでプレビュー |
| `pnpm check` | エラーチェック |
| `pnpm format` | Biome でコード整形 |
| `pnpm new-post <slug>` | 新しい記事を作成 |

> `dist/` は `.gitignore` 済み。デプロイのたびに GitHub Actions 側でビルドし直されるので、push するのはソースだけでOK。

## 🙏 クレジット

このサイトは [Fuwari](https://github.com/saicaca/fuwari)（by saicaca / MIT License）をベースに構築しています。
