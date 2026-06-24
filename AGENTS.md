# AGENTS.md

このリポジトリは [Astro](https://astro.build/) + [fuwari](https://github.com/saicaca/fuwari) で構築した個人ブログ（`nonrem.net`）。
記事の作成と画像管理について、以下のルールに必ず従うこと。

## 技術スタック

- フレームワーク: Astro（fuwariテーマ）
- パッケージマネージャ: pnpm 9（Node.js 22）
- バージョン管理: Volta
- ホスティング: Cloudflare Workers Static Assets
- CI/CD: GitHub Actions（`main`へのpushで自動ビルド・デプロイ）

記事は `src/content/posts/` 配下に置く。`main` にpushすると自動で公開される。

## 新規記事の作成手順

記事を新しく作るときは、必ず以下の順で行う。

1. 雛形を生成する。スラッグは英語のkebab-caseで指定する。
   ```bash
   pnpm new-post <english-slug>
   ```
   例: `pnpm new-post deploy-astro-cloudflare`
   これでフロントマター（`published` は当日の日付が自動入力される）入りの `.md` が `src/content/posts/` に生成される。

2. 生成された `<english-slug>.md` を **フォルダ形式に変換する**。
   ```
   src/content/posts/<english-slug>.md
   ↓
   src/content/posts/<english-slug>/index.md
   ```
   つまり同名フォルダを作り、その中に `index.md` として移動する。

3. 画像はこのフォルダ内に配置する（後述のルール参照）。

## ディレクトリ構成

記事は必ず「1記事 = 1フォルダ」とし、本文と画像を共置する。

```
src/content/posts/
└── deploy-astro-cloudflare/
    ├── index.md          # 記事本体
    ├── cover.png         # カバー画像（フロントマターのimage）
    ├── 01-actions-yaml.png
    └── 02-secrets.png
```

理由: 画像を `src/` 配下に置くとAstroが最適化・バンドルしてくれる。`public/` に置くと最適化されず生のまま配信される。また記事ごとにフォルダを分けることで、記事削除時にフォルダごと消せて画像の孤児が出ない。

例外: 複数記事で使い回す共通画像や、OGP・favicon など直リンクで安定URLが欲しいものだけ `public/images/` に置く。それ以外は記事フォルダに共置すること。

## 命名規則

| 対象 | ルール | 例 |
|------|--------|-----|
| 記事フォルダ（=スラッグ=URL） | 英語のkebab-case。内容を表す語にする | `deploy-astro-cloudflare` |
| 記事本体 | 必ず `index.md` | `index.md` |
| カバー画像 | `cover.png`（または `cover.jpg`） | `cover.png` |
| 本文画像 | `連番-内容` のkebab-case | `01-actions-yaml.png` |

- 画像の連番は記事に登場する順に振る（ファイラーで並べたとき記事の流れと一致する）。
- フォルダ名・ファイル名に**日付を入れない**。日付はフロントマターの `published` が唯一のソース。二重管理とURL汚染を避ける。

## フロントマター

`pnpm new-post` が生成する雛形を基本とする。画像参照は相対パスで書く。

```yaml
---
title: 記事タイトル
published: 2026-06-24   # new-postが当日の日付を自動入力
description: "記事の説明"
image: ./cover.png      # 同フォルダのカバー画像を相対パスで指定
tags: ["タグ1", "タグ2"]
category: カテゴリ名
draft: false            # 公開前はtrueにしておく
---
```

本文中の画像も相対パスで参照する。

```markdown
![Secretsの設定画面](./02-secrets.png)
```

## やってはいけないこと

- 記事を `.md` 単体のまま放置しない（必ずフォルダ形式に変換する）。
- 画像を `public/` に安易に置かない（共通画像・favicon・OGPを除く）。
- フォルダ名・ファイル名に日付を入れない。
- スラッグ（フォルダ名）に日本語を使わない。

## 公開フロー

記事が完成したら以下で公開する。

```bash
git add .
git commit -m "post: <english-slug>"
git push origin main
```

push後、GitHub Actionsが自動でビルドしてCloudflare Workersにデプロイする。
