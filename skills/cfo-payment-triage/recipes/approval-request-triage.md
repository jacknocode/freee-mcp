# 各種申請トリアージ レシピ

`approval_requests` の詳細分析。フォーム定義依存のため、
`request_items[]` の解析が中心。金額フィールドは任意で存在しない場合もある。

## payment_requests / expense_applications との主な差異

| 項目 | approval_requests | その他2種 |
|------|------------------|---------|
| 金額 | `request_items[]` の amount タイプ。任意 | 専用フィールドあり |
| 取引先 | `request_items[]` の partner タイプ。任意 | payment: partner_id あり |
| 明細 | `request_items[]`（フォーム定義依存） | 構造化された行 |
| 支払期限 | `request_items[]` の date タイプ。任意 | payment: payment_date あり |
| フォーム種別 | `form_id` で分類 | なし |

## Step 1: 詳細取得

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/approval_requests/<id>"
}
```

主要フィールド:

| フィールド | 意味 | 用途 |
|-----------|------|------|
| `title` | 申請タイトル | フォーム種別の手がかり |
| `form_id` | 申請フォームID | フォーム種別分類 |
| `request_items[]` | 申請項目 | 金額・取引先・理由の抽出源 |
| `comments[]` | コメント | 往復数・URL抽出 |
| `approval_flow_logs[]` | 承認履歴 | 差し戻し理由 |
| `current_step_id` | 現在ステップID | 承認アクション用 |
| `current_round` | ラウンド | 再申請検出 |
| `applicant_id` | 申請者ID | 申請者パターン取得 |
| `application_date` | 申請日 | 待機日数計算 |

## Step 2: request_items の解析

`request_items[]` は申請フォームで定義された項目の配列。
各エントリは `id`・`type`・`value` を持つ。

### タイプ別の解釈

| type | 意味 | トリアージ用途 |
|------|------|--------------|
| `title` | 申請タイトル | フォームタイトル（通常は不要） |
| `single_line` | 1行テキスト | 申請理由・背景（シグナル語検出元） |
| `multi_line` | 複数行テキスト | 詳細説明・URL抽出元 |
| `select` | プルダウン選択 | 費用種別・部門・優先度等 |
| `date` | 日付 | 発生日・期限（urgency 判断用） |
| `amount` | 金額 | リスクスコアの金額算出元 |
| `receipt` | 証憑ファイルID | 根拠資料の有無 |
| `section` | 部門ID | 承認経路の確認 |
| `partner` | 取引先ID | 関連取引先の特定 |

### 金額の算出

```
total_amount = sum(
  int(item.value)
  for item in request_items
  if item.type == "amount" and item.value is not null
)
```

`type: "amount"` のエントリがゼロ件の場合は `total_amount: null` とし、金額ベースのゲートを適用しない。

### 日付フィールドの取得

`type: "date"` のエントリを期限として扱い、urgency スコアの `due_date` に使用する。
複数の date フィールドがある場合は最も早い日付を優先する。

### テキストフィールドのシグナル検出

`type: "single_line"` / `multi_line"` の `value` から URL とシグナル語を抽出する。
`payment_requests` の `description` + `comments` と同じロジックを適用する（`payment-request-triage.md` Step 3 参照）。

### receipt フィールドの確認

`type: "receipt"` エントリが存在するのに `value` が空・null の場合:

→ AR-G001: 根拠資料未添付（フォームが添付を要求しているが未添付）

## Step 3: フォーム種別の判断

`form_id` と `title` からフォームの種別を推定し、評価の重みを調整する。

### フォーム種別のヒューリスティック

| タイトルパターン | 推定フォーム種別 | 主要確認ポイント |
|----------------|---------------|--------------|
| 稟議・予算申請 | 予算・投資稟議 | 金額根拠・上位承認経緯 |
| 出張・旅費 | 出張申請 | 日程・目的・概算額の整合 |
| 採用・人事 | 人事関連 | 人数・待遇条件 |
| 契約・締結 | 契約承認 | 契約条件・法務確認の有無 |
| 購入・調達 | 購買申請 | 見積書添付・相見積有無 |
| その他・不明 | 汎用 | テキスト内容の具体性 |

## Step 4: 過去類似案件の取得

