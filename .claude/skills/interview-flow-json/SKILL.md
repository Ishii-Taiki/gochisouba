---
name: interview-flow-json
description: tree 形式の問診フロー（インデント箇条書き／Mermaid／自然言語の分岐記述など）を受け取り、interview-designer (https://github.com/Ishii-Taiki/coco-tsubo の apps/interview-designer) の JSON Panel にそのまま貼り付け可能な `Flow[]` JSON を生成する。「問診フローを JSON に」「interview-designer 用 JSON 作って」「このツリーを designer に流したい」などのリクエスト時に使用。
---

# Interview Flow JSON Skill

ユーザーが渡す **tree 形式の問診フロー** を、interview-designer の JSON Panel が受け付ける `Flow[]` 形式に変換して出力する。生成 JSON はそのままコピペで designer のキャンバスに反映できる状態にすること。

正本スキーマは `/Users/taikiishii/coco-tsubo/apps/interview-designer/src/lib/schema.ts`。実際のバリデーションは `flows-to-graph.ts#parseFlows` が行う。スキルを使う前にこの 2 ファイルを **必ず確認** し、当ドキュメントとズレていれば現物を優先する。

## Output Schema（要点）

```ts
type Flow = {
  id: string;
  name: string;
  nodes: Record<string, FlowNode>;
  start: string | null;
};

type FlowNode = SingleChoiceQuestion | Result;

type SingleChoiceQuestion = {
  type: "question";
  input: "single-select";        // designer は現状 single-select のみ受理
  text: string;
  image?: string;                 // 空なら **キーごと省略**
  options: ChoiceOption[];
};

type ChoiceOption = {
  id: string;
  text: string;
  next: string | null;            // 遷移先の nodes キー or null（未接続）
  image?: string;                 // 空ならキーごと省略
};

type Result = {
  type: "result";
  name: string;                   // 結果の短い識別名（例: "受診"）
  text: string;                   // 利用者向けの説明文
};
```

ルート要素は **配列 (`Flow[]`)**。flow が 1 本でも `[ { ... } ]` で出す。

## Input Forms（受け付ける tree 形)

ユーザーは以下のいずれかで渡してくる。判別して読み取ること。

1. **インデント箇条書き**（推奨形式）
   ```
   発熱トリアージ
   - 発熱はありますか?
     - はい
       - 38度以上ですか?
         - はい → [受診] すぐに受診してください
         - いいえ → [自宅安静] 経過観察
     - いいえ → [自宅安静] 様子を見てください
   ```
   ルート行が flow 名、`- 質問` がクエスチョン、その下の `- 選択肢` がオプション、`→ [結果名] 結果文` で result リーフ、`→ <別質問の参照>` で既存ノード再利用。

2. **Mermaid `flowchart` / `graph TD`**
   ```
   flowchart TD
     Q1[発熱はありますか?] -->|はい| Q2[38度以上ですか?]
     Q1 -->|いいえ| R2[自宅安静: 様子を見てください]
     Q2 -->|はい| R1[受診: すぐに受診してください]
     Q2 -->|いいえ| R2
   ```
   矩形 `[...]` を質問／結果、エッジラベル `|...|` をオプション text として読む。`R*` のように先頭が R / Result / 「結果」を含むラベル → result ノード。

3. **自然言語の分岐記述**
   形式が緩い場合は、まず構造（質問の親子関係・選択肢・結果）を抽出して 1. のインデント形に内部変換してから JSON を組む。

迷ったら **AskUserQuestion で flow 名や result の name を確認** する。勝手に省略しない。

## ID 命名規則

- `Flow.id`: 短い slug（例: `fever-triage`）。日本語のみが渡された場合は英訳または kebab-case 化を提案する。
- 質問ノードキー: `q1`, `q2`, ... の連番（出現順）
- 結果ノードキー: `r1`, `r2`, ...（result ごとの一意識別子）
- オプション ID: `<question_id>_<index>`（例: `q1_1`, `q1_2`）。`option.id` と `nodes` のキーは同じ flow 内で衝突しないこと。
- 同じ result に複数の選択肢から流入する場合は、result ノードを **1 つだけ作って共有**（重複生成しない）。`option.next` で同じ result key を指す。

## 変換手順

1. **入力形式を判別**（箇条書き / Mermaid / 自然言語）
2. flow 単位の境界を決める。複数 flow なら配列に分割
3. 各 flow について:
   - flow 名と flow id を確定
   - 全ノードを列挙（質問・結果）し、上記命名規則で id を採番
   - `nodes` レコードを構築。質問は `options[]` に各選択肢の `{ id, text, next }` を埋める
   - `start` は最初の質問の id（質問が 1 つも無ければ null）
4. **画像情報** は入力に明示があるときだけ `image` キーを付ける。空文字や `undefined` を入れない
5. **未接続オプション**（→ で遷移先が示されていない）は `next: null` にする
6. JSON を整形（2 スペースインデント）して出力

## バリデーションチェックリスト

出力前に以下を全てパス：

- [ ] ルートが配列
- [ ] 各 flow に `id` / `name` / `nodes` / `start` がある
- [ ] `start` は `null` か `nodes` のキーに存在する文字列
- [ ] 質問は `type: "question"`, `input: "single-select"`, `text`, `options` を持つ
- [ ] 質問に `slider` などは出さない（designer が拒否する）
- [ ] 各 `option.id` は同 flow の他 option と重複しない
- [ ] 各 `option.next` は `null` か `nodes` のキーに存在
- [ ] 結果は `type: "result"`, `name`, `text` を持つ
- [ ] `image` キーは値が非空のときのみ存在（空文字は出さない）
- [ ] 到達不能ノードを作らない（start から辿れない node を入れない）

## 出力フォーマット

チャットには以下の順で出す：

1. 1 行サマリ（flow 数 / 質問数 / 結果数）
2. ` ```json ` フェンス内で整形済み JSON（コピー用）
3. 補足（曖昧だった箇所の解釈、確認したいポイント）があれば末尾に

ユーザーがファイル保存を希望する場合のみファイルに書く（既定では出力しない）。保存する場合は相対パスを確認し、ユーザー指定がなければ `interview-flows/<flow-id>.json` を提案する。

## Example

**入力（インデント形）**

```
発熱トリアージ
- 発熱はありますか?
  - はい
    - 38度以上ですか?
      - はい → [受診] すぐに医療機関を受診してください
      - いいえ → [自宅安静] 様子を見てください
  - いいえ → [自宅安静]
```

**出力**

```json
[
  {
    "id": "fever-triage",
    "name": "発熱トリアージ",
    "start": "q1",
    "nodes": {
      "q1": {
        "type": "question",
        "input": "single-select",
        "text": "発熱はありますか?",
        "options": [
          { "id": "q1_1", "text": "はい", "next": "q2" },
          { "id": "q1_2", "text": "いいえ", "next": "r2" }
        ]
      },
      "q2": {
        "type": "question",
        "input": "single-select",
        "text": "38度以上ですか?",
        "options": [
          { "id": "q2_1", "text": "はい", "next": "r1" },
          { "id": "q2_2", "text": "いいえ", "next": "r2" }
        ]
      },
      "r1": {
        "type": "result",
        "name": "受診",
        "text": "すぐに医療機関を受診してください"
      },
      "r2": {
        "type": "result",
        "name": "自宅安静",
        "text": "様子を見てください"
      }
    }
  }
]
```

## Important Rules（絶対遵守）

- スキーマの正本は `coco-tsubo/apps/interview-designer/src/lib/schema.ts`。当 SKILL.md と食い違ったら **現物優先**（変更されている可能性あり）
- `slider` 形式は出さない（designer の `parseFlows` が拒否する）
- `image` を空文字で出さない（designer 側で `image: ""` は省略済み扱いと不整合になる）
- 同一 result に流入する複数オプションは result ノードを共有させ、重複生成しない
- `option.next` の参照先は必ず `nodes` のキーか `null`。存在しない id を書かない
- 入力に flow 名がなければユーザーに確認する。勝手に英語化して `id` を決めない
- JSON 内に // コメントを書かない（標準 JSON）
- 出力 JSON はそのまま designer の JSON Panel に貼って通る形にする。検証は `flows-to-graph.ts#parseFlows` のロジックを参照
