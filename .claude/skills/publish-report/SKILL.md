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

レポートは **6セクション固定**。各セクションは独立して省略可能だが、原則すべて存在する想定。

| markdown セクション | section id | HTML コンポーネント |
|---|---|---|
| `# 📊 ご馳走場 — YYYY-MM-DD` | — | `<header class="top-bar">`（ブランド「ご馳走場」＋日付） |
| `## 1. 脳汁チャンス` | `chance` | `.chance-banner`（橙赤グラデの注意帯）＋ `.chance-list` × N（`.chance` 各カード：脳汁シナリオ/地獄シナリオ の2セル＋メモ） |
| `## 2. 高ボラティリティ重要決算スケジュール` | `earnings` | `.events` ＞ `.event-card` × N（日付バッジ＋Ticker/企業名＋`.move-tag` 想定変動率＋`.imp` 重要度バッジ／`.branch-grid` 予想超え▲予想未達▼／`.pre-post` 決算前・決算後アクション） |
| `## 3. 今週の重要イベント（決算を除く）` | `events` | `.events` ＞ `.event-card` × N（日付バッジ＋イベント名＋重要度バッジ／`.branch-grid` 上振れ▲下振れ▼／`.pre-post` イベント前・後アクション）。`.move-tag` は付けない |
| `## 4. 10バガー候補銘柄` | `tenbagger` | `.theme-intro`（注記1行）＋ `.watch` ＞ `.stock` × N（米国上場株の長期調査候補。区分バッジ／強気・弱気シナリオ／監視KPI／スコア） |
| `## 5. 量子コンピューティング関連ニュースおよび銘柄` | `quantum` | `.block-label`「📰 関連ニュース」＋ `.news-list` ＞ `.news` × N、続いて `.block-label`「📈 関連銘柄」＋ `.watch` ＞ `.stock` × N |
| `## 6. 宇宙関連ニュースおよび銘柄` | `space` | `.block-label`「📰 関連ニュース」＋ `.news-list` ＞ `.news` × N、続いて `.block-label`「📈 関連銘柄」＋ `.watch` ＞ `.stock` × N |

> 投資助言ではない旨の `.disclaimer` ブロックは描画しない。markdown 側に残っていても HTML には出さない（本サイトは作者個人の利用が主目的のため）。
> 旧構成（`verdict` / `caution` / `triggers` / `therm` / `assets` / `news` / `concl` / `archive`）のセクションは **生成しない**。

### 下部固定ナビ
固定で **6リンク** を置く（順序固定。6セクションと1対1対応）：

```
🔥 脳汁 → #chance
📊 決算 → #earnings
📅 予定 → #events
🚀 10倍 → #tenbagger
⚛️ 量子 → #quantum
🛰️ 宇宙 → #space
```

各セクションには上記 ID（`chance` / `earnings` / `events` / `tenbagger` / `quantum` / `space`）を必ず付与する。

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
- 共通 CSS を `<link rel="stylesheet" href="../assets/style.css?v=YYYY-MM-DD">` で読み込む（`?v=日付` はブラウザキャッシュ回避のため必須。レポート日付に置き換える）
- 外部リンクは `target="_blank" rel="noopener"`
- ページ末尾に `.bottom-nav`（6リンク）を必ず置く
- JavaScript は不要。スムーススクロールは CSS の `html { scroll-behavior: smooth; }` のみで賄う

### セクション生成ルール
- markdown の各セクションを上記 Design System 表に従って HTML コンポーネントへ変換
- セクションは **1〜6 の番号・並び順固定**。`.sec-h .num` は 1〜6 を振る
- markdown にデータが欠けているセクションは **DOM ごと省略**（空の枠を残さない）
- 重要度バッジ（`.imp` / `.event-card .imp` / `.news .imp`）: `.high`（赤）／`.mid`（橙）／`.low`（灰）
- 想定変動率（決算のみ）: `.event-card .move-tag`（紫ピル、例「想定 ±8〜12%」）。「3. 今週の重要イベント」では使わない
- 騰落・上下の方向感は `.up` / `.down` / `.flat` クラス（`.chance .ch-scen .s.up/.down`、`.branch.upside/.downside`）
- 数値はカード内に短く表示。横スクロール表は使わない

