---
name: publish-report
description: reports/ 配下の最新マーケットレポート markdown を「投資判断ダッシュボード」として HTML 化し、index.html を最新レポートで上書きして GitHub Pages を更新、コミット & push まで実行する。「レポート公開」「publish」「サイト更新」「公開して」などのリクエスト時に使用。GitHub Pages 公開先は https://ishii-taiki.github.io/gochisouba/。
---

# Publish Report Skill

`reports/` 内の最新 markdown レポートを **モバイルファーストの投資判断ダッシュボード HTML** に変換し、GitHub Pages にデプロイする。サイトはトップページ (`index.html`) が常に最新レポートで上書きされる単一ページ運用。一覧ページは作らない。コミット & プッシュまで一気通貫で行う。

## Design System

**デザインの正本（canonical）は `/Users/taikiishii/gochisouba/prototypes/mobile.html`**。同ファイルの構造・CSS・コンポーネントを参照・踏襲する。新しい要素を追加する場合もデザイントークン（CSS 変数）と既存コンポーネントとの一貫性を保つこと。

### デザイン原則
- **投資判断ダッシュボード**: 単なる情報一覧ではなく「読者の投資行動が決まる」設計。各セクションは「情報 → 相場への影響 → 自分の行動 → 判断が変わる条件」の流れに変換する
- **モバイルファースト**: 主要表示は iPhone 想定。最小幅 320px で破綻しない
- **横スクロールテーブルを使わない**: データはカードグリッド／2列セル／タイムラインで表現
- **下部固定ナビ**: 主要セクションへのワンタップ移動（safe-area-inset 対応）
- **ライト/ダーク両対応**: `prefers-color-scheme` で自動切替
- **タッチ最適化**: タップ領域広め、`-webkit-tap-highlight-color: transparent`
- **情報量より行動量**: 重要度の低い網羅情報は `<details>` で折りたたむ

### コンポーネントと markdown セクションの対応

| markdown セクション | HTML コンポーネント |
|---|---|
| `# 📊 ご馳走場 — YYYY-MM-DD` | `<header class="top-bar">`（ブランド「ご馳走場」＋日付） |
| `## 1. 今日の投資判断` | `.verdict-hero`（グラデーション枠）<br>＋ `.pill-row`（スタンス／確信度／ポジション量）<br>＋ `.verdict-grid`（推奨行動／新規買い／利確方針／損切り条件）<br>＋ `.do-dont`（やること／やらないこと）<br>＋ `.next-trigger`（次に判断が変わるイベント） |
| `## 2. 今日は控えたいこと` | `.forbidden`（赤帯カード、🚫 付きリスト） |
| `## 3. 脳汁チャンス` | `.chance-banner`（橙赤グラデの注意帯）＋ `.chance-list` × N（`.chance` 各カード：脳汁シナリオ/地獄シナリオ の2セル＋メモ） |
| `## 4. こうなったら、こう判断する` | `.scenarios` ＞ `.scenario.bull` / `.wait` / `.bear` の **3カードのみ**（条件／起きやすい相場／自分の判断／確認指標） |
| `## 5. マーケット温度計` | `.therm-list` の最上段に `.therm-sent`（掲示板の民の動向：4.3倍ブル ＋ 3.8倍ベアIII の2カード ＋ `.ts-summary` 2掲示板の総括）。続いて `.therm` × 4（米VIX／日経VI／米10年債利回り／日本10年債利回り） |
| `## 6. 今週のイベント` | `.events` ＞ `.event-card` × N（日付バッジ＋イベント名＋重要度バッジ／`.branch-grid` 上振れ↑下振れ↓／`.pre-post` イベント前・後アクション） |
| `## 7. 資産別アクション` | `.asset-cards` ＞ `.asset-card` × 7（資産名＋判断バッジ／`.action-now` 今日の行動帯／`.meta` 買い条件・売り条件 の2セル／`.reason`） |
| `## 8. ニュース` | `.news-list` ＞ `.news` × N（重要度バッジ＋見出し／影響資産・相場への影響・自分の行動・変更条件） |
| `## 9. 詳細データ・ソース` | `<details class="archive">` × N（価格詳細・経済指標一覧 等）＋ `<details class="sources">`（情報源リンク） |
| `## 今日の結論（3行）` | `.concl-3`（グラデーション枠＋3項目の番号付きリスト） |

> 投資助言ではない旨の `.disclaimer` ブロックは描画しない。markdown 側に残っていても HTML には出さない（本サイトは作者個人の利用が主目的のため）。

### 下部固定ナビ
固定で **6リンク** を置く（順序固定）：

```
🎯 判断 → #verdict
🔥 脳汁 → #chance
🧭 見方 → #triggers
🌡️ 温度 → #therm
📅 予定 → #events
💼 資産 → #assets
```

各セクションには上記 ID（`verdict` / `caution` / `chance` / `triggers` / `therm` / `events` / `assets` / `news` / `archive`）を必ず付与する。`caution` と `news` はナビからは省略するが ID は付ける。

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

## Step 2: レポート HTML 生成