同一フォームの過去案件を取得してベースラインを確認する。

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/approval_requests",
  "query": {
    "form_id": <form_id>,
    "start_application_date": "<今日から365日前>",
    "limit": 50
  }
}
```

確認する指標:

- 同一フォームで承認された案件の `request_items[].value`（amount）の分布
- 過去の差し戻し案件の `approval_flow_logs[].body`（差し戻し理由パターン）
- 申請者の過去提出頻度

`total_amount: null` の場合は金額比較をスキップし、定性的なパターン確認のみ行う。

## Step 5: Gate / Heuristic 評価

### ゲートルール

#### AR-G001: フォーム必須の証憑が未添付

条件: `request_items[]` に `type: "receipt"` エントリがあり、かつ `value` が空
結果: blocking — 根拠資料の添付を求める

#### AR-G002: 金額根拠のテキスト説明が極めて薄い

条件:
- `total_amount` が存在（>= 300,000円） かつ
- `single_line` / `multi_line` フィールドのテキスト合計が50文字未満 かつ
- `type: "receipt"` フィールドがない

結果: blocking — 金額根拠の説明を求める

### ポリシールール

#### AR-P001: 高額稟議・予算申請

条件: `total_amount >= 1,000,000`
結果: high — CFO確認対象

#### AR-P002: 契約系フォームの法務確認

条件: タイトルまたはテキストに contract_signal かつ `total_amount >= 500,000`
結果: high — 契約条件の法務確認経緯を確認

#### AR-P003: 再申請

条件: `current_round > 0`
結果: medium — 前回差し戻し理由（`approval_flow_logs[]`）の解消を確認

#### AR-P004: 金額フィールドなし・高リスクタイトル

条件: `total_amount: null` かつ タイトルに exception_signal / contract_signal
結果: medium — 金額規模の確認と影響範囲の確認

### ヒューリスティック

#### AR-H001: select フィールドが空

条件: `type: "select"` エントリの `value` が空
結果: medium — 必須選択肢の入力漏れ確認

#### AR-H002: 説明フィールドが極端に短い

条件: `multi_line` フィールドの `value` が 30文字未満
結果: medium — 詳細説明の追記を求める

#### AR-H003: コメント往復多数

payment-request-triage.md の H001 と同じ基準（往復5回以上）
結果: medium — 未解決論点の確認

## Step 6: 推奨アクション

```
if blocking gate findings > 0:
  recommended_action = "return_candidate"

elif total_amount >= 5,000,000 AND (exception_signal OR contract_signal):
  recommended_action = "escalate"

elif gate findings = 0 AND high findings = 0 AND unresolved <= 1:
  recommended_action = "approve_candidate"

else:
  recommended_action = "approve_after_check"
```

`total_amount: null` の場合は金額条件をスキップし、定性リスクのみで判断する。

## Step 7: ブリーフ出力

```
## [優先度] 各種申請 No.{application_number}: {title}

### なぜ今見るべきか
{待機日数・推定フォーム種別・再申請の有無}

### この申請は何か
申請者ID: {applicant_id}
申請フォーム: form_id={form_id}（推定種別: {推定フォーム種別}）
申請日: {application_date}
金額: ¥{total_amount:,}（{total_amount: null なら「金額フィールドなし」}）

### 申請内容（request_items 要約）
{type別に整理した申請内容の要約}
例:
- 申請理由（single_line）: 「Q1予算が不足したため追加申請」
- 金額（amount）: ¥2,000,000
- 期限（date）: 2026-03-31
- 証憑（receipt）: 添付あり

### 過去同一フォームの傾向
承認件数（12ヶ月）: {N} 件
平均金額: ¥{avg:,}（金額フィールドある場合）
よくある差し戻し理由: {パターンがあれば記載}

### ゲート評価
{blocking があれば}

### 未解決論点（最大3件）
1. ...

### 推奨アクション
{approve_candidate / approve_after_check / return_candidate / escalate}

---
最終操作は必ず人間が行ってください。
```

## 承認アクション

```
freee_api_post {
  "service": "accounting",
  "path": "/api/1/approval_requests/<id>/actions",
  "body": {
    "company_id": <company_id>,
    "approval_action": "approve",
    "target_step_id": <current_step_id>,
    "target_round": <current_round>
  }
}
```

`approval_action` の選択肢: `approve` / `force_approve` / `reject` / `feedback` / `cancel`
