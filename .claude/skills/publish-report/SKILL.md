---
name: publish-report
description: reports/ 配下の最新マーケットレポート markdown を、モバイル最適化されたリッチな HTML に変換し、index.html (レポート一覧) と共に GitHub Pages の静的サイトを更新、コミット & push まで実行する。「レポート公開」「publish」「サイト更新」「公開して」などのリクエスト時に使用。GitHub Pages 公開先は https://ishii-taiki.github.io/gochisouba/。
---

# Publish Report Skill

`reports/` 内の最新 markdown レポートを **モバイルファーストのリッチな HTML** に変換し、GitHub Pages にデプロイする。`index.html` を更新し、コミット & プッシュまで一気通貫で行う。

## Design System

**デザインの正本（canonical）は `/Users/taikiishii/gochisouba/prototypes/mobile.html`**。同ファイルの構造・CSS・コンポーネントを参照・踏襲すること。新しい要素を追加する際もデザイントークン（変数）と既存コンポーネントとの一貫性を保つ。

### デザイン原則
- **モバイルファースト**: 主要表示は iPhone 想定。最小幅 320px で破綻しない
- **横スクロールテーブルを使わない**: データはカードグリッド/縦リスト/タイムラインで表現
- **下部固定ナビ**: 主要セクションへのワンタップ移動（safe-area-inset 対応）
- **ライト/ダーク両対応**: `prefers-color-scheme` で自動切替
- **タッチ最適化**: タップ領域広め、`-webkit-tap-highlight-color: transparent`
- **絵文字でアフォーダンス**: 国旗・指標アイコンは絵文字で軽量化
- **触れる操作要素**: 業種タブ（米/日切替）と Topic アコーディオン（`<details>`）は最小限の JS で実装

### コンポーネントと markdown セクションの対応

| markdown セクション | HTML コンポーネント |
|---|---|
| `# 📊 マーケットサマリー...` | `<header class="top-bar">` + ヒーロー（`.hero`、グラデーション、ピル式バッジ） |
| `## 1. 値動きサマリー` `### 主要7資産` | `.asset-grid` (2列カード、国旗絵文字、騰落ストライプ＋バッジ) |
| `### 相場の温度感（ボラティリティと金利）` | `.temp-grid` (2列カード、横バーゲージ＋マーカー) |
| `## 2. セクター別の動き` | `.tabs`（米/日切替）+ `.sector-list`（進捗バー付き） |
| `## 3. 注目銘柄ピックアップ` | `.stock-scroll` (横スクロールカルーセル、scroll-snap) |
| `## 4. 関連する情報` Topic | `<details class="topic">` アコーディオン（重要度バッジ：高/中/低で色分け） |
| `## 5. 来週の予定` | タイムライン形式 (`.cal-item` × n、日付ブロック + タグ色分け：米/日/決算/地政学) |
| `## 6. AIの売り買い判断` | `.stance-hero`（グラデーション文字＋確信度バッジ）+ `.stance-grid`（2列） |
| `## 7. 掲示板の民` | `.board` × 2（ブル/ベア）、左右ピン式ゲージ、注目ワードはチップ |
| `## 8. 今日の結論` | `.concl-hero`（強調枠）+ `.concl-grid`（強気/弱気/様子見/次イベント の4カード、bull/bear クラスで色分け） |
| `### ⚠️ 但し書き` | `.disclaimer`（黄色背景） |
| `### Sources` | `<details class="sources">` 折りたたみ |

下部に **固定ナビ** `.bottom-nav`（市場/業種/銘柄/予定/民/結論 の6リンク）を必ず設置。

## Pre-flight

1. CWD が gochisouba リポジトリのルート (`/Users/taikiishii/gochisouba`) であることを確認
2. `git status` で未コミットの想定外変更がないか確認。あればユーザーに告げて続行確認
3. `reports/` 配下から最新の `.md` ファイルを特定:
   ```bash
   ls -t reports/*.md 2>/dev/null | head -1
   ```
   無い場合 → 「`/market-report` でレポートを先に生成してください」と告げて終了
4. ターゲット: 引数で日付指定があればそれを優先、なければ最新を採用

## Step 1: 共通 CSS の準備

`assets/style.css` を共通スタイルシートとして使う。

1. `/Users/taikiishii/gochisouba/assets/style.css` が存在しなければ作成する
2. 内容は **`prototypes/mobile.html` の `<style>...</style>` ブロックをそのまま** 抜き出して保存（`<style>` タグは含めない、CSS のみ）
3. プロトタイプから機能追加・デザイン変更があった場合は `assets/style.css` も同時に更新する
4. 既に存在し最新なら触らない

これにより各レポート HTML は CSS を共有でき、保守が一元化される。

## Step 2: レポート HTML 生成

選定した markdown を HTML に変換し `reports/<basename>.html` として保存する。