選定した markdown を HTML に変換し `reports/<basename>.html` として保存する。

### 共通要件
- `<!doctype html>` `<html lang="ja">` から始める
- `<meta charset="utf-8">` `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">` `<meta name="theme-color" content="#0f172a">` を必ず含める
- `<title>`: 「ご馳走場 — YYYY-MM-DD」（サブタイトルなし）
- ヘッダーの `.brand` は **「ご馳走場」**（英字 `gochisouba` は使わない）
- 共通 CSS を `<link rel="stylesheet" href="../assets/style.css">` で読み込む
- 外部リンクは `target="_blank" rel="noopener"`
- ページ末尾に `.bottom-nav`（6リンク）を必ず置く
- JavaScript は不要。スムーススクロールは CSS の `html { scroll-behavior: smooth; }` のみで賄う

### セクション生成ルール
- markdown の各セクションを上記 Design System 表に従って HTML コンポーネントへ変換
- markdown にデータが欠けているセクションは **DOM ごと省略**（空の枠を残さない）
- 判定バッジ（`.level`）の色分け:
  - `.level.calm`（平静、緑）／`.level.warn`（警戒、橙）／`.level.high`（高水準、赤）／`.level.over`（過熱、紫）／`.level.off`（リスクオフ、濃赤）
- シナリオの色分け（`.scenario`）: `.bull`（緑）／`.wait`（橙）／`.bear`（赤）
- 重要度バッジ（`.imp` / `.event-card .imp`）: `.high`（赤）／`.mid`（橙）／`.low`（灰）
- 資産カードの判断バッジ（`.verdict`）: `.bull` / `.bear` / `.warn`（中立）
- 騰落の方向感は `.up` / `.down` / `.flat` クラス
- 数値はカード内に「現在値（前日比）」の形式で短く表示。表は使わない

### ヒーロー（`.verdict-hero`）の組み立て
- `<h1>` には markdown 「ヘッドコピー」（一文）を入れる
- `.pill-row`: 総合スタンス（`.pill.solid`）＋ 確信度 ＋ ポジション量目安
- `.verdict-grid`: 推奨行動 / 新規買い / 利確方針 / 損切り条件 の 4 セル
- `.do-dont`: 「✅ 今日やること」「⛔ 今日やらないこと」の 2 ボックス
- `.next-trigger`: 「⏰ 次に判断が変わるイベント」（日時 + イベント名 + 1行説明）

### 「今日の結論（3行）」（`.concl-3`）
- ページ最後（`## 9. 詳細データ・ソース` の **直前**）に置く
- 番号付きリストで 3 項目厳守

## Step 3: index.html 更新（最新レポートで上書き）

`index.html` は **最新レポートそのもの** をトップページとして提供する。レポート一覧ページは作らない（毎日上書きされる単一ページ運用）。

### 手順
1. Step 2 で生成した `reports/<basename>.html` を **そのままの内容** で `index.html` にコピー（上書き）
2. ルート直下 (`index.html`) からの CSS パスのみ書き換える:
   - レポート版: `<link rel="stylesheet" href="../assets/style.css">`
   - index 版: `<link rel="stylesheet" href="assets/style.css">`
3. それ以外（タイトル・本文・ナビ）はレポート版と完全一致

### 注意
- 過去レポートは `reports/` 配下にファイルとして残るが、トップからのリンクは張らない
- `index.html` も `<title>` は「ご馳走場 — YYYY-MM-DD」で統一

## Step 4: コミット & プッシュ

1. `git add reports/ index.html assets/` （`-A` は使わない）
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
   - トップ（最新レポート）: `https://ishii-taiki.github.io/gochisouba/`
   - 同一内容のアーカイブ: `https://ishii-taiki.github.io/gochisouba/reports/YYYY-MM-DD.html`
6. 「GitHub Pages のビルドに 1〜2 分かかります」と告知

## エラーハンドリング

| 状況 | 対応 |
|---|---|
| `reports/` が空 | `/market-report` を先に実行するよう案内 |
| `prototypes/mobile.html` が無い | デザインの正本が欠けている旨を告げ中止 |
| `assets/style.css` が古い（プロトタイプと不一致） | プロトタイプから再生成して上書き |
| 既に同日 .html が存在 & .md と同期済み | スキップして「変更なし」と表示 |
| `git push` 失敗 | エラー全文を表示、ユーザーに修正依頼 |
| GitHub Pages 未有効化 | `gh api -X POST repos/Ishii-Taiki/gochisouba/pages -f 'source[branch]=main' -f 'source[path]=/'` を提案 |

## Notes

- 1日に複数回公開する場合は `reports/YYYY-MM-DD-HHMM.md` のような命名を許容
- 機密情報やドラフトは公開対象外（公開リポジトリのため）
- HTML 内の `<` `>` `&` は適切にエスケープ（`&lt;` `&gt;` `&amp;`）
- レポートが一部セクションを欠いていてもページが破綻しないよう、各セクションは独立して省略可能にする
- デザイン変更を行う際は **必ず `prototypes/mobile.html` を先に更新** し、それを正本として `assets/style.css` および各レポート HTML を再生成する