### 「2. 高ボラティリティ重要決算スケジュール」（`.events` / `.event-card`）
- `## 2.` がある場合のみ描画。`<section id="earnings">`、`.sec-h .num` は `2`
- `.ehead`: `.date`（日付）＋ `.ename`（Ticker / 企業名）＋ `.move-tag`（想定変動率、あれば）＋ `.imp`（重要度）
- `.branch-grid`: 左 `.upside`「▲ 予想超え」、右 `.downside`「▼ 予想未達」
- `.pre-post`: 左「決算前」、右「決算後」

### 「3. 今週の重要イベント（決算を除く）」（`.events` / `.event-card`）
- `<section id="events">`、`.sec-h .num` は `3`。`.move-tag` は付けない
- `.branch-grid`: 左 `.upside`「▲ 上振れ」、右 `.downside`「▼ 下振れ」
- `.pre-post`: 左「イベント前」、右「イベント後」

### 「4. 10バガー候補銘柄」（`.watch` / `.stock`）
- `## 4.` がある場合のみ描画し、無い場合は DOM ごと省略する
- `<section id="tenbagger">`、`.sec-h .num` は `4`、見出しは「10バガー候補銘柄」、サブコピーは「米国株の長期調査候補」
- 冒頭に `.theme-intro` で短い注記を 1 行: 「買い推奨ではなく、5〜10年の事業成長を深掘りする調査候補。」
- 各候補は `.stock` カードに変換する:
  - `.top`: `.nm` に `Ticker — Company name`、`.role` に `High-conviction compounder` / `Speculative high-upside` / `Turnaround growth` / `Watchlist only` / `Avoid`
  - 本文上部（`.ch-note` を流用）に Sector / Industry、Market cap、Action classification、Confidence score を短く表示
  - `.signals`: 左 `.sig.bull`「10倍シナリオ」、右 `.sig.bear`「弱気・除外シナリオ」
  - `.act`: 「四半期で見るKPI」と「次の確認アクション」を 1〜2 行で表示
  - Investment thesis、Key growth drivers、Competitive advantage、Financial quality、Valuation scenario、Major risks は長文にせず要点だけ残す
- スコア（Market opportunity / Revenue growth quality / Margin expansion potential / Competitive advantage / Balance sheet strength / Management quality / Valuation attractiveness / 10-bagger plausibility）は `.stock` 内の短い箇条書き、または銘柄ごとの `<details class="archive">` に入れる

### 「5. 量子コンピューティング」「6. 宇宙」（`.news-list` ＋ `.watch`）
- それぞれ `<section id="quantum">`（num `5`）／`<section id="space">`（num `6`）
- 構成: `.block-label`「📰 関連ニュース」＋ `.news-list` ＞ `.news` × N → `.block-label`「📈 関連銘柄」＋ `.watch` ＞ `.stock` × N
- `.news`: `.imp` 重要度 ＋ `.title` 見出し／`.nrows` に「関連銘柄・相場への影響・注目点」
- `.stock`: `.nm` に `Ticker — Company name`、`.role` に区分、`.ch-note` を流用して「分野 / 時価総額 / 区分」、`.signals` に強気材料・弱気材料、`.act` に注目KPI
- ニュースまたは銘柄の一方しか無い場合、存在する `.block-label` ＋ ブロックのみ描画する

## Step 3: index.html 更新（最新レポートで上書き）

`index.html` は **最新レポートそのもの** をトップページとして提供する。レポート一覧ページは作らない（毎日上書きされる単一ページ運用）。

### 手順
1. Step 2 で生成した `reports/<basename>.html` を **そのままの内容** で `index.html` にコピー（上書き）
2. ルート直下 (`index.html`) からの CSS パスのみ書き換える（`?v=日付` のキャッシュバスターは保持）:
   - レポート版: `<link rel="stylesheet" href="../assets/style.css?v=YYYY-MM-DD">`
   - index 版: `<link rel="stylesheet" href="assets/style.css?v=YYYY-MM-DD">`
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