### 共通要件
- `<!doctype html>` `<html lang="ja">` から始める
- `<meta charset="utf-8">` `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">` `<meta name="theme-color" content="#0f172a">` を必ず含める
- `<title>`: "gochisouba — YYYY-MM-DD マーケットサマリー"
- 共通 CSS を `<link rel="stylesheet" href="../assets/style.css">` で読み込む
- リンクは `target="_blank" rel="noopener"` を付ける
- ページ末尾に **下部固定ナビ** (`.bottom-nav`) を必ず置く
- 業種タブ切替の最小 JS を `<script>` で埋め込む（プロトタイプ参照）

### セクション生成ルール
- markdown の各セクションを `Design System` 表に従って HTML コンポーネントへ変換
- markdown にデータが欠けているセクションは **DOM ごと省略**（空の枠を残さない）
- 数値の方向感（🟢🔴⚪）は `.up` / `.down` / `.flat` クラスにマッピング
- 重要度（高/中/低）は `.badge.high` / `.mid` / `.low` にマッピング
- カレンダーのタグは国/種別で `.cal-tag.us` / `.jp` / `.earnings` / `.geopolitics` に色分け
- 注目銘柄のテーマは `.theme-tag` に直接記載

### ヒーロー
- レポートの「ひとことで言うと」を `.hero h1` に。
- 「総合判断」「確信度」「主なリスク」を `.pill` バッジ 3 つに（絵文字付き）

### 下部ナビ
固定で下記リンクを置く:
```
📊 市場 → #market
🏭 業種 → #sector
⭐ 銘柄 → #picks
📅 予定 → #calendar
💬 民   → #bbs
🎯 結論 → #concl
```

各セクションには上記 ID を必ず付与する。

### 業種タブ切替 JS
ページ末尾に最小 JS を埋め込む（プロトタイプ参照）:
```html
<script>
  document.querySelectorAll('.tab').forEach(t => {
    t.addEventListener('click', () => {
      const target = t.dataset.tab;
      document.querySelectorAll('.tab').forEach(x => x.classList.toggle('active', x === t));
      document.querySelectorAll('.sector-pane').forEach(p => p.classList.toggle('active', p.dataset.pane === target));
    });
  });
</script>
```

## Step 3: index.html 更新

ルートの `index.html` を **完全に書き換える**。

### 構成
- 共通 CSS 読み込み: `<link rel="stylesheet" href="assets/style.css">`
- `<header class="top-bar">`: サイト名 "gochisouba" + 「個人投資家向けマーケットサマリー」
- ヒーロー（短め）: サイト説明1行 + GitHub リンクバッジ
- レポート一覧 (新しい順):
  - `ls -t reports/*.html` で取得
  - 各エントリは `.report-card`（カード型）として表示
  - 内容: 日付 (大) / ひとことサマリー (該当 .md の「ひとことで言うと」直下の引用を抽出) / 総合判断バッジ
  - タップで該当レポートへ遷移
- フッター: 投資助言ではない旨 + GitHub リポジトリへのリンク
- 下部固定ナビは index には不要（一覧画面なのでセクション間ジャンプ不要）

### .report-card のスタイルが style.css に未定義の場合
プロトタイプにない要素なので、index.html 専用に最小スタイルを `<style>` で追記して良い。デザイントークン（CSS変数 `--bg-elev`, `--border` 等）は使い回す。

## Step 4: コミット & プッシュ

1. `git add reports/ index.html assets/` (新規・変更ファイルのみ。`-A` は使わない)
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
| `prototypes/mobile.html` が無い | デザインの正本が欠けている旨をユーザーに告げ、一旦中止 |
| `assets/style.css` が古い（プロトタイプと不一致） | プロトタイプから再生成して上書き |
| 既に同日 .html が存在 & .md と同期済み | スキップして「変更なし」と表示 |
| `git push` 失敗 | エラー全文を表示、ユーザーに修正依頼 |
| GitHub Pages 未有効化 | `gh api -X POST repos/Ishii-Taiki/gochisouba/pages -f 'source[branch]=main' -f 'source[path]=/'` を提案 |

## Notes

- 1日に複数回公開する場合は `reports/YYYY-MM-DD-HHMM.md` のような命名を許容
- 機密情報やドラフトは公開対象外（公開リポジトリのため）
- markdown のテーブル内パイプ `|` を HTML 化する際は `&#124;` でエスケープ
- HTML 内の `<` `>` `&` は適切にエスケープ（`&lt;` `&gt;` `&amp;`）
- レポートが新セクション（VIX、業種、銘柄、カレンダー等）を欠いていてもページが破綻しないよう、各セクションは独立して省略可能にする
- デザイン変更を行う際は **必ず `prototypes/mobile.html` を先に更新** し、それを正本として `assets/style.css` および各レポート HTML を再生成する
