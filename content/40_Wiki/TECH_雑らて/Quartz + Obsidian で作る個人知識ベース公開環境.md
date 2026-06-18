---
publish: true
aliases:
  - 雑らて 環境紹介
  - Quartz環境
title: Quartz + Obsidian で作る個人知識ベース公開環境
created: 2026-06-18
modified: 2026-06-18T16:00:57.765+09:00
tags:
  - tech/quartz
  - tech/obsidian
  - tech/github-pages
---

# Quartz + Obsidian で作る個人知識ベース公開環境

**Obsidian** で書いたノートを、**Quartz v5** 経由で GitHub Pages に自動公開している。

> [!info] 公開サイト
> https://note.ratesystem.biz

この記事では、構成の全体像・PARA 管理との組み合わせ・デプロイの仕組みを紹介する。

---

## 全体構成

```
Obsidian Vault（PARA 構成）
    │
    │  Quartz Syncer プラグイン（公開ノートだけ選択）
    ▼
GitHub リポジトリ（v5 ブランチ）
    │
    │  GitHub Actions（push トリガー）
    │  ① PARA フォルダ平坦化
    │  ② カテゴリプレフィックス除去
    │  ③ Quartz ビルド
    ▼
GitHub Pages — 公開サイト
```

ポイントは **「Vault 全体を公開しない」** こと。\
Quartz Syncer で公開対象を選び、GitHub Actions でフォルダ構造を変換してからビルドする。

---

## PARA 構成 × Quartz Syncer

### Vault は PARA で整理

| フォルダ | 用途 |
|---|---|
| `10_Atlas/` | 進行中プロジェクト |
| `20_Resources/` | 分野別の参考資料・知識ノート |
| `30_Archive/` | 完了・休眠プロジェクト |
| `40_Wiki/` | 定義・手順・辞書的ノート |

サブフォルダ名には `TECH_`・`PRI_` などのカテゴリプレフィックスを付けて分類を明示している。

### Quartz Syncer で公開ノートを選択

> [!tip] なぜ Syncer を使うか
> Vault 全体を Git 管理すると、プライベートノートや未完成メモが意図せず公開される。\
> Quartz Syncer はノート単位で公開フラグを管理し、`content/` に同期するファイルを制御する。

- `draft: true` のノートは自動除外
- `98_Clipper/` など個人フォルダは同期対象から外す
- 公開範囲を Obsidian 側だけで完結して管理できる

---

## GitHub Actions によるデプロイ

`v5` ブランチへ push すると以下が自動実行される。

```yaml
# .github/workflows/deploy.yml（抜粋）
- name: Flatten top-level PARA folders        # ① PARA フォルダを平坦化
- name: Strip category prefix before underscore # ② プレフィックスを除去
- name: Build Quartz                           # ③ サイトビルド
- name: Deploy to GitHub Pages                 # ④ GitHub Pages へデプロイ
```

### ①②の変換処理が肝

Obsidian 側の PARA 構造をそのまま公開すると URL が煩雑になる。\
そこでデプロイ時にシェルで変換を挟んでいる。

| 処理 | 変換例 |
|---|---|
| PARA フォルダを平坦化 | `content/20_Resources/note.md` → `content/note.md` |
| プレフィックスを除去 | `TECH_雑らて/` → `雑らて/` |

Obsidian 側は PARA 構成を維持したまま、公開サイトはすっきりした URL になる。

#### 平坦化（rsync + rm）

```bash
for dir in content/*/; do
  rsync -a "$dir" content/
  rm -rf "$dir"
done
```

#### プレフィックス除去（find + mv）

```bash
find content/ -depth -name '*_*' | while IFS= read -r path; do
  dir=$(dirname "$path")
  base=$(basename "$path")
  newbase="${base#*_}"
  [ "$base" != "$newbase" ] && [ ! -e "$dir/$newbase" ] && mv "$dir/$base" "$dir/$newbase"
done
```

---

## 日々の運用フロー

```
1. Obsidian でノートを書く
2. Quartz Syncer で公開ノートを選択・同期
3. ローカルプレビューで確認  →  npx quartz build --serve
4. git push  →  GitHub Actions が自動ビルド＆デプロイ
5. 数分後に公開サイトへ反映
```

> [!warning] 公開前チェック
> `content/` に個人情報・業務情報が混入していないか、必ず目視確認すること。

---

## 技術スタック

| 項目 | 内容 |
|---|---|
| ノートエディタ | Obsidian |
| 知識管理手法 | PARA メソッド |
| 公開選択プラグイン | Quartz Syncer |
| 静的サイトジェネレータ | Quartz v5 |
| ホスティング | GitHub Pages |
| CI/CD | GitHub Actions |
| デプロイブランチ | `v5` |

---

## まとめ

| 課題 | 解決策 |
|---|---|
| Vault 全体を公開したくない | Quartz Syncer で公開ノートを選択 |
| PARA の数字プレフィックスが URL に出る | Actions のシェルで平坦化＋除去 |
| 毎回手動でビルドしたくない | git push だけで自動デプロイ |

Obsidian の使い心地を変えずに、公開サイトとしてきれいな形を保てるのがこの構成の強みだ。
