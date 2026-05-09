---
name: publish-report
description: reports/ 配下の最新マーケットレポート markdown を読みやすい HTML に変換し、index.html (レポート一覧) と共に GitHub Pages の静的サイトを更新、コミット & push まで実行する。「レポート公開」「publish」「サイト更新」「公開して」などのリクエスト時に使用。GitHub Pages 公開先は https://ishii-taiki.github.io/gochisouba/。
---

# Publish Report Skill

`reports/` 内の最新 markdown レポートを HTML 化して GitHub Pages にデプロイする。`index.html` を更新し、コミット & プッシュまで一気通貫で行う。

## Pre-flight

1. CWD が gochisouba リポジトリのルート (`/Users/taikiishii/gochisouba`) であることを確認
2. `git status` で未コミットの想定外変更がないか確認。あればユーザーに告げて続行確認
3. `reports/` 配下から最新の `.md` ファイルを特定:
   ```bash
   ls -t reports/*.md 2>/dev/null | head -1
   ```
   無い場合 → 「`/market-report` でレポートを先に生成してください」と告げて終了
4. ターゲット: 引数で日付指定があればそれを優先、なければ最新を採用

## Step 1: HTML 生成

選定した markdown を HTML に変換し `reports/<basename>.html` として保存する。

要件:
- 単一ファイル完結 (外部 JS/CSS なし、CSS は `<style>` 埋め込み)
- レスポンシブ・読みやすいデザイン (max-width 760px、フォントは system-ui)
- 表 (`<table>`) を整形 (border, padding, hover)
- 絵文字 🟢🔴⚪ をそのまま保持
- リンクは `target="_blank" rel="noopener"`
- ページ上下に "← 一覧に戻る" ナビ (`../index.html`)
- `<title>`: "gochisouba — YYYY-MM-DD マーケットサマリー"
- `<meta charset="utf-8">`, `<meta name="viewport" content="width=device-width, initial-scale=1">`
- ライト/ダーク両対応 (`prefers-color-scheme`)

変換は Claude が markdown を読み取り、HTML を直接 Write で書き出す。
(pandoc 等の外部依存に依存しない。プロジェクトに node が無くても動く。)

## Step 2: index.html 更新

ルートの `index.html` を **完全に書き換える**。内容:

- ヘッダー: サイト名 "gochisouba" / 説明 "個人投資家向けマーケットサマリー"
- レポート一覧 (新しい順):
  - `reports/*.html` を `ls -t` で取得
  - 各エントリ: 日付 + ひとことサマリー (該当 .md の "ひとことで言うと" セクション直下の引用文を抽出) + リンク
- フッター: "投資助言ではありません" の注意書きと GitHub リポジトリへのリンク
- レポート HTML と同じ CSS デザインを共有

## Step 3: コミット & プッシュ

1. `git add reports/ index.html` (新規・変更ファイルのみ。`-A` は使わない)
2. `git status` で内容確認
3. コミット:
   ```bash
   git commit -m "$(cat <<'EOF'
   Publish report YYYY-MM-DD

   Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
   EOF
   )"
   ```
4. `git push origin main`
5. デプロイ URL を表示:
   - レポート: `https://ishii-taiki.github.io/gochisouba/reports/YYYY-MM-DD.html`
   - 一覧: `https://ishii-taiki.github.io/gochisouba/`
6. 「GitHub Pages のビルドに 1〜2 分かかります」と告知

## エラーハンドリング

| 状況 | 対応 |
|---|---|
| `reports/` が空 | `/market-report` を先に実行するよう案内 |
| 既に同日 .html が存在 & .md と同期済み | スキップして「変更なし」と表示 |
| `git push` 失敗 | エラー全文を表示、ユーザーに修正依頼 |
| GitHub Pages 未有効化 | `gh api -X POST repos/Ishii-Taiki/gochisouba/pages -f 'source[branch]=main' -f 'source[path]=/'` を提案 |

## Notes

- 1日に複数回公開する場合は `reports/YYYY-MM-DD-HHMM.md` のような命名を許容
- 機密情報やドラフトは公開対象外（公開リポジトリのため）
- HTML 変換時、markdown の table 内パイプ `|` のエスケープに注意
